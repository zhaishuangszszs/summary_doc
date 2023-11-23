[rust安装时rustup文档](https://rust-lang.github.io/rustup/installation/index.html#choosing-where-to-install)

* rustup enabling you to easily switch between stable, beta, and nightly compilers and keep them updated.
* rustup On Unix it is located at `$HOME/.cargo/bin`,on Windows at `%USERPROFILE%\.cargo\bin`
* `rustup self uninstall`卸载rust
* **rust自定义安装**通过设置环境变量`CARGO_HOME`和`RUSTUP_HOME`在运行rustup-init之前来实现。
* `RUSTUP_HOME` sets the root rustup folder, which is used for storing installed toolchains and configuration options. `CARGO_HOME` contains cache files used by cargo.
* 安装时选择编译器
`$ rustup set default-host i686-pc-windows-msvc
` 
`$ rustup set default-host x86_64-pc-windows-gnu
`
* 安装后安装编译工具
`rustup toolchain install stable-x86_64-pc-windows-gnu `

* 选择编译工具 
`rustup default stable-x86_64-pc-windows-msvc`
`rustup default stable-x86_64-pc-windows-gnu`
