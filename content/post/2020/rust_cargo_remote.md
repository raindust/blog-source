---
title: "用cargo remote解放你的Mac电脑"
date: 2020-07-29T21:40:15+08:00
lastmod: 2020-07-29T21:40:15+08:00
keywords: ["rust"]
description: ""
tags: ["rust"]
categories: ["rust"]
---

rust编译起来比较慢，而且我的MBP 2018的散热不太好跑起来笔记本发烫，于是就想用手头的一台云主机来替我编译。

虽然找到了[用Clion远程调试](https://medium.com/nearprotocol/remote-development-and-debugging-of-rust-with-clion-39c38ced7cc1)的方法，不过需要用鼠标点来点去，对于习惯用键盘和命令行的我只能表示这不是我的菜。于是我继续寻找发现了[cargo remote](https://github.com/sgeisler/cargo-remote)，安装之后就可以通过类似`cargo remote -r name@host -c -- build --release`这样的命令进行远程编译啦。

如果想进一步偷懒还可以在`~/.config/cargo-remote/cargo-remote.toml`文件中添加类似`remote = "name@host"`的一行配置，这样就可以省去`remote`参数的配置，远程编译命令变成`cargo remote -c -- build --release`。

是不是感觉和本地编译差不多啦？不过上面的命令成功仅限于本地和远程操作系统一样的情况。我本地操作系统是Mac OSX，而远程则是CentOS7，远程编译之后的可执行在本地跑不起来。大概分析一下cargo remote工具发现它的工作原理是这样的，利用`rsync`工具拷贝代码到远程，然后在远程运行`cargo remote -c --`后面的命令，并在编译完成后把结果拷贝到本地。由于`build --release`会默认选择操作系统所在的target（在CentOS下是`x86_64-unknown-linux-gnu`），编译后是linux的可执行，在本地自然是运行不起来的。

不过幸运的是`rustup`工具为我们提供了方便的交叉编译命令，我们只要通过`rustup target add x86_64-apple-darwin`这样的命令就可以添加对Mac OSX平台（darwin是Mac系统的代号）的支持了。

*顺便提一句，如果想对如nightly这样其他的toolchain添加target，可以通过类似`rustup target add x86_64-apple-darwin --toolchain nightly`这样通过参数指定toolchain的方式来添加，运行cargo命令时也需要像`cargo +nightly build --release`这样显示地指定toolchain。*

好了，我们把编译命令改成`cargo remote -c -- build --release --target x86_64-apple-darwin`来远程编译，这时远程编译器会报“*Linking with `cc` failed: exit code: 1*”这样的错误。这是一个gcc链接的错误，接着去rust语言官方issue逛逛，发现有人说这是一个很久没有解决的问题。

后面又在[Everything you need to know about cross compiling Rust programs!](https://github.com/japaric/rust-cross#i-want-to-build-binaries-for-linux-mac-and-windows-how-do-i-cross-compile-from-linux-to-mac)这样一个Github repo中发现类似问题的FAQ，作者给出了一个简短的答案：**不行！** 原因是不同操作系统上C语言的toolchain有差别所以没法直接支持交叉编译。

怎么办，我的远程编译之旅要被判死刑了么？不死心的我在[reddit](https://www.reddit.com/r/rust/comments/6rxoty/tutorial_cross_compiling_from_linux_for_osx/)上发现一个老兄写了一个和我情况类似的教程，顺着这条线索我找到了[osxcross](https://github.com/tpoechtrager/osxcross)这个Github库。

看这个名字就感觉有戏呀，而且star数也上千了，于是我决定试一下。这个库的README上给出的解决方案是：在mac和linux上各拷贝一份代码，先在mac上利用XCode先编译出来Mac OS SDK，然后拷贝到linux系统上进行编译，在Linux编译成功后会在`target/bin`子目录下出现一系列以`x86_64-apple-darwin`开头的可执行和库文件。我们可以按照README的指示进一步把刚才的目录加到`$PATH`中去。

接下来我们需要在`~/.cargo/config`文件中添加下面的配置：

```toml
[target.x86_64-apple-darwin]
linker = "x86_64-apple-darwin19-clang"
ar = "x86_64-apple-darwin19-ar"
```

注意我的系统版本是19，因此在配置中会出现`darwin19`这样的字样，如果你的版本和我不一样需要做更改。当然最简单的办法就是去刚才osxcross编译后的`target/bin`目录下看看对应的文件是什么名字，然后拷贝到这里就可以了。

接下来我们在本地再次运行`cargo remote -c -- build --release --target x86_64-apple-darwin`发现已经build成功啦🎉不过要注意编译后的结果方在本地`target`目录的`x86_64-apple-darwin`子目录（因为我们制定了target）下，而不是像本地默认编译一样`target`下一级就是`release`或者`debug`了。

-----

远程编译还会存在一些小瑕疵：比如远程编译后在本地通过`cargo run`的方式运行还会触发编译，因此最好还是直接采用运行编译好的可执行的方式；还有原生的`cargo remote`工具会在远程的`$HOME`目录下新建一个`remotes-build`子目录来编译，由于每一次编译完成不会清理（这也在情理之中，不清理下次远程编译会很快），如果你的远程`$HOME`目录所在盘空间也捉襟见肘的话可以试试[我的改进版](https://github.com/raindust/cargo-remote)，主要是添加了一个可以配置远程编译目录的`build-root`参数，这个也一样支持在`~/.config/cargo-remote/cargo-remote.toml`中配置默认值哦。