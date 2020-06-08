---
title: "Solidity语法简介"
date: 2020-06-08T19:39:36+08:00
lastmod: 2020-06-08T19:39:36+08:00
keywords: ["blockchain", "solidity"]
description: ""
tags: ["blockchain", "solidity"]
categories: ["blockchain"]
---

以下为按个人理解对solidity语法中一些概念的简单总结，内容不多重在归类梳理，这里分享给大家希望能有所帮助。

## 关键字

### 实体

- contract：用来定义一个合约，可以理解是其他语言中的类，可以包含变量和方法，其中可以定义一个构造方法（constructor方法），内部也可以定义struct和enum
- struct：定义一组数据的集合
- enum：枚举类型
- library：库，用来定义一些公共的结构体、枚举，和方法
- function：方法，这里虽然直译的话应该是函数（方法对应的是method），但solidity中的function终究还是用来定义“一个对象内部的行为”，因此叫做方法更贴切一些
- contructor：一种特殊的方法，构造函数（习惯原因，这里就不纠结为啥不叫构造方法了）

### 可见性修饰符

- public：可以被合约内部和外部访问
- private：只能在合约内部访问，继承合约也无法访问
- internal：可以在合约内部和继承合约访问
- external：只可以被外部访问（内部访问的话也需要以this操作符来访问），以非拷贝时访问，当需要传出大型对象（如数组）时建议视同该修饰符

**注意**方法的默认可见性为public，而变量的默认可见性为internal。

获取你也看出来public和external修饰符使用情况有些重叠，除了上述描述外还可以参考一下这里的[best practises](https://ethereum.stackexchange.com/questions/19380/external-vs-public-best-practices)。

可见性修饰符可作用于变量和方法，后续就不再重复介绍。

### 变量修饰符

- constant：常量修饰符，作用于成员变量，被修饰后变量不可被修改

### 方法修饰符

- view：只读方法，不会修改state
- pure：工具方法，不会读取或修改state
- payable：可被支付修饰符，标识之后才可以接收Ether

### 存储修饰符

- memory：临时（内存中）存储，一般用于暂存临时变量和传参
- storage：永久存储在区块链上，不做显式申明时的默认存储方式

### 其他

- using：用于申明别名，支持for从句
- import：用于导入一个其他文件，支持as、from从句

- require：solidity中的断言方式，类似于其他语言的assert

## 合约

### 抽象合约

至少有一个成员未具体实现时，该合约则为抽象合约。抽象合约的直接影响是无法被实例化，这和Java、C#等我们常用的语言很相似

### 接口

通过`interface`关键字代替`contract`，和抽象合约的主要区别是，接口内部不允许有实现的方法，而不允许有以下成员定义：

- 构造函数
- 成员变量
- 结构体
- 枚举

这里需要注意的是，接口不能继承自接口，从C#过来的同学可能会不适应

### 方法

#### 构造方法

- 构造方法可以不声明，不声明时默认为`contructor() public {}`
- 构造方法如果使用internal修饰符，则意味着这是一个抽象合约：因为该合约无法直接实例化
- 继承一个合约时，构造方法继承时通过Base指代父构造方法：如`constructor(uint y) Base(y) public {}`

#### getter方法

如果变量申明为public，则会自动生成相应的getter方法

#### 方法重载

solidity允许方法重载（overloading），即相同的方法名允许定义不同的参数，然后调用时根据传入参数来决定具体使用哪个方法

### 函数修改器

用来改变一个方法的行为，作用类似于Java、C#、Rust等语言的Attribute，可以查看一个官方的简答示例：

```
pragma solidity ^0.4.0;

contract owned {
    function owned() { owner = msg.sender; }
    address owner;

    // This contract only defines a modifier but does not use
    // it - it will be used in derived contracts.
    // The function body is inserted where the special symbol
    // "_;" in the definition of a modifier appears.
    // This means that if the owner calls this function, the
    // function is executed and otherwise, an exception is
    // thrown.
    modifier onlyOwner {
        if (msg.sender != owner)
            throw;
        _;
    }
}


contract mortal is owned {
    // This contract inherits the "onlyOwner"-modifier from
    // "owned" and applies it to the "close"-function, which
    // causes that calls to "close" only have an effect if
    // they are made by the stored owner.
    function close() public onlyOwner {
        selfdestruct(owner);
    }
}
```

可以看到修改器放在方法返回值之前的位置，用于修饰一个方法。

修改器中的下划线`_`起到占位符的作用。

### selfdestruct

销毁操作符，会从区块链中删除合约，如果合约地址内存在Ether会转到参数制定的地址中去

## 库

申明时通过关键字`library`标识， 在虚拟机中运行时只会被加载一次，用来存放一些公共逻辑供不同合约使用。

以下为库的一些限制：

- 不能定义成员变量
- 不能继承或被继承
- 不能用来接收Ether

## 详细参考

- [官方语法解释](https://solidity.readthedocs.io/en/v0.4.24/solidity-in-depth.html)
- [中文翻译](https://www.tryblockchain.org/Solidity-source-mapping.html)

