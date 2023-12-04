## write部分
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

**之后**
`let file_name = &config.files.last().unwrap();`filename取最新的文件
之后注册出一个`MultiPartAsyncWriter`

**回到**`execute_upsert`
`let _ = writer.write_batch(batch);`
`let _ = writer.flush_and_close();`
写入文件
