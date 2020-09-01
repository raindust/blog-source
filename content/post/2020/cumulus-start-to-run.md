---
title: "Cumulus测试环境搭建"
date: 2020-08-30T21:24:19+08:00
lastmod: 2020-08-30T21:24:19+08:00
keywords: ["polkadot", "blockchain"]
description: ""
tags: ["polkadot", "blockchain"]
categories: ["polkadot", "blockchain"]
---

## Cumulus是什么？

相信了解Polkadot相关技术的人对Substrate一定不陌生，那Cumulus是什么呢？Cumulus与Substrate一起构成了目前的Polkadot跨链开发工具包（英文为[Parachain Development Kit](https://wiki.polkadot.network/docs/en/build-pdk)简称PDK），它实际上是一个基于Substrate的扩展，可以方便地将一条Substrate链变成Polkadot的平行链（Parachain）。

在本文中知道这么多就足够了，我会先带大家基于Cumulus构建一条Polkadot兼容的平行链，让大家有一个具体的印象。在建立了对Cumulus的基础印象之后，会在后续的文章里进一步介绍Cumulus的组成及基本原理等。

好了，那我们开始吧。

## 环境准备

为了排除干扰让大家能快速上手，本文将构建一个最简单的跨链网络。其中包含两个中继链（Relay Chain）的节点，分别是Alice和Bob；以及一个平行链的验证节点（Validator）。

### 语言环境

首先，我们需要准备rust语言的相关环境，这包括编译器rustc、包管理器cargo、工具链安装工具rustup，除了与平台相关的工具链之外我们还需要安装wasm工具链。如果你对rust还不够熟悉的话，好消息是substrate中有一个安装所有相关依赖的脚本，只要运行如下命令就可以搞定：

```shell
curl https://getsubstrate.io -sSf | bash -s -- --fast
```

### 编译中继链节点

准备好语言环境后，我们先来编译中继链节点。因为我们是要演示兼容polkadot的跨链网络，这里要编译的节点自然是要用polkadot源码了。首先通过如下命令下载源码：

```shell
git clone https://github.com/paritytech/polkadot.git
```

由于polkadot和cumulus是并行开发的，如果直接使用polkadot库master分支的最新源码可能会出现不兼容的情况，因此这里需要切换到固定的commit上：

```shell
git checkout fd4b176
```

接下来就是编译了，我们可以通过`cargo build`来编译debug版本，好处是编译速度快，但是占用磁盘空间大，运行速度较慢（其实不至于影响演示）；也可以通过`cargo build --release`来编译release版本，好处是占用磁盘空间小，但是编译速度会慢一点。根据情况自己决定即可，注意debug的编译目录为`targe/debug`，而release的编译目录为`target/release`。如果编译成功，相应编译目录下会出现名为`polkadot`的可执行文件。我们后面执行polkadot相关命令时都会事先假设你已经进入到该可执行文件所在的目录，下文不再一一赘述。

### 编译平行链节点

cumulus的源码地址在[这里](https://github.com/paritytech/cumulus)，不过我们这里将使用[substrate-parachain-template](https://github.com/substrate-developer-hub/substrate-parachain-template)来做部署演示。[substrate-parachain-template](https://github.com/substrate-developer-hub/substrate-parachain-template)和Cumulus之间的关系，与[substrate-node-template](https://github.com/substrate-developer-hub/substrate-node-template)和Substrate之间的关系类似，都是为了简化开发而单独抽离出来的模板。我们先通过如下命令克隆代码：

```shell
git clone https://github.com/substrate-developer-hub/substrate-parachain-template.git
```

同样完成克隆后进入到源码目录，为了排除潜在的运行不兼容情况，我们也需要切换到特定commit下：

```shell
git checkout 2710c42
```

同样的，我们可以通过`cargo build`或`cargo build --release`来编译。编译完成后会在相应的目录下会产生`parachain-collator`可执行文件。

## 运行

### 准备链配置文件

如上文所说，我们这里会准备一个最简单的跨链网络，只包含两个中继链（Relay Chain）的节点和一个平行链的验证节点（Validator）。cumulus-workshop中已经有定义好的chain spec，我们这里通过wget下载：

```shell
wget -O spec.json https://substrate.dev/cumulus-workshop/specs/rococo-local.json
```

我们后面会假设这里下载的`spec.json`文件已经拷贝到`polkadot`和`parachain-collator`可执行文件所在目录，我们后续会在命令行中直接引用它。

### 运行中继链

我们先通过如下命令启动Alice节点：

```shell
./polkadot --chain spec.json --tmp --ws-port 9944 --port 30333 --alice
```

这里大概解释一下命令参数：

- chain：用来指定链配置文件，也就是我们上文中下载的配置文件
- tmp：会标识运行在一个临时目录，这样的好处是不用自己手动再去清理之前的链数据目录
- ws-port：指定web socket的端口，由于web ui是通过web socket与链上交互，因此这里指定的就是web服务的端口
- port：指定p2p连接端口，当其他节点与该节点建立p2p连接时会用到该端口
- alice：指定我们这里使用预先在链中预定的alice账户，alice具有sudo权限，我们后续的侧链设置也会通过alice来实现

当启动之后，我们会看到命令行中会有类似如下的输出：

```
Local node identity is: 12D3KooWGLdqHXgE9ymNVJuHytvTvvicfH5PrvpNUL7FcSpFkSdj
```

这里的`12D3KooWGLdqHXgE9ymNVJuHytvTvvicfH5PrvpNUL7FcSpFkSdj`是我们刚才启动节点的peer id，我们知道substrate底层采用libp2p做节点间点对点通讯，这里的peer概念实际上出自libp2p，表示p2p网络中的一个节点。我们后面会结合peer id以及之前启动参数中指定的p2p端口参数来链接到刚才启动的节点（我们后面简称为alice节点）

如果只运行alice节点，我们会发现链的高度始终为0，这是因为我们在chain spec中已经指定了需要两个节点共同验证才能出块。接下来我们就启动中继链中的第二个节点，这个节点使用了链中预定义的bob账户，因此我们后面也可以简称为bob节点。启动的命令如下：

```shell
./polkadot --chain spec.json --tmp --ws-port 9955 --port 30334 --bob --bootnodes /ip4/127.0.0.1/tcp/30333/p2p/12D3KooWGLdqHXgE9ymNVJuHytvTvvicfH5PrvpNUL7FcSpFkSd
j
```

可以看到与启动alice节点相比多了一个名为`bootnodes`的参数，这是因为我们需要让bob与alice节点建立p2p连接，因此在这里指定了alice的libp2p地址。我们从libp2p的地址段中可以看到alice和bob都是在同一台机器上，因此IP地址使用了`127.0.0.1`。而`30333`端口则是我们在启动alice节点时指定的p2p端口，最后的peer id也是上文中我们从alice启动之后从命令行中拷贝过来的。

启动后bob节点与alice节点建立连接后，我们就可以在命令行中看到正常出块的日志了。

### 运行平行链

首先，我们进去到平行链编译后可执行文件所在模流，通过如下命令生成一个Wasm的验证函数：

```shell
./parachain-collator export-genesis-wasm > para-200-wasm
```

当生成成功后，同级目录下会生成一个para-200-wasm文件，这个文件当我们后续在中继链上注册该平行侧链时会用到。

然后我们通过如下命令启动平行链节点：

```shell
./parachain-collator --tmp --ws-port 9977 --port 30336 --parachain-id 200 --validator -- \ 
 --chain spec.json --bootnodes /ip4/127.0.0.1/tcp/30333/p2p/12D3KooWGLdqHXgE9ymNVJuHytvTvvicfH5PrvpNUL7FcSpFkSd
```

注意命令行中第一张最末尾用单独的`--`隔开了，这是因为将要启动的平行链节点同时承担了平行链的验证人（实际上的名称为collator，不过暂时不需要太在意，我会在后续文章中说明）和中继链节点两个角色：

- `--`之前是平行链节点配置项，陌生的参数有标识平行链ID值的`parachain-id`，以及标识为验证人的`validator`
- `--`之后的是作为普通的中继链节点的配置项，相关参数我们在之前已经有接触这里就不再介绍

运行成功后，你会看到日志中会同时显示中继链和平行链的链高，分别以`[Relaychain]`和`[Parachain]`标识。我们可以看到中继链高在不断的增加，而平行链的高度始终为0，这是因为平行链需要在中继链上进行注册。

此外，在日志中我们会看到如下日志：

```shell
Parachain genesis state: 0x00000000000000000000000000000000000000000000000000000000000000000097600fcfeeed0c7c2e7d922081a466c4c00f2af96ce17f4a07d59e7d47e8354b03170a2e7597b7b7e3d84c05391d139a62b157e78786d8c082f29dcf4c11131400
```

这里的`genesis state`值我们也需要先存下来，后续在注册侧链时会有用到。

好了，相关信息已经准备齐全，我们接下来会在Web UI中完成注册的过程。

## 注册侧链

如果在远程部署节点，可以通过`ssh -L 9944:127.0.0.1:9944 -N -T [your-ip]`和`ssh -L 9977:127.0.0.1:9977 -N -T [your-ip]`分别为alice节点和平行链节点建立Web UI隧道。

我们这里使用默认具有sudo权限的alice节点配置平行链。首先打开[https://polkadot.js.org/apps](https://polkadot.js.org/apps)链接，通过左上角的网络选择Local Node：`127.0.0.1:9944`，其中`9944`端口为alice开放的Web Socket端口。

接下来，我们点击`开发者->撤销`菜单按钮，选择`registrar::registerPara方法`，按如下图所示填入注册参数：

![](/images/2020/cumulus_start_to_run/register-para-chain.png)

其中`ParaId`是我们在启动平行链时给定的Id值（目前为200）；`Validation Code`为我们在运行平行链前预生成的Wasm，点击上传即可；`HeadData`需要我们填入我们在启动平行链后存下来的`Parachain genesis state`。

填写完毕后点击最下方的`Submit Sudo`按钮，等到交易上链后我们回到平行链的日志，可以看到带`[Parachain]`标识的平行链高度已经开始增长。

至此polkadot的跨链网络就搭建完成了。我们可以通过`Network->平行链`菜单项查看当前已经注册完成的平行链，由于我们只注册了一条平行链，因此这里只有一条记录：

![](/images/2020/cumulus_start_to_run/para-chains.png)

此外，我们也可以在[https://polkadot.js.org/apps](https://polkadot.js.org/apps)用和切换到alice节点同样的方法切换到`127.0.0.1:9977`(我们之前为平行链节点配置了9977的Web Socket端口)，通过Web UI查看平行链的运行情况。这些就留给大家自行探索啦。

## 后记

Cumulus目前还处于快速开发期，之前基于部署的Cumulus部署的文档很快会过时，包括[polkdot的wiki文档](https://wiki.polkadot.network/docs/en/build-deploy-parachains)甚至是Cumulus主页的README也无法保证可以成功部署。而[Polkadot git仓库](https://github.com/paritytech/polkadot)也中也会有多个对应cumulus的分支，但也没有官方说明要以哪个为准。关于部署教程，曾经有一个`joshy-workshop`用来指导大家部署跨链网络，但后来相关链接都已经失效……正因为如此，在很长一段时间里Cumulus对社区的友好性很差。

在四周之前Cumulus发布了第一个版本，[Substrate Developer Hub](https://github.com/substrate-developer-hub)上也推出了相应的[substarte-parachain-template](https://github.com/substrate-developer-hub/substrate-parachain-template)和[cumulus-workshop](https://github.com/substrate-developer-hub/cumulus-workshop)，这对我们社区开发者来说无疑是一个福音，而本文也是参考了[cumulus-workshop](https://github.com/substrate-developer-hub/cumulus-workshop)实现的。

接下来我计划会再整理出一些cumulus原理和开发相关的文章，希望能为大家Polkadot跨链开发提供帮助。我们下期再会。