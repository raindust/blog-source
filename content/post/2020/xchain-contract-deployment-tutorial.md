---
title: "XuperChain从节点部署到合约调用完整向导"
date: 2020-06-08T20:12:41+08:00
lastmod: 2020-06-08T20:12:41+08:00
keywords: ["blockchain", "xuper"]
description: ""
tags: ["blockchain", "xuper"]
categories: ["blockchain"]
---

XuperChain官网分别提供了节点部署、创建合约账户、调用合约等操作向导，但没有形成统一的文档进行串联，为了让大家快速入门XuperChain的合约部署调用操作，这里提供一个完整的向导供大家参考。

这里假设你已经下载[源码](https://github.com/xuperchain/xuperchain)，并且成功编译（执行make即可，这里不再做额外说明），进入到可执行文件xchain及xchain-cli所在目录下：

### 启动链

1. 初始化链上配置：

   ```shell
   ./xchain-cli createChain
   ```

2. 启动链

   ```shell
   nohup ./xchain &
   ```

3. 成功启动后可以通过以下命令查看链上状态：

   ```shell
   ./xchain-cli status
   ```

   可在json类型的返回值中找到键为`height`，其值为当前链高

### 准备合约账户

1. 调用cli命令创建一个合约账户：

   ```shell
   ./xchain-cli account new --account 1111111111111111 --fee 1000
   ```

   这里指定账户名为`1111111111111111`，创建成功后相应的合约地址为`XC1111111111111111@xuper`，其中`xuper`为默认启动链时默认分配的链名称

2. 通过如下命令查询并验证新生成的合约账户ACL相关信息：

   ```
   xchain-cli acl query --account XC1111111111111111@xuper
   ```

3. 准备符合权限的地址列表

   首先在`data`目录下创建子目录`acl`：

   ```shell
   mkdir data/acl
   ```

   调用如下命令生成地址列表文件：

   ```shell
   echo "XC1111111111111111@xuper/dpzuVdosQrF2kmzumhVeFQZa1aYcdgFpN" > data/acl/addrs
   ```

4. 向新生成的合约账户中转账

   ```shell
   ./xchain-cli transfer --to XC1111111111111111@xuper --amount 100000000 --keys data/keys/
   ```

5. 成功转账后可以查询合约账户中的余额

   ```
   ./xchain-cli account balance XC1111111111111111@xuper
   ```

   这里会返回余额和步骤2中指定的`amount`相同

### 编译合约

1. 进入到合约代码所在位置，运行如下命令对go语言源码进行wasm合约编译：

   ```shell
   GOOS=js GOARCH=wasm go build counter.go
   ```

   如果编译成功会生成一个名为counter的可执行文件

2. 将可执行拷贝到xchain程序目录下：

   ```shell
   cp counter [your path]/data/blockchain/xuper/wasm/
   ```

### 部署合约

1. 通过如下命令进行试生成合约原始交易

   ```shell
   ./xchain-cli wasm deploy --account XC1111111111111111@xuper --cname counter -m -a '{"creator": "xchain"}' -A data/acl/addrs -o tx.output
    --keys data/keys --name xuper --runtime go data/blockchain/xuper/wasm/counter
   ```

   如果试生成成功会返回如下响应消息：

   ```shell
   contract response:
   The gas you cousume is: 5182110
   You need add fee
   ```

2. 添加步骤1中的返回fee值后重复步骤1中的命令：

   ```shell
   ./xchain-cli wasm deploy --account XC1111111111111111@xuper --cname counter -m -a '{"creator": "xchain"}' -A data/acl/addrs -o tx.output
    --keys data/keys --name xuper --runtime go data/blockchain/xuper/wasm/counter --fee 5182110
   ```

   若成功在相同目录下生成名为`tx.output`的文件

3. 对步骤2中生成的原始交易签名

   ```shell
   ./xchain-cli multisig sign --tx tx.output --output sign.out --keys data/keys/
   ```

   签名成功后在相同目录下生成对原始交易的签名文件`sign.out`

4. 将原始交易及签名发送到链上

   ```shell
   ./xchain-cli multisig send --tx tx.output sign.out sign.out
   ```

   发送成功后会返回交易的Hash值

5. 查询合约账户验证部署结果

   ```shell
   ./xchain-cli account contracts --account XC1111111111111111@xuper
   ```

   返回结果中包含刚才生成原始交易时指定的合约名则说明部署成功

### 调用合约

1. 通过如下命令试生成合约调用的原始交易

   ```shell
   ./xchain-cli wasm invoke counter -a '{"key":"counter"}' --method increase -m --output increase.out
   ```

   如果试生成成功，则会返回如下响应消息：

   ```
   contract response: 1
   The gas you cousume is: 100662
   You need add fee
   ```

2. 添加步骤1中的返回fee值后重复步骤1中的命令：

   ```shell
   ./xchain-cli wasm invoke counter -a '{"key":"counter"}' --method increase -m --output increase.out --fee 100662
   ```

   若生成成功则会在相同目录下生成名为`increase.out`的文件

3. 对步骤2中生成的原始交易签名

   ```shell
   ./xchain-cli multisig sign --tx increase.out --output increase.sign --keys data/keys/
   ```

   签名成功后在相同目录下生成对原始交易的签名文件`increase.sign`

4. 将原始交易及签名发送到链上

   ```shell
   ./xchain-cli multisig send --tx increase.out increase.sign increase.sign
   ```

   发送成功后会返回交易的Hash值

5. 测试调用合约结果

   ```shell
   ./xchain-cli wasm invoke counter -a '{"key":"counter"}' --method increase -m --output increase.out
   ```

   如上述命令所示，我们可以再次调用increase方法，通过合约试运行中返回结果查看存储于链上key为`counter`对应的值变化情况。由于我们已经成功新增过一次计数，这里的返回值应该是`2`，以下为测试中返回的结果：

   ```shell
   contract response: 2
   The gas you cousume is: 100834
   You need add fee
   ```

### 参考链接

1. [XuperChain环境部署](https://xuperchain.readthedocs.io/zh/latest/quickstart.html#basic-operation)
2. [创建合约](https://xuperchain.readthedocs.io/zh/latest/advanced_usage/create_contracts.html)
3. [超级链测试环境使用指南](https://xuperchain.readthedocs.io/zh/latest/test_network/guides.html#id11)