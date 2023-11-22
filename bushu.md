> rust cargo build一直出现 Blocking waiting for file lock on package cache
如果确定没有多个程序占用，可以删除`rm -rf ~/.cargo/.package-cache`，然后再执行


## 测试
`cargo +nightly test   --package lakesoul-datafusion -- --nocapture`
不行就
`cargo update`

[Rust:test](https://github.com/lakesoul-io/LakeSoul/blob/main/rust/lakesoul-datafusion/src/test/upsert_tests.rs)
[Spark:test](https://github.com/lakesoul-io/LakeSoul/blob/main/lakesoul-spark/src/test/scala/org/apache/spark/sql/lakesoul/commands/UpsertSuiteBase.scala#L557)

* 测试模块的`#[cfg(test)]` 标注告诉 Rust 只在执行 cargo test 时才编译和运行测试代码，而在运行 cargo build 时不这么做


## ubuntu上部署c++使用arrow
部署失败
* 根本原因：Google 放弃了对 Linux 上 32 位 Chrome 的支持，在 64 位系统中更新 apt 时触发错误（启用了多架构）...详细信息如下：http: //www.omgubuntu.co.uk/2016/ 03/修复无法获取 google-chrome-apt-error-ubuntu

[谷歌后的解决办法](https://askubuntu.com/questions/741410/skipping-acquire-of-configured-file-main-binary-i386-packages-as-repository-x)


## lakesoul reader
* **pub(crate)** 使一个程序项在当前 crate 中可见。