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