> Rust编程语言的官方网站是 https://www.rust-lang.org/  

Rust运行时开销小、高效，可以直接操作内存，甚至于没有运行库。

> Rust的标准库文档位于https://doc.rust-lang.org/std/  
> Rust编译器的源码位于https://github.com/rust-lang/rust  
> 语言设计和相关讨论位于https://github.com/rust-lang/rfcs

### Rust 版本
Rust编译器的版本号采用了“语义化版本号”​（Semantic Versioning）规划。在这个规则之下，版本格式为：`主版本号．次版本号．修订号`。版本号递增规则如下：  
❏ 主版本号：当你做了不兼容的API修改  
❏ 次版本号：当你做了向下兼容的功能性新增  
❏ 修订号：当你做了向下兼容的问题修正

为了兼顾更新速度以及稳定性，Rust使用了多渠道发布的策略：  
❏ nightly版本：是每天在主版本上自动创建出来的版本，这个版本上的功能最多，更新最快，但是某些功能存在问题的可能性也更大。因为新功能会首先在这个版本上开启，供用户试用。  
❏ beta版本：每隔一段时间，将一些在nightly版本中验证过的功能开放给用户使用。它可以被看作stable版本的“预发布”版本。  
❏ stable版本：正式版，每隔6个星期发布一个新版本，一些实验性质的新功能在此版本上无法使用。它也是最稳定、最可靠的版本。

<span style="color: red;font-weight: bold;">※</span> 在nightly版本中使用试验性质的功能，必须手动开启feature gate。就是要在当前项目的入口文件中加入一条#! [feature(…name…)]语句。否则是编译不过的。等到该功能正式发布，stable版本会警告你可以去掉这个feature gate了。

### 安装开发环境

在官网上下载rustup-init程序，以命令行形式安装。  
在Windows平台上，Rust支持两种形式的ABI（Application Binary Interface）​，一种是原生的MSVC版本，另一种是GNU版本。MSVC版本与Windows兼容性更好，但是需要下载VisualC++的工具链，详细内容可以参考[rustup文档](https://rust-lang.github.io/rustup/installation/windows.html)。  
安装完成之后，在$HOME/.cargo/bin文件夹下可以看到一系列的可执行程序：rustc.exe是编译器，cargo.exe是包管理器，cargo-fmt.exe和rustfmt.exe是源代码格式化工具，rust-gdb.exe和rust-lldb.exe是调试器，rustdoc.exe是文档生成器，rls.exe和racer.exe是为编辑器准备的代码提示工具，rustup.exe是管理这套工具链下载更新的工具。  
<span style="color: red;font-weight: bold;">※</span> 如果想指定Rust的安装位置，在运行rustup-init程序前，需要在系统变量添加如下变量：

```shell
    CARGO_HOME=C:\Develop\Rust\.cargo
    RUSTUP_HOME=C:\Develop\Rust\.rustup
```

###### 用rustup管理工具链

```rust
    // 更新rustup本身
    $ rustup self update
    // 卸载rust所有程序
    $ rustup self uninstall
    // 更新工具链
    $ rustup update
    // 安装nightly版本的编译工具链
    $ rustup install nightly
    // 设置默认工具链是nightly版本
    $ rustup default nightly
```

###### 设置环境变量

为了提高访问速度，中国科技大学Linux用户协会（USTC LUG）提供了一个代理服务，其官方网址为<https://lug.ustc.edu.cn/wiki/mirrors/help/rust-static>  

```shell
    export RUSTUP_DIST_SERVER=https://mirrors.ustc.edu.cn/rust-static
    export RUSTUP_UPDATE_ROOT=https://mirrors.ustc.edu.cn/rust-static/rustup
```

###### 远程仓库

Rust官方工具链还提供了重要的包管理工具cargo.exe，通过这个工具轻松导入或者发布开源库。官方的管理仓库在<https://crates.io/>。大型项目往往需要依赖这些开源库，cargo会帮我们自动下载编译。  
为了解决网络问题，需要利用USTC提供的代理服务，使用方式为，在$HOME/.cargo目录下创建一个名为config的文本文件，其内容为：  

```rust
    [source.crates-io]
    registry = "https://github.com/rust-lang/crates.io-index"
    replace-with = 'ustc'
    [source.ustc]
    registry = "git://mirrors.ustc.edu.cn/crates.io-index"
```

###### RLS

RLS（Rust Language Server）是官方提供的一个开源的标准化的编辑器增强工具。项目地址在<https://github.com/rust-lang-nursery/rls>。它是一个单独的进程，通过进程间通信给编辑器或者集成开发环境提供一些信息，实现比较复杂的功能，比如代码自动提示、跳转到定义、显示函数签名等。安装最新的RLS的方法为：  

```rust
    // 更新rustup到最新
    rustup self update
    // 更新rust编译器到最新的nightly版本
    rustup update nightly
    // 安装RLS
    rustup component add rls --toolchain nightly
    rustup component add rust-analysis --toolchain nightly
    rustup component add rust-src --toolchain nightly
```

### Rust的帮助命令

1. rustc -h ：查看rustc的基本用法；  
2. cargo -h ：查看cargo的基本用法；  
3. rustc -C help ：查看rustc的一些跟代码生成相关的选项；  
4. rustc -W help ：查看rustc的一些跟代码警告相关的选项；  
5. rustc -Z help ：查看rustc的一些跟编译器内部实现相关的选项；  
6. rustc --help -V ：查看rustc的更详细的选项说明。  

