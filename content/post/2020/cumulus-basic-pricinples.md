---
title: "Cumulus基本原理介绍"
date: 2020-09-07T15:26:20+08:00
lastmod: 2020-09-07T15:26:20+08:00
keywords: ["polkadot", "blockchain"]
description: ""
tags: ["polkadot", "blockchain"]
categories: ["polkadot", "blockchain"]
---

我们在上一篇文章带领大家基于Cumulus的模板快速搭建了一个简单的跨链网络，现在让我们回过头来看看Cumulus的基本原理。

## 跨链主要角色介绍

在具体介绍Cumulus之前我们先简单看一下Polkadot跨链网络中的主要角色，建立对Polkadot网络的整体认知对我们了解Cumulus有很大的帮助。

### 跨链的组成

在Polkadot网络里作为中心的“主链”，也会有各种“侧链”。这些侧链中一部分和中心链是同构的（都是基于substrate实现），而另一部分本身就是独立的链（比如：比特币、以太坊等）。Polkadot网络对这些角色做了如下的划分：

- Relay Chain（中继链）：中继链是Polkadot网路的“主链”，为整个网络提供了统一的共识和安全性保障
- Parachain（平行链）：是与中继链同构的侧链，Polkadot网络中大部分计算工作都会委托给各个平行链处理，平行链一般会与具体业务场景结合（如去中心化交易所DEX、去中心化身份认证DID等）
- Bridgechain（桥接链）：一般为与Polkadot相独立的链，通过转接桥（Bridge）和中继链建立链间通信

下图为由上述三种角色组成的跨链网络示意图，出自[这里](https://www.parity.io/a-brief-summary-of-everything-substrate-polkadot/)：

![](/images/2020/cumulus_basic_pricinples/Polkadot_RelayChain.png)

### 主要角色

在Relay Chain和Parachain中会有多个角色，这些角色之间互相配合完成共识出块的工作：

- Validator（验证人）：验证人在Polkdadot网络中扮演重要的角色，在Relay chain中它负责共识出块；同时，当Parachain上有提名出块时，它也会负责验证并在Relay chain的出块中将Parachain区块进行关联
- Collactor（收集人）：收集人会是一个Parachain的全节点，同时也会是一个Relay chain上的轻节点或者全节点，它会帮助验证人收集、验证和提交备选的平行链区块
- Fisherman（钓鱼人）：钓鱼人会监控Polkadot网络，当发现有非法交易或者区块时，通过举报获取奖励
- Nominator（提名人）：提名人不会直接参与共识，而是通过质押token为验证人增加选票，以此来分享验证人的收益

下图为角色之间的关系示意，出自[这里](https://polkadot.network/PolkaDotPaper.pdf)：

![](/images/2020/cumulus_basic_pricinples/roles_of_polkadot.png)

## Cumulus组成

好了，进入正题开始聊Cumulus吧。那么Cumulus属于Relay chain还是Parachain，它可以用来实现哪个角色的功能呢？在回答这个问题之前，我们在看看[官方](https://wiki.polkadot.network/docs/en/build-cumulus#docsNav)给出的Cumulus的定义：

> [Cumulus](https://github.com/paritytech/cumulus) is an extension to Substrate that makes it easy to make any Substrate built runtime into a Polkadot compatible parachain.

根据这个定义我们提取出三个关键词进行一一分析：

- Extension：Cumulus是Substrate的扩展，这意味着Cumulus是基于Substrate实现的，底层还是一个Substrate
- Substrate built runtime：这里的runtime即Substrate [FRAME](https://substrate.dev/docs/en/knowledgebase/runtime/frame) runtime
- Polkadot compatible parachain：最终的结果是一个Polkadot网络中的Parachain，这个转换过程通过一个名叫`register_validate_block!`的宏来实现非常方便

总结一下吧，Cumulus只是一个对Substrate的扩展，它用来把一个Substrate [FRAME](https://substrate.dev/docs/en/knowledgebase/runtime/frame) runtime变得对Polkadot Parachain兼容，这样就可以方便地把一条基于Substrate构建的独立的链，变为Polkadot的平行链（Parachain）。

了解了Cumulus是什么之后，我们看一下Cumulus的三个基本组成部分：

- Cumulus Consensus：是一个让Substrate可以同步Polkadot Relay chain数据的共识引擎。共识引擎的实现方式是，Cumulus内部会维护一个Polkadot Relay chain的节点，通过该节点监听链上数据，自己确定可信任的最高块及已确认区块
- Cumulus Runtime：在Substrate Runtime之上封装（就是通过我们前面提到的`register_validate_block!`宏来封装），使得原Runtime可以被Polkadot的验证人所验证。
- Cumulus Collactor：在前面我们已经介绍过，在Polkadot网络中一个叫Collactor的角色用来帮助Validator准备侧链的备选区块，而这里实现的正是它。

看完上面的定义和基本组成之后，再回到我们最初的疑问你会发现严格意义上链来说Cumulus既不属于Relay chain也不属于Parachain，而是介于二者之间起到链接的作用：Cumulus Consensus用于感知Relay chain的共识逻辑，而Cumulus Runtime会将一个Substrate实现的区块链变为Parachain。

由于（Cumulus）Collactor是起到桥梁作用的角色，所以为了加深理解为Cumulus挂靠一个角色的话，我觉得可以说Cumulus主要是用来实现Collactor的。

## Cumulus运作原理

### Polkadot跨链

在Polkadot跨链系统中，主链保障共识的安全可信，而侧链则无须关心这些。那么主链的信任如何传递到侧链呢？

我们自己可以先想想如果要自己实现可以通过哪些方式：

- 由于侧链没有自己的共识安全保障，所以每一个侧链节点内部可以嵌入一个主链（全节点或者轻节点），这样每个节点都能感知到主链的共识自然也就引入了主链的信任
- 反过来我们也可以在主链这边想办法，使得主链能够（一般以交易的形式）将侧链的状态记录下来，这样因为侧链的状态时经过主链验证的而且在主链留有存证，因此也可以做到信任的传递
- 或者我们可以在主链和侧链之外引入第三方，通过一些机制保证第三方是可信的，然后就可以让中间人进行主、侧链之间的状态链接
- 我们还可以想办法可以让资产可以在链上设置锁定和过期时间，然后利用秘钥交换协议来让两条链上的资产拥有者实现资产互转。这样比较简单但是也只能用于已知两个人之间的资产转移

第三种和第四种方式主要用在相互独立的链中，这两种方式没法保障一条链上的共识安全性，因此范围就缩小到第一种和第二种方式了。而Polkadot跨链是使用的第二种方式，即在主链中以交易的方式记录了侧链的状态存证，这样侧链出块会以主链中的记录的侧链交易为准，通过这种方式保障了侧链的出块安全。侧链的状态存证在Polkadot中也有一个专有名词——candidatae receipts，具体细节我会再写一篇文章做说明。

实际上Polkadot采用的是中继（Relay，从polkadot的主链——Relay chain也能看出端倪）这种跨链技术，除了这种跨链方式之外还有公证人（Notary）、哈希锁定（Hash-Locking）等其他方式，这些实现方式也绕不开我们刚才自己的预想。如果有兴趣可以自己查看对比这些跨链方式的优劣势等。

### 一次完整的侧链打包

在了解了Polkadot跨链的工作方式之后，我们看看Polkadot中为了完成一次侧链区块打包各角色都做了哪些工作，相信走过一次打包流程之后大家会有更清晰的认知。

![](/images/2020/cumulus_basic_pricinples/parachain_block_generation.png)

在具体介绍出块流程之前我们先简要说明一下上图中每个角色：

- normal parachain：我们知道侧链中除了collator之外还存在其他的普通节点，他们会根据自己具体的业务需要产生交易
- collator：收集人我们之前已经提到过，需要注意的是从必要性上来说一条Parachain只需要一个收集人就够了，当然Parachain自己也可以设计自己共识来确定某一时刻的收集人，这为我们留下足够的灵活性
- parachain validator：在一个特定的周期中polkadot网络会有多个验证人，但对于一个侧链而言，只会有部分验证人会直接参与该侧链的验证，而该侧链上的一个具体区块的出块任务也会落到一个确定的验证人身上
- BABE validator：这里加上BABE前缀是想强调该验证人是通过BABE共识确定出的负责当前区块打包的验证人
- polkadot network：这是一个很大的范围，而事实上一个Relay chain的区块也不光会广播给Relay chain，包括Parachain的节点以及诸如钱包这样的轻节点也会收到

好了，让我们回到上面的流程图。首先，normal parachain node会根据业务需求在Parachain中广播交易，collator收集这些交易并打包成Parachain的区块。由于collator和parachain validator会建立连接，collactor会将Parachain的区块转换成一种统一格式的区块（统一格式区块在Relay chain中定义），并将区块及验证相关的数据都一起发给parachain validator。parachain validator完成区块验证后会生成一个包含parachain状态存证（candidatae receipts）的交易并广播到Relay chain网络，负责打包区块的BABE validator收到这条交易之后将该交易打包进下一个区块中，最后将包含parachain存证交易的Relay chain区块广播出去。

Cumulus在我们刚才所说的打包流程中主要做的工作集中在collator和parachain validator的交互过程，具体交互细节及代码说明我会在下一篇文章中说明。那么我们下一篇见啦。

## 参考链接

- https://www.parity.io/a-brief-summary-of-everything-substrate-polkadot/
- https://polkadot.network/PolkaDotPaper.pdf
- https://wiki.polkadot.network/docs/en/build-cumulus#docsNav
- https://github.com/paritytech/cumulus/blob/master/docs/overview.md
- https://polkadot.network/the-path-of-a-parachain-block/

