[toc]
# <center> 项目理解</center>
## 1.目录结构
### 1. rust虚拟清单类型的工作空间。
<p style="text-indent:2em">rust package 中Cargo.toml内容</p>

```
# SPDX-FileCopyrightText: 2023 LakeSoul Contributors
#
# SPDX-License-Identifier: Apache-2.0

[workspace]
members = ["lakesoul-metadata", "lakesoul-metadata-c", "proto", "lakesoul-io", "lakesoul-io-c"]
resolver = "2"

[profile.release]

```
&emsp;&emsp;rust目录不含有`package`所以不是根package是虚拟清单的工作空间，对于没有主 package 的场景或你希望将所有的 package 组织在单独的目录中时，这种方式就非常适合。  
工作空间特点：
  - 所有的 package 共享同一个 Cargo.lock 文件，该文件位于工作空间的根目录中
  - 所有的 package 共享同一个输出目录，该目录默认的名称是 target ，位于工作空间根目录下
  - 只有工作空间根目录的 Cargo.toml 才能包含`[patch], [replace] 和 [profile.*]`，而成员的 Cargo.toml 中的相应部分将被自动忽略

[workspace]  
* Cargo.toml 中的 [workspace] 部分用于定义哪些 packages 属于工作空间的成员:

选择工作空间
+ 选择工作空间有两种方式：Cargo 自动查找、手动指定 package.workspace 字段。当位于工作空间的子目录中时，Cargo 会自动在该目录的父目录中寻找带有 [workspace] 定义的 Cargo.toml，然后再决定使用哪个工作空间。

选择 package
* 若没有指定任何参数，则 Cargo 将使用当前工作目录的中的 package 。若工作目录是虚拟清单类型的工作空间，则该命令将作用在所有成员上（也可以指定package）
### package目录结构
一个典型的 Package 目录结构如下：
```
.
├── Cargo.lock
├── Cargo.toml
├── src/
│   ├── lib.rs
│   ├── main.rs
│   └── bin/
│       ├── named-executable.rs
│       ├── another-executable.rs
│       └── multi-file-executable/
│           ├── main.rs
│           └── some_module.rs
├── benches/
│   ├── large-input.rs
│   └── multi-file-bench/
│       ├── main.rs
│       └── bench_module.rs
├── examples/
│   ├── simple.rs
│   └── multi-file-example/
│       ├── main.rs
│       └── ex_module.rs
└── tests/
    ├── some-integration-tests.rs
    └── multi-file-test/
        ├── main.rs
        └── test_module.rs

```
这也是 Cargo 推荐的目录结构，解释如下：
* Cargo.toml 和 Cargo.lock 保存在 package 根目录下
* 源代码放在 src 目录下
* 默认的 lib 包根是 src/lib.rs
* **默认的二进制包根**是 src/main.rs
**其它二进制包根**放在 src/bin/ 目录下
* 基准测试 benchmark 放在 benches 目录下
* 示例代码放在 examples 目录下
* 集成测试代码放在 tests 目录下

### lib.rs和main.rs关系
* 没有mian.rs说明是一个**库package**,crate指的就是**lib.rs**。
两个同时存在，说明同时存在两个crate相互独立，同时编译。`mian.rs`中的**crate**指的就是`main.rs`**自己**binary crate。`lib.rs`中指的就是`lib.rs`**自己**library crate。
* main.rs想**使用lib.rs中定义的结构**需要`use package（名字）::<lib.rs中的结构>`，**但是main.rs不是lib.rs的子模块因此，lib.rs模块是pub时才能用**。
* 想要mian.rs**使用lib.rs中的结构**，Cargo.tml中加入lib。同样只能引用pub模块
```
[lib]
name="项目名"//注意不能有连字符等
```


## 模块总结
```
lakesoul-datafusion/
├── Cargo.toml
└── src
    ├── lib.rs
    └── test
        ├── mod.rs
        └── upsert_tests.rs
```
`src/lib.rs`是crate根文件,内容
```
// SPDX-FileCopyrightText: 2023 LakeSoul Contributors
//
// SPDX-License-Identifier: Apache-2.0

#[cfg(test)]
mod test;
```
* 在 `mod test;` 后使用**分号**，而不是代码块，这将告诉 Rust 在另一个与模块**同名的文件**中加载模块的内容。
* 如果需要将**文件夹作为一个模块**，我们需要进行显示指定暴露哪些子模块。我们有两种方法：
> * 在 test 目录里创建一个 mod.rs，如果你使用的 rustc 版本 1.30 之前，这是唯一的方法。
> * 在 test 同级目录里创建一个与模块（目录）同名的 rs 文件 test.rs，在新版本里，更建议使用这样的命名方式来避免项目中存在大量同名的 mod.rs 文件（ Python 点了个 踩）。

## lakesoul-io
cargo.toml中内容
```
[package]
name = "lakesoul-io"
version = "2.3.0"
edition = "2021"
```
由以上知lakesoul-io是一个package,package目录下
```
├── Cargo.toml
└── src
    ├── constant.rs
    ├── datasource
    │   ├── mod.rs
    │   ├── parquet_sink.rs
    │   └── parquet_source.rs
    ├── default_column_stream
    │   ├── empty_schema_stream.rs
    │   └── mod.rs
    ├── filter
    │   ├── mod.rs
    │   └── parser.rs
    ├── hdfs
    │   ├── mod.rs
    │   └── util.rs
    ├── lakesoul_io_config.rs
    ├── lakesoul_reader.rs
    ├── lakesoul_writer.rs
    ├── lib.rs
    ├── projection
    │   └── mod.rs
    ├── sorted_merge
    │   ├── combiner.rs
    │   ├── merge_operator.rs
    │   ├── mod.rs
    │   ├── sorted_stream_merger.rs
    │   └── sort_key_range.rs
    └── transform.rs
```
没有bin目录因此不是二进制的crate
有一个lib.rs因此是库crate
lib.rs的内容
```
pub mod lakesoul_reader;
pub mod filter;
pub mod lakesoul_writer;
pub mod lakesoul_io_config;
pub mod sorted_merge;
pub mod datasource;
mod projection;

#[cfg(feature = "hdfs")]
mod hdfs;

mod default_column_stream;
mod constant;
mod transform;

pub use datafusion::arrow::error::Result;
pub use tokio;
pub use datafusion;
pub use arrow;
pub use serde_json;
```
**#[cfg(feature = "hdfs")]**是条件编译 Features,
cargo.toml中有
```
[features]
default=["hdfs"]//默认enable
hdfs=[]
```
mod hdfs才能编译，或者rustc提供参数

### lakesoul_reader.rs
> **SessionContext**:Main interface for executing queries with DataFusion. Maintains the state of the connection between a user and an instance of the DataFusion engine.

## lakesoul-datafusion
是一个package并且依赖同一工作空间中的其它两个package`lakesoul-io`,`lakesoul-metadata`
> cargo 并不假定工作空间中的 Crates 会相互依赖，所以需要明确表明工作空间中 crate 的依赖关系。
```
[package]
name = "lakesoul-datafusion"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
lakesoul-io = { path = "../lakesoul-io" }
lakesoul-metadata = { path = "../lakesoul-metadata" }
```
