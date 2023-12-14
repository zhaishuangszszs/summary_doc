## client write 流程
* 设置表名`let table_name = "test_merge_same_column_i32";`
* 设置客户端连接`let client = Arc::new(MetaDataClient::from_env().await?);`
* 配置LakeSoulIOConfig`let builder = LakeSoulIOConfigBuilder::new()
                .with_schema(schema)
                .with_primary_keys(pks);`
* `let lakesoul_table = LakeSoulTable::for_name(table_name).await?;`
* `lakesoul_table.execute_upsert(batch).await`


### LakeSoulTable

#### LakeSoulTable定义
```
pub struct LakeSoulTable {
    client: MetaDataClientRef,
    table_info: Arc<TableInfo>,
    table_name: String,
    table_schema: SchemaRef,
    primary_keys: Vec<String>,
}
```
#### LakeSoulTable::for_name
```
pub async fn for_name(table_name: &str) -> Result<Self> {
        Self::for_namespace_and_name("default", table_name).await
    }
```
调用`Self::for_namespace_and_name`函数，传入`namespace为"default"`,`table_name`参数
#### LakeSoulTable::for_namespace_and_name
```
pub async fn for_namespace_and_name(namespace: &str, table_name: &str) -> Result<Self> {
        let client = Arc::new(MetaDataClient::from_env().await?);
        let table_info = client.get_table_info_by_table_name(table_name, namespace).await?;
        Self::new_with_client_and_table_info(client, table_info).await
    }
```
1. 设置meatdate客户端连接pg【**metadate原数据存在pg里**】
2. 通过客户端查询元数据`TableInfo`信息给`table_info`
3. 调用`Self::new_with_client_and_table_info(client, table_info)`

#### LakeSoulTable::new_with_client_and_table_info
```
pub async fn new_with_client_and_table_info(client: MetaDataClientRef, table_info: TableInfo) -> Result<Self> {
        let table_schema = schema_from_metadata_str(&table_info.table_schema);
        
        let table_name = table_info.table_name.clone();
        let (_, hash_partitions) = parse_table_info_partitions(table_info.partitions.clone());


        Ok(Self { 
            client,
            table_info: Arc::new(table_info),
            table_name, 
            table_schema,
            primary_keys: hash_partitions
        })
    }
```
1. 查询元数据得到的`table_info`获取`schema,table_name`信息
2. `table_info.partitions.clone()`获取`partitions` `String`信息,在之后根据`; ,`获得`hash_keys range_keys`
3. 返回LakeSoulTable给`lakesoul_table`，主键为`hash_keys`

#### LakeSoulTable::execute_upsert
```
pub async fn execute_upsert(&self, record_batch: RecordBatch) -> Result<()> {
        let client = Arc::new(MetaDataClient::from_env().await?);
        let builder = create_io_config_builder(client, None, false).await?;
        let sess_ctx = create_session_context_with_planner(
            &mut builder.clone().build(), 
            Some(LakeSoulQueryPlanner::new_ref())
        )?;
        
        let schema = record_batch.schema();
        let logical_plan = LogicalPlanBuilder::insert_into(
            sess_ctx.read_batch(record_batch)?.into_unoptimized_plan(), 
            self.table_name().to_string(), 
            schema.deref(),
            false)?
            .build()?;
        let dataframe = DataFrame::new(sess_ctx.state(), logical_plan);
        
        let results = dataframe
            // .explain(true, false)?
            .collect()
            .await?;

        Ok(())
        // Ok(print_batches(&results)?)

    }
```
1. 和pg建立客户端连接
2. 根据client的元数据信息构建`LakeSoulIOConfigBuilder`
3. 传递`config`和`LakeSoulQueryPlanner`构建`SessionContext`
4. 获得要插入`batch`的schema
5. 设置逻辑计划，插入指定的表明中【**可在这实现排序功能**】
6. 执行逻辑计划

### 其它函数

#### MetaDataClient::from_env()
读取环境变量"lakesoul_home"路径，存在则读取该路径下文件中内容到config，之后根据文件内容中"="和换行将内容解析为HashMap。url则获取解析后lakesoul.pg.url=对应的url值。利用tokio_postgres和PG建立连接。

#### create_table()
其中调用client.create_table传入TableInfo ，其中TableInfo 配置了问价路径，uuid等信息

#### create_session_context_with_planner

```
pub fn create_session_context_with_planner(config: &mut LakeSoulIOConfig, planner: Option<Arc<dyn QueryPlanner + Send + Sync>>) -> Result<SessionContext> {
    let mut sess_conf = SessionConfig::default()
        .with_batch_size(config.batch_size)
        .with_parquet_pruning(true);
        // .with_prefetch(config.prefetch_size);

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
    let state = if let Some(planner) = planner {
        SessionState::new_with_config_rt(sess_conf, Arc::new(runtime))
            .with_query_planner(planner)
    } else {
        SessionState::new_with_config_rt(sess_conf, Arc::new(runtime))
    };
        
    Ok(SessionContext::new_with_state(state))
}
```
1. `SessionConfig`根据传入的`config`进行配置同时进行别的可选配置
2. 构建`RuntimeEnv`并限制memory for sort writer
3. 解析`config`配置中`object_store_options`的信息，获得`fs.defaultFS`配置信息【注：`fs.defaultFS` 是 `Hadoop` 配置中一个重要的属性，用于指定 Hadoop 分布式文件系统（HDFS）的默认文件系统 URI。】并将获得的信息，配置到`config.default_fs`中
4. `register_object_store(&fs, config, &runtime)?;` 其中fs为解析得到的URI，根据URI注册object store并根据config的相关信息将路径进行规范化处理。
5. `config.files = normalized_filenames;`5. 修改config.files 的文件地址为规范化后的地址
6. 根据传递的参数`ueryPlanner`，前面的`SessionConfig`,`RuntimeEnv`构建`SessionContext`并返回


## client reade 流程
* `let table_name = "test_merge_same_column_i32";`
设置要读的表明
* `let client = Arc::new(MetaDataClient::from_env().await?);`建立客户端连接
* `let sess_ctx = create_session_context(&mut builder.clone().build())?;`构建`SessionContext`【注：这个指和datdfusion连接的会话】
* `let dataframe = lakesoul_table.to_dataframe(&sess_ctx).await?;`

### LakeSoulTable
#### LakeSoulTable::to_dataframe
```
pub async fn to_dataframe(&self, context: &SessionContext) -> Result<DataFrame> {
        let config_builder = create_io_config_builder(self.client(), Some(self.table_name()), true)
            .await?;
        let provider = Arc::new(LakeSoulTableProvider::try_new(&context.state(), config_builder.build(), self.table_info(), false).await?);
        Ok(context.read_table(provider)?)
}
```
1. 构建config_builder
2. 构建LakeSoulTableProvider

### 其他函数
#### create_io_config_builder
```
pub(crate) async fn create_io_config_builder(client: MetaDataClientRef, table_name: Option<&str>, fetch_files: bool) -> Result<LakeSoulIOConfigBuilder> {
    if let Some(table_name) = table_name {
        let table_info = client.get_table_info_by_table_name(table_name, "default").await?;
        let data_files = if fetch_files {
            client.get_data_files_by_table_name(table_name, vec![], "default").await?
        } else {
            vec![]
        };
        Ok(
            create_io_config_builder_from_table_info(Arc::new(table_info))
            .with_files(data_files)
        )
        
    } else {
        Ok(LakeSoulIOConfigBuilder::new())
    }
}
```
根据client获得元数据来构建LakeSoulIOConfigBuilder

#### create_session_context
内部调用`create_session_context_with_planner`

### provider

#### LakeSoulTableProvider
```
pub struct LakeSoulTableProvider {
    listing_table: Arc<LakeSoulListingTable>, 
    table_info: Arc<TableInfo>,
    schema: SchemaRef,
    primary_keys: Vec<String>,
}
```
##### LakeSoulTableProvider::try_new
```
pub async fn try_new(
        session_state: &SessionState,
        lakesoul_io_config: LakeSoulIOConfig,
        table_info: Arc<TableInfo>,
        as_sink: bool
    ) -> crate::error::Result<Self> {
        let schema = schema_from_metadata_str(&table_info.table_schema);
        let (_, hash_partitions) = parse_table_info_partitions(table_info.partitions.clone());

        let file_format: Arc<dyn FileFormat> = match as_sink {
            true => Arc::new(LakeSoulMetaDataParquetFormat::new(
                    Arc::new(ParquetFormat::new()),
                    table_info.clone()
                ).await?),
            false => Arc::new(LakeSoulParquetFormat::new(
                Arc::new(ParquetFormat::new()), 
                lakesoul_io_config.clone()))
        };
        Ok(Self {
            listing_table: Arc::new(LakeSoulListingTable::new_with_config_and_format(
                session_state, 
                lakesoul_io_config, 
                file_format,
                as_sink
            ).await?),
            table_info,
            schema,
            primary_keys: hash_partitions,
        })
    }
```
1. 根据元数据获得的table_info获取schema,hash_keys信息
2. 