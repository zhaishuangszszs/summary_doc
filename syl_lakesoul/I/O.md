## write部分

### builder写
```
fn init_table(batch: RecordBatch, table_name: &str, pks:Vec<String>) -> LakeSoulIOConfigBuilder {

        let builder = LakeSoulIOConfigBuilder::new()
                .with_schema(batch.schema())
                .with_primary_keys(pks);
        execute_upsert(batch, table_name, builder.clone())
    }
```
`init_table`看起来创建一个表，指定`batch`数据，表名，主键。
***
* 具体内部构造如下

一个结构体包裹`LakeSoulIOConfig`
```
pub struct LakeSoulIOConfigBuilder {
    config: LakeSoulIOConfig,
}
```
`结构体包含很多配置信息`
```
#[derivative(Default, Clone)]
pub struct LakeSoulIOConfig {
    // files to read or write
    pub(crate) files: Vec<String>,
    // primary key column names
    pub(crate) primary_keys: Vec<String>,
    ......
}
```
`LakeSoulIOConfigBuilder`的new函数，调用`LakeSoulIOConfig`派生出的`default`函数
```
impl LakeSoulIOConfigBuilder{
    pub fn new() -> Self {
        LakeSoulIOConfigBuilder {
            config: LakeSoulIOConfig::default(),
        }
    }
}
```
* 综上：`builder`只是配置了下信息并没有产生具体文件
* 下一个步
`execute_upsert(batch, table_name, builder.clone())`

```
fn execute_upsert(batch: RecordBatch, table_name: &str, builder: LakeSoulIOConfigBuilder) -> LakeSoulIOConfigBuilder {
        let file = [env::temp_dir().to_str().unwrap(), table_name, format!("{}.parquet", SystemTime::now().duration_since(SystemTime::UNIX_EPOCH).unwrap().as_millis().to_string()).as_str()].iter().collect::<PathBuf>().to_str().unwrap().to_string();
        let builder = builder.with_file(file.clone()).with_schema(batch.schema());
        let config = builder.clone().build();

        let writer = SyncSendableMutableLakeSoulWriter::try_new(config, Builder::new_current_thread().build().unwrap()).unwrap();
        let _ = writer.write_batch(batch);
        let _ = writer.flush_and_close();
        builder
    }
```
file的结果看一个例子
```
use std::env;
use std::path::PathBuf;
use std::time::SystemTime;
fn main() {
    let table_name="table_name_test";
    let file = [env::temp_dir().to_str().unwrap(), table_name, format!("{}.parquet", SystemTime::now().duration_since(SystemTime::UNIX_EPOCH).unwrap().as_millis().to_string()).as_str()].iter().collect::<PathBuf>().to_str().unwrap().to_string();
    println!("{}",file);
}
```
* `file="/tmp/table_name_test\1701678179506.parquet"`时间戳命名的文件，文件夹是表名。

* `builder`配置文件名和schema

`SyncSendableMutableLakeSoulWriter`包含三个部分
```
pub struct SyncSendableMutableLakeSoulWriter {
    inner: Arc<Mutex<SendableWriter>>,
    runtime: Arc<Runtime>,
    schema: SchemaRef,
}
```
`SendableWriter`如下是一个别名
```
type SendableWriter = Box<dyn AsyncBatchWriter + Send>;
```
`AsyncBatchWriter`是一个trait
```
pub trait AsyncBatchWriter {
    async fn write_record_batch(&mut self, batch: RecordBatch) -> Result<()>;

    async fn flush_and_close(self: Box<Self>) -> Result<()>;

    async fn abort_and_close(self: Box<Self>) -> Result<()>;
}
```
`try_new`需要两个参数config和runtime其中`Builder::new_current_thread().build().unwrap()`用到tokio库构建一个runtime

```
impl SyncSendableMutableLakeSoulWriter {
    pub fn try_new(config: LakeSoulIOConfig, runtime: Runtime) -> Result<Self> {
        let runtime = Arc::new(runtime);
        runtime.clone().block_on(async move {
            // if aux sort cols exist, we need to adjust the schema of final writer
            // to exclude all aux sort cols
            let writer_schema: SchemaRef = if !config.aux_sort_cols.is_empty() {
                let schema = config.schema.0.clone();
                let proj_indices = schema
                    .fields
                    .iter()
                    .filter(|f| !config.aux_sort_cols.contains(f.name()))
                    .map(|f| schema.index_of(f.name().as_str()).map_err(DataFusionError::ArrowError))
                    .collect::<Result<Vec<usize>>>()?;
                Arc::new(schema.project(proj_indices.borrow())?)
            } else {
                config.schema.0.clone()
            };

            let mut writer_config = config.clone();
            writer_config.schema = IOSchema(uniform_schema(writer_schema));
            let writer = MultiPartAsyncWriter::try_new(writer_config).await?;

            let schema = writer.schema.clone();
            let writer: Box<dyn AsyncBatchWriter + Send> = if !config.primary_keys.is_empty() {
                Box::new(SortAsyncWriter::try_new(writer, config, runtime.clone())?)
            } else {
                Box::new(writer)
            };

            Ok(SyncSendableMutableLakeSoulWriter {
                inner: Arc::new(Mutex::new(writer)),
                runtime,
                schema, // this should be the final written schema
            })
        })
    } 

}
```
* `try_new`中`config.schema.0.clone()`解释
在`LakeSoulIOConfig`结构体中有
```
    // arrow schema
    pub(crate) schema: IOSchema,
```
而`struct IOSchema`是元组结构体
```
pub struct IOSchema(pub(crate) SchemaRef);
```
所以就是schame的引用

* 没有辅助排序列，所以`writer_schema`就是配置的schame
* ` writer_config`是配置的config的克隆
之后修改shcema
`writer_config.schema = IOSchema(uniform_schema(writer_schema));`中`uniform_schema`将field为timestamp类型的时区统一为**UTC**时区

**接着**
`let writer = MultiPartAsyncWriter::try_new(writer_config).await?;`
`MultiPartAsyncWriter`是一个结构体
```
pub struct MultiPartAsyncWriter {
    in_mem_buf: InMemBuf,
    sess_ctx: SessionContext,
    schema: SchemaRef,
    writer: Box<dyn AsyncWrite + Unpin + Send>,
    multi_part_id: MultipartId,
    arrow_writer: ArrowWriter<InMemBuf>,
    _config: LakeSoulIOConfig,
    object_store: Arc<dyn ObjectStore>,
    path: Path,
}
```
* `try_new`实现
```
impl MultiPartAsyncWriter {
    pub async fn try_new(mut config: LakeSoulIOConfig) -> Result<Self> {
        if config.files.is_empty() {
            return Err(Internal("wrong number of file names provided for writer".to_string()));
        }
        let sess_ctx = create_session_context(&mut config)?;
        let file_name = &config.files.last().unwrap();

        // local style path should have already been handled in create_session_context,
        // so we don't have to deal with ParseError::RelativeUrlWithoutBase here
        let (object_store, path) = match Url::parse(file_name.as_str()) {
            Ok(url) => Ok((
                sess_ctx
                    .runtime_env()
                    .object_store(ObjectStoreUrl::parse(&url[..url::Position::BeforePath])?)?,
                Path::from(url.path()),
            )),
            Err(e) => Err(DataFusionError::External(Box::new(e))),
        }?;

        let (multipart_id, async_writer) = object_store.put_multipart(&path).await?;
        let in_mem_buf = InMemBuf(Arc::new(AtomicRefCell::new(VecDeque::<u8>::with_capacity(
            16 * 1024 * 1024,
        ))));
        let schema: SchemaRef = config.schema.0.clone();

        let arrow_writer = ArrowWriter::try_new(
            in_mem_buf.clone(),
            uniform_schema(schema.clone()),
            Some(
                WriterProperties::builder()
                    .set_max_row_group_size(config.max_row_group_size)
                    .set_write_batch_size(config.batch_size)
                    .set_compression(Compression::SNAPPY)
                    .build(),
            ),
        )?;

        Ok(MultiPartAsyncWriter {
            in_mem_buf,
            sess_ctx,
            schema: uniform_schema(schema),
            writer: async_writer,
            multi_part_id: multipart_id,
            arrow_writer,
            _config: config,
            object_store,
            path,
        })
    }
}
```
* `create_session_context`函数传入参数`LakeSoulIOConfig`配置**datafusion的SessionContext**并返回SessionContext
```
pub fn create_session_context(config: &mut LakeSoulIOConfig) -> Result<SessionContext> {
    let mut sess_conf = SessionConfig::default()
        .with_batch_size(config.batch_size)
        .with_parquet_pruning(true)
        .with_prefetch(config.prefetch_size);

    sess_conf.options_mut().optimizer.enable_round_robin_repartition = false; // if true, the record_batches poll from stream become unordered
    sess_conf.options_mut().optimizer.prefer_hash_join = false; //if true, panicked at 'range end out of bounds'
    sess_conf.options_mut().execution.parquet.pushdown_filters = true;
    // sess_conf.options_mut().execution.parquet.enable_page_index = true;

    // limit memory for sort writer
    let runtime = RuntimeEnv::new(RuntimeConfig::new().with_memory_limit(128 * 1024 * 1024, 1.0))?;

    // firstly parse default fs if exist
    let default_fs = config
        .object_store_options
        .get("fs.defaultFS")
        .or_else(|| config.object_store_options.get("fs.default.name"))
        .cloned();
    if let Some(fs) = default_fs {
        config.default_fs = fs.clone();
        register_object_store(&fs, config, &runtime)?;
    };

    // register object store(s) for input/output files' path
    // and replace file names with default fs concatenated if exist
    let files = config.files.clone();
    let normalized_filenames = files
        .into_iter()
        .map(|file_name| register_object_store(&file_name, config, &runtime))
        .collect::<Result<Vec<String>>>()?;
    config.files = normalized_filenames;

    // create session context
    Ok(SessionContext::with_config_rt(sess_conf, Arc::new(runtime)))
}
```

> fs.defaultFS 是 Hadoop 配置中一个重要的属性，用于指定 Hadoop 分布式文件系统（HDFS）的默认文件系统 URI。这个配置项告诉 Hadoop 客户端和其他服务，它们应该连接到哪个 HDFS 集群。fs.default.name 是过时的配置项

```
 <property>
     <name>fs.defaultFS</name>
     <value>hdfs://node01:8020</value>
 </property>
```

```
let default_fs = config
        .object_store_options
        .get("fs.defaultFS")
        .or_else(|| config.object_store_options.get("fs.default.name"))
        .cloned();
```
这里，`object_store_options`,是结构体`LakeSoulIOConfig`中成员
```
    // object store related configs
    pub(crate) object_store_options: HashMap<String, String>,
```
`default_fs`就是url字符串用optiona包裹，修改`LakeSoulIOConfig`中`default_fs`为配置的hdfs的**url**
之后
`register_object_store(&fs, config, &runtime)?;`
注册对象存储，并根据配置的url**返回存储路径**
```
// register object store(s) for input/output files' path
    // and replace file names with default fs concatenated if exist
    let files = config.files.clone();
    let normalized_filenames = files
        .into_iter()
        .map(|file_name| register_object_store(&file_name, config, &runtime))
        .collect::<Result<Vec<String>>>()?;
    config.files = normalized_filenames;
```
**根据文件名字类型中的schema[指hdfs://...]来在文件路径前加上配置的default_fs。**（猜测本地文件匹配file://）

* 之后

`let file_name = &config.files.last().unwrap();`filename取最新的文件
之后注册出一个`MultiPartAsyncWriter`

* 接着
```
let writer: Box<dyn AsyncBatchWriter + Send> = if !config.primary_keys.is_empty() {
                Box::new(SortAsyncWriter::try_new(writer, config, runtime.clone())?)
            } else {
                Box::new(writer)
            };

```
主键非空返回`SortAsyncWriter`,否则返回`MultiPartAsyncWriter`

**回到**`execute_upsert`
`let _ = writer.write_batch(batch);`调用`SyncSendableMutableLakeSoulWriter`的`write_batch`
```
pub fn write_batch(&self, record_batch: RecordBatch) -> Result<()> {
        let inner_writer = self.inner.clone();
        let runtime = self.runtime.clone();
        runtime.block_on(async move {
            let mut writer = inner_writer.lock().await;
            writer.write_record_batch(record_batch).await
        })
    }
```
`let inner_writer = self.inner.clone();`中`inner`实际为`AsyncBatchWriter`
而`AsyncBatchWriter`是一个trait
```
#[async_trait]
pub trait AsyncBatchWriter {
    async fn write_record_batch(&mut self, batch: RecordBatch) -> Result<()>;

    async fn flush_and_close(self: Box<Self>) -> Result<()>;

    async fn abort_and_close(self: Box<Self>) -> Result<()>;
}
```
之后` writer.write_record_batch(record_batch).await`调用AsyncBatchWrite的`write_record_batch`。


### client写

`let client = Arc::new(MetaDataClient::from_env().await?);`
`MetaDataClient`是一个结构体
`pub struct MetaDataClient {
    client: Arc<Mutex<Client>>,
    prepared: Arc<Mutex<PreparedStatementMap>>,
    max_retry: usize,
}`
`from_env()`函数
```
impl MetaDataClient {
    pub async fn from_env() -> Result<Self> {
        match env::var("lakesoul_home") {
            Ok(config_path) => {
                let config = fs::read_to_string(&config_path).unwrap_or_else(|_| panic!("Fails at reading config file {}", &config_path));
                let config_map = config.split('\n').filter_map(|property| {
                    property.find('=').map(|idx| property.split_at(idx + 1))
                }).collect::<HashMap<_, _>>();
                let url = Url::parse(&config_map.get("lakesoul.pg.url=").unwrap_or(&"jdbc:postgresql://127.0.0.1:5432/lakesoul_test?stringtype=unspecified")[5..]).unwrap();
                Self::from_config(
                    format!(
                        "host={} port={} dbname={} user={} password={}", 
                        url.host_str().unwrap(), 
                        url.port().unwrap(), 
                        url.path_segments().unwrap().next().unwrap(),
                        config_map.get("lakesoul.pg.username=").unwrap_or(&"lakesoul_test"), 
                        config_map.get("lakesoul.pg.password=").unwrap_or(&"lakesoul_test"))
                ).await
            }
            Err(_) => Self::from_config(
                    "host=127.0.0.1 port=5432 dbname=lakesoul_test user=lakesoul_test password=lakesoul_test".to_string()
                ).await
        }
        
    }
}
```
读取环境变量`"lakesoul_home"`路径，存在读取该路径下文件中内容到`config`，之后根据文件内容中"="和换行将内容解析为`HashMap`。`url`则获取解析后`lakesoul.pg.url=`对应的url值。
* 之后调用`from_config`
* `from_config`调用`from_config_and_max_retry`

```
 pub async fn from_config_and_max_retry(config: String, max_retry: usize) -> Result<Self> {
        let client = Arc::new(Mutex::new(create_connection(config).await?));
        let prepared = Arc::new(Mutex::new(PreparedStatementMap::new()));
        Ok(Self {
            client,
            prepared,
            max_retry
        })
    }
```

`reate_connection`利用`tokio_postgres`和PG建立连接
```
pub async fn create_connection(
    config: String
) -> Result<Client> {    
    let (client, connection) = 
        match tokio_postgres::connect(config.as_str(), NoTls).await {
            Ok((client, connection))=>(client, connection),
            Err(e)=>{
                eprintln!("{}", e);
                return Err(LakeSoulMetaDataError::from(ErrorKind::ConnectionRefused))
            }
    };

    spawn(async move {
        if let Err(e) = connection.await {
            eprintln!("connection error: {}", e);
        }
    });

    Ok( client )
}
```
`PreparedStatementMap`是HashMap类型的别名
```
pub type PreparedStatementMap = HashMap<DaoType, Statement>;
```
`DaoType`是枚举类型,`pub const DAO_TYPE_QUERY_ONE_OFFSET : i32 = 0;`这些`TYPE_QUERY_ONE_OFFSET`都是常量

```
pub enum DaoType{
    SelectNamespaceByNamespace = DAO_TYPE_QUERY_ONE_OFFSET,
    SelectTablePathIdByTablePath = DAO_TYPE_QUERY_ONE_OFFSET + 1,
    SelectTableInfoByTableId = DAO_TYPE_QUERY_ONE_OFFSET + 2,
    SelectTableNameIdByTableName = DAO_TYPE_QUERY_ONE_OFFSET + 3,
    ......
}
```
`Statement`是`tokio_postgres`中的**猜测是查询状态**

* 之后`from_env`返回一个client

查看
`init_table`，`builder`不需要看了是一个`LakeSoulIOConfigBuilder`
```
async fn init_table(batch: RecordBatch, table_name: &str, schema: SchemaRef,  pks:Vec<String>, client: MetaDataClientRef) -> Result<()> {
        let builder = LakeSoulIOConfigBuilder::new()
                .with_schema(schema)
                .with_primary_keys(pks);
        create_table(client.clone(), table_name, builder.build()).await?;
        execute_upsert(batch, table_name, client).await
    }
```
查看`create_table(client.clone(), table_name, builder.build()).await?;`
```
pub(crate) async fn create_table(client: MetaDataClientRef, table_name: &str, config: LakeSoulIOConfig) -> Result<()> {
    client.create_table(
        TableInfo {
            table_id: format!("table_{}", uuid::Uuid::new_v4().to_string()),
            table_name: table_name.to_string(), 
            table_path: [env::temp_dir().to_str().unwrap(), table_name].iter().collect::<PathBuf>().to_str().unwrap().to_string(),
            table_schema: serde_json::to_string::<ArrowJavaSchema>(&config.schema().into()).unwrap(),
            table_namespace: "default".to_string(),
            properties: "{}".to_string(),
            partitions: ";".to_owned() + config.primary_keys_slice().iter().map(String::as_str).collect::<Vec<_>>().join(",").as_str(),
            domain: "public".to_string(),
    }).await?;
    Ok(()) 
}
```
其中调用`client.create_table`传入`TableInfo `，其中`TableInfo `配置了问价路径uuid等信息


**暂时返回init_table**

* 接着调用
`execute_upsert(batch, table_name, client).await`
```
async fn execute_upsert(record_batch: RecordBatch, table_name: &str, client: MetaDataClientRef) -> Result<()> {
        let builder = create_io_config_builder(client, None).await?;
        let sess_ctx = create_session_context(&mut builder.clone().build())?;
        
        let provider = LakeSoulSinkProvider::new(table_name).await?;
        sess_ctx.register_table(table_name, Arc::new(provider)).unwrap();

        let num_rows = record_batch.num_rows();
        let schema = record_batch.schema();

        let logical_plan = LogicalPlanBuilder::insert_into(
            sess_ctx.read_batch(record_batch)?.into_unoptimized_plan(), 
            table_name.to_string(), 
            &schema.deref())?
            .build()?;
        let dataframe = DataFrame::new(sess_ctx.state(), logical_plan);
        
        let results = dataframe
            // .explain(true, false)?
            .collect()
            .await?;

        // print_batches(&results);

        assert_batches_eq!(&[
            "+-----------+",
            "| row_count |",
            "+-----------+",
            format!("| {:<10}|", num_rows).as_str(),
            "+-----------+",
            ], &results);
        Ok(())
    }
```
`create_io_config_builder(client, None).await?;`根据`client`构造`LakeSoulIOConfigBuilder`。
因为`init_table`中`create_table`用到了`table_name`,并且传给了`client.create_table`所以这里传`None`

查看下`create_session_context`
```
pub fn create_session_context(config: &mut LakeSoulIOConfig) -> Result<SessionContext> {
    let mut sess_conf = SessionConfig::default()
        .with_batch_size(config.batch_size)
        .with_parquet_pruning(true)
        .with_prefetch(config.prefetch_size);

    sess_conf.options_mut().optimizer.enable_round_robin_repartition = false; // if true, the record_batches poll from stream become unordered
    sess_conf.options_mut().optimizer.prefer_hash_join = false; //if true, panicked at 'range end out of bounds'
    sess_conf.options_mut().execution.parquet.pushdown_filters = true;
    // sess_conf.options_mut().execution.parquet.enable_page_index = true;

    // limit memory for sort writer
    let runtime = RuntimeEnv::new(RuntimeConfig::new().with_memory_limit(128 * 1024 * 1024, 1.0))?;

    // firstly parse default fs if exist
    let default_fs = config
        .object_store_options
        .get("fs.defaultFS")
        .or_else(|| config.object_store_options.get("fs.default.name"))
        .cloned();
    if let Some(fs) = default_fs {
        config.default_fs = fs.clone();
        register_object_store(&fs, config, &runtime)?;
    };

    // register object store(s) for input/output files' path
    // and replace file names with default fs concatenated if exist
    let files = config.files.clone();
    let normalized_filenames = files
        .into_iter()
        .map(|file_name| register_object_store(&file_name, config, &runtime))
        .collect::<Result<Vec<String>>>()?;
    config.files = normalized_filenames;

    // create session context
    Ok(SessionContext::with_config_rt(sess_conf, Arc::new(runtime)))
}
```
`create_session_context`会修改builder的文件路径进行标准化。
返回`datafusion::execution::context`

* 回到execute_upsert

**datafusion实现写入表有序**
```
let logical_plan = LogicalPlanBuilder::insert_into(
            sess_ctx.read_batch(record_batch)?.into_unoptimized_plan(), 
            table_name.to_string(), 
            &schema.deref())?
            .build()?;
        let dataframe = DataFrame::new(sess_ctx.state(), logical_plan);
        
        let results = dataframe
            // .explain(true, false)?
            .collect()
            .await?;
```
* 猜测
1. 在insert_into 的第一个参数加排序
2. 在logical_plan处加一个sort

`LogicalPlanBuilder`中的sort
```
pub fn sort(
    self,
    exprs: impl IntoIterator<Item = impl Into<Expr>> + Clone
) -> Result<LogicalPlanBuilder, DataFusionError>
```
最后经过尝试在1处实现sort
之后
```
 let primary_keys=builder.primary_keys_slice();
        println!("{:?}",primary_keys);
```
打印为空。
最早传入pks的在`init_table`
之后在`create_table`中传给`partitions`,传给`client`
之后的`create_io_config_builder`里面经测试也没问题。
最后发现`execute_upsert`里的`create_io_config_builder`参数`table_name`不能为`None`
```
async fn execute_upsert(record_batch: RecordBatch, table_name: &str, client: MetaDataClientRef) -> Result<()> {
        println!("before create_io_config");
        let builder = create_io_config_builder(client, Some(table_name)).await?;
        println!("after create_io_config");
        let a=builder.primary_keys_slice();
        println!("2");
        println!("{:?}",a);

        let sess_ctx = create_session_context(&mut builder.clone().build())?;
        
        println!("before second in");
        let provider = LakeSoulSinkProvider::new(table_name).await?;
        println!("after second in");
        sess_ctx.register_table(table_name, Arc::new(provider)).unwrap();

        let num_rows = record_batch.num_rows();
        let schema = record_batch.schema();
        
        //my function
        let primary_keys=builder.primary_keys_slice();
        println!("hello");
        println!("{:?}",primary_keys);
        let sort_expr:Vec<Expr>=primary_keys.into_iter().map(|pk| col(pk).sort(true, false)).collect();
        //-----------

        // println!("{:?}",schema);
        let logical_plan = LogicalPlanBuilder::insert_into(
            sess_ctx.read_batch(record_batch)?.sort(sort_expr)?.into_unoptimized_plan(), 
            table_name.to_string(), 
            &schema.deref())?
            .build()?;
        // println!("{:?}",logical_plan.schema());
        
        
        // println!("ok");
        let dataframe = DataFrame::new(sess_ctx.state(), logical_plan);
        // println!("{:?}",dataframe.schema());
        
        // println!("ok2");
        let results = dataframe
            // .explain(true, false)?
            .collect()
            .await?;
        // print_batches(&results);

        assert_batches_eq!(&[
            "+-----------+",
            "| row_count |",
            "+-----------+",
            format!("| {:<10}|", num_rows).as_str(),
            "+-----------+",
            ], &results);
        Ok(())
    }
```


## writer部分

### MultiPartAsyncWriter

#### MultiPartAsyncWriter定义
```
pub struct MultiPartAsyncWriter {
    in_mem_buf: InMemBuf,
    sess_ctx: SessionContext,
    schema: SchemaRef,
    writer: Box<dyn AsyncWrite + Unpin + Send>,
    multi_part_id: MultipartId,
    arrow_writer: ArrowWriter<InMemBuf>,
    _config: LakeSoulIOConfig,
    object_store: Arc<dyn ObjectStore>,
    path: Path,
}
```
#### MultiPartAsyncWriter::try_new()


### SortAsyncWriter
#### SortAsyncWriter定义
```
pub struct SortAsyncWriter {
    sorter_sender: Sender<Result<RecordBatch>>,
    _sort_exec: Arc<dyn ExecutionPlan>,
    join_handle: JoinHandle<Result<()>>,
}
```
#### SortAsyncWriter::try_new()
```
 pub fn try_new(
        async_writer: MultiPartAsyncWriter,
        config: LakeSoulIOConfig,
        runtime: Arc<Runtime>,
    ) -> Result<Self> {
        let _ = runtime.enter();
        let schema = config.schema.0.clone();
        let receiver_stream_builder = RecordBatchReceiverStream::builder(schema.clone(), 8);
        let tx = receiver_stream_builder.tx();
        let recv_exec = ReceiverStreamExec::new(receiver_stream_builder, schema);

        let sort_exprs: Vec<PhysicalSortExpr> = config
            .primary_keys
            .iter()
            // add aux sort cols to sort expr
            .chain(config.aux_sort_cols.iter())
            .map(|pk| {
                let col = Column::new_with_schema(pk.as_str(), &config.schema.0)?;
                Ok(PhysicalSortExpr {
                    expr: Arc::new(col),
                    options: SortOptions::default(),
                })
            })
            .collect::<Result<Vec<PhysicalSortExpr>>>()?;
        let sort_exec = Arc::new(SortExec::new(sort_exprs, Arc::new(recv_exec)));

        // see if we need to prune aux sort cols
        let exec_plan: Arc<dyn ExecutionPlan> = if config.aux_sort_cols.is_empty() {
            sort_exec
        } else {
            let proj_expr: Vec<(Arc<dyn PhysicalExpr>, String)> = config
                .schema
                .0
                .fields
                .iter()
                .filter_map(|f| {
                    if config.aux_sort_cols.contains(f.name()) {
                        // exclude aux sort cols
                        None
                    } else {
                        Some(col(f.name().as_str(), &config.schema.0).map(|e| (e, f.name().clone())))
                    }
                })
                .collect::<Result<Vec<(Arc<dyn PhysicalExpr>, String)>>>()?;
            Arc::new(ProjectionExec::try_new(proj_expr, sort_exec)?)
        };

        let mut sorted_stream = exec_plan.execute(0, async_writer.sess_ctx.task_ctx())?;

        let mut async_writer = Box::new(async_writer);
        let join_handle = tokio::task::spawn(async move {
            while let Some(batch) = sorted_stream.next().await {
                match batch {
                    Ok(batch) => {
                        async_writer.write_record_batch(batch).await?;
                    }
                    // received abort singal
                    Err(_) => return async_writer.abort_and_close().await,
                }
            }
            async_writer.flush_and_close().await?;
            Ok(())
        });

        Ok(SortAsyncWriter {
            sorter_sender: tx,
            _sort_exec: exec_plan,
            join_handle,
        })
    }
```
* `let receiver_stream_builder = RecordBatchReceiverStream::builder(schema.clone(), 8);`
* `let tx = receiver_stream_builder.tx();`
tx是一个Sender，缓存块数为8
* `let recv_exec = ReceiverStreamExec::new(receiver_stream_builder, schema);`
```
pub struct ReceiverStreamExec {
    receiver_stream_builder: AtomicRefCell<Option<RecordBatchReceiverStreamBuilder>>,
    schema: SchemaRef,
}

impl ReceiverStreamExec {
    pub fn new(receiver_stream_builder: RecordBatchReceiverStreamBuilder, schema: SchemaRef) -> Self {
        Self {
            receiver_stream_builder: AtomicRefCell::new(Some(receiver_stream_builder)),
            schema,
        }
    }
}
```
