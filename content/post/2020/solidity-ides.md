---
title: "Solidity开发环境"
date: 2020-06-08T19:51:57+08:00
lastmod: 2020-06-08T19:51:57+08:00
keywords: ["blockchain", "solidity"]
description: ""
tags: ["blockchain", "solidity"]
categories: ["blockchain"]
---

## 开发环境比较

结合个人开发习惯，找到以下三个候选开发环境：

### IntelliJ插件

[插件网页网址](https://plugins.jetbrains.com/plugin/9475-intellij-solidity)

优势：

- 语法高亮
- 支持跳转、查找引用
- IntelliJ系列IDE很熟悉

不足：

- 不支持编译
- 不支持调试
- 插件基本上不再更新

### VS Code插件

有多个插件选择，最为流行的是[这个](https://github.com/juanfranblanco/vscode-solidity)

虽然在跳转、查找引用等方面稍逊IntelliJ一筹，不过总体开发体验还是不错的

优势：

- 更丰富的语法高亮
- 支持编译
- 支持跳转
- 保存自动格式化
- lint检查

不足：

- 不支持debug
- 不支持查找引用
- 智能输入感应偏保守，很多已声明变量都得靠手打

### remix ide

以太坊官方提供的网页版IDE，github网址可以参见[这里](https://github.com/ethereum/remix-ide)，虽然编写代码体验不及本地IDE，但对各个版本的支持以及运行调试环境的支持都是最好的

优势：

- 便捷的版本切换
- 支持调试
- 支持在线部署

不足：

- 不支持跳转和查找引用
- 网上开发网速不好的时候，需要切compiler比较影响开发效率
- 智能输入感应有点激进（未作语义分析的结果）

### 小结

IntelliJ插件支持度不高且基本上已不再维护，可以第一个排除。

使用VS Code还是remix-ide各有利弊：VS Code看起来很懂solidity，包括语法检查、选择高亮、跳转等等，但关键的智能输入恨不给力，像我这种习惯了不打完单次的懒人感觉很难受；要说部署和调试的话，还是得在remix-ide上来做，打字至少不用全打完，不过代码和文件多了的时候由于网页开发本身的局限性感觉会陷入细节中去。

两者可以根据个人喜好来选择，或者结合者来：平时敲代码用VS Code，要调试的话再搬到remix-ide上去。

## VS Code

### 环境准备

1. 可在VS Code插件中搜索solidity（注意是小写）出现的第一个就是（开发者名为Juan Blanco，也可以通过这个进一步确认）

2. 安装完成之后，需要配置solidity的版本，因为我们需要的是0.4.24版本，可以先下载[0.4.24可执行](https://github.com/ethereum/solc-bin/blob/gh-pages/bin/soljson-v0.4.24%2Bcommit.e67f0147.js)，放置到合适的路径下
3. 在VS Code配置中指定使用该版本，可以在VS Code的通过菜单打开Preference->Settings对话框，搜索solidity找到"Compile Using Local Version"配置项，填写上一步放置的路径

![设置示意图](/images/2020/solidity-ides/settings.jpg)

## remix-ide

### 环境准备

remix-ide支持在线开发，也支持离线安装。下面给出通过Node.js的离线安装步骤：

1. 选择合适的Node.js版本，这里建议使用10.X版本，remix-ide就是在这个大版本下测试的，Node版本切换的可以使用[nvm](https://github.com/nvm-sh/nvm)
2. 通过`npm install remix-ide -g`命令全局安装
3. 成功安装后输入`remix-ide`命令启动服务，然后通过`http://localhost:8080`访问IDE
4. 在IDE的插件管理器中安装所需插件：`Solidity compiler`是用来编译的，代码编写时最好先安装该插件用于阶段性错误检查；`Debugger`和`Deploy & run transactions`在调试、试运行时很有用，也可以事先安装好；其他插件可以后续按需安装

## IntelliJ

直接在IDE中搜索安装插件“Intellij-Solidity”，安装后重启即可。

由于IntelliJ不支持编译和调试，因此不建议作为主要开发，如果有兴趣可自己探索