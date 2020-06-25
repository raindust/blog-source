---
title: "rust包使用入门"
date: 2020-06-25T13:21:13+08:00
lastmod: 2020-06-25T13:21:13+08:00
keywords: ["rust"]
description: ""
tags: ["rust"]
categories: ["rust"]
---

在开始学习rust的时候，相信不少人和我一样对rust中的包使用一头雾水，rust包中存在一些陌生的概念，我觉得正是因为这些不同于其他语言的概念在阻碍我们的认知。接下来我们先过一下这些概念，理清概念之间的关系，之后再通过对包的常用操作加深理解。
## 概念
### crate
我们理解新事物时一般会通过与已知事物建立连接，而编程过程中如果创造一些全新的概念会增加我们的心智负担，所以我们也会与日常事务关联来加深印象。常见的单词如bucket（桶）、slot（水槽）、library（图书馆）、repository（储藏室），都是我们编程中常见的单词，它们都代表某种形式的容器。

我们回过头来看看`crate`这个rust包管理中的概念，crate对应的英文单词有板条箱的含义。那么，板条箱是什么呢？一个典型的板条箱长这样：

![板条箱](/images/2020/introduce_to_rust_package/crate.jpg)

可以看出来板条箱主要是用木条板做成的，板之间有缝隙通风性好可以装透气的物品，此外拆装方便因此在运输货物的时候也很有用，我之前让德邦物流帮忙运行李时也用到了简易版的板条箱用来固定和保护诸如自行车、电脑这类易损物品。那么我们这里说的板条箱和rust中的象征意义是一回事吗？我觉得是的，不信你可以看看[crates官网](https://crates.io/)上的图标。

说完crate这个词，我们再看看`cargo`：cargo在英文中有轮船、飞机等大型交通工具装载的货物这个意思，而cargo在rust语言中作为包管理工具，负责管理一个个需要运输的crate（板条箱），成功地将它们运送到客户（rust开发者）手中。怎么样？理解了这两个单词的真正含义之后是不是觉得rust中的包管理不再那么陌生了？我们再到rust语言中对crate进一步接触吧。

crate是rust包中的核心概念，在我们使用rust编程时会经常碰到，但是在不同的场景下意义有所不同，大体说来可以分为“作为包共享的基本单位（编译前的板条箱）”和“代表一个二进制生成单位（编译后的板条箱）”两种类型含义。

我们先看看前者，在[crates官网](https://crates.io/)上共享的一个包，以及rust文档中（可以通过cargo doc生成）的crate都可以看做是这个含义的典型代表。作为包共享的基本单位，一个crate源码中需要有对应的cargo.toml文件，用于描述包的基本信息、定义依赖项等。

而后者主要体现在包依赖的路径上，对此我们在[The Rust Programming Language一书对于crate的描述](https://doc.rust-lang.org/book/ch07-01-packages-and-crates.html#packages-and-crates)也能看出一些端倪，书中提到了**crate root**的概念：表示一个作为编译入口的源文件。如果是lib库对应包的src子目录下的lib.rc文件，而可执行则对应包的src子目录下的main.rc文件。我们在导入包时所用到的`crate`关键字其实也指的是这个crate root。

好了，get到crate这个比较陌生的概念之后，我们再看看其他包相关的概念吧，这些概念我们都能从其他语言中看到所以会简单带过。

### package
package在英文中有包的意思，我们在使用node的npm、python的pip、java的maven或者gradle、go语言的mod时操作的都是类似的概念，简单来说就是一个高内聚、可共享的基本单位。

不过我们需要注意在rust中package和crate的区别，我们还是在看看[The Rust Programming Language对此的描述](https://doc.rust-lang.org/book/ch07-01-packages-and-crates.html#packages-and-crates)：一个package由一个或多个crate组成，用于提供一组功能。由此可以看到二者是包含关系，注意当一个package包含多个crate时，一般只会有一个lib库，但是会有多个可执行。其实我们看到package和**作为包共享的基本单位**的crate含义是等价的。

### workspace
当工程逻辑比较复杂时，我们会希望拆分出多个package（可能会有多个用于提供功能的包，以及一个或多个用于编译可执行的包），这时候就需要workspace了。workspace在英文中有工作空间的意思，在不少软件中也都有workspace作为顶层的文件组织概念，比较好理解我们就不再赘述了。
### module
module是模块的意思，这是我们最熟悉不过的单词了，在rust中通过`mod`关键词来表示。需要注意的是rust中的模块有两种定义方式：一种是显示地申明`mod A`；此外，当我们新建一个”*.rs”文件时，也会自动地以文件名生成一个mod，mod支持嵌套定义。

熟悉C++、C#等语言的同学可能会发现这里的mod和namespace（命名空间）的概念很像，只是在rust中会根据源文件自动生成mod。

### scope
scope就是作用域了，有人可能会奇怪为什么在说包管理时会提到作用域，这是因为作用域的概念能帮助我们更好地理解包依赖关系。其实module是在我们真正的代码逻辑之外的一层作用域，当导入（`use`）一个包或者类型的时候，实际上是将其他作用域中的元素导入到当前module所在的作用域。

理解到这里，对于下面这个最简单的例子也许你就不会奇怪了：为什么同一个文件中的单元测试还需要通过super来导入呢？因为我们定义了一个子作用域用来包裹所有的测试函数。

```rust
struct A;

#[cfg(test)]
mod tests {
  use super::A;
  // ...
}
```
-----

到这里我们今天的主角都介绍完了，让我们用一个简单的命令行连线图来示意一下他们的层级关系吧：

```shell
  workspace1
 └── package1
 │   ├── libA
 │   │   ├── mod(crate root)
 │   │       ├── modA_1
 │   │       │   ├── modA_1_tests
 │   │       └── modA_2
 │   └── binB
 │   │   ├── mod(crate root)
 │   └── binC
 │       ├── mod(crate root)
 │           ├── modC_1
 │           ├── modC_2
 └── package2
     ├── libD
         ├── mod(crate root)
             ├── modD_1
             └── modD_2
```

## 导出操作

提到模块导出我们有必要说明一下rust的可见性。其实rust的可见性规则很简单，总结下来只有一个规则：**除了trait之外的其他作用域内部元素都是默认对外不可见的**。有了这个原则之后，我们看看两种常见的module导出吧：

第一种其实是在定义module之间的从属关系。比如我们要在一个crate中添加一个文件`a.rs`，若想让其被正常调用一般会在crate root文件（一般是lib.rs、main.rs）中通过`mod a;`的方式申明，这样实际上是在告诉编译器“module a”是从属于crate root的。从上面我们知道crate root的内部元素默认对外不可见，那我们如果想要让"module a"对包外可见，则可以在申明时加上访问修饰符pub：`pub mod a;`，这样外部就可以访问到"module a"了。

这里有两点需要额外说明一下。其一是上面我们通过`pub mod a;`的申明方式只是让外部可以看到"module a"，对于该模块内部的元素也需要添加访问修饰符pub才可见。其二是我们上面只说了一个crate的顶层申明，我们知道在rust的包里也支持通过子目录来组织模块，子目录下的mod.rs文件对该目录内的其他文件组织方式和我们上面提到的crate root文件组织方式是一样的，这里就不再做具体说明了。

第二种导出方式需要用到use关键词。从上文对作用域的说明我们知道，通过use我们可以将其他作用域下的元素导入到当前作用域下。接着如果我们在use之前加上pub修饰符，就可以在其他作用域内访问该作用域use所导入的元素了。这么说可能有点绕，我们看一个实际的例子吧：

```rust
mod aa {
    pub struct A;
}

mod bb {
    pub use super::aa::A;
}

#[test]
fn test() {
    let obj1 = aa::A;
    let obj2 = ba::A;
}
```

可以看到我们通过在"module b"中使用`pub use`的方式，让结构体A在该模块内也可见，然后就可以通过`bb::A`的方式访问到结构体A。不过需要注意的是，`pub use`并不能改变结构体A的可访问性：如果我们将"module a"中对于结构体A的定义从`pub struct A;`更改为`struct A;`，rust编译器就会找你的麻烦。

此外，我们可以通过`as`关键字改变导出的名称，比如下面我通过as修饰符将结构体A改名为B：

```rust
mod aa {
    pub struct A;
}

mod bb {
    pub use super::aa::A as B;
}

#[test]
fn test() {
    let obj1 = aa::A;
    let obj2 = bb::B;
}
```

## 导入操作

导出和导入操作并没有严格的界限，实际上当我们要导出一个元素时，必须先让该元素在该作用域下可见，而这实际上就是在导入。这里我们列一下导入操作中会用到的三个关键字：

- **use**：前面我们已经多次提到了use，它会将其他作用域下的元素导入到当前作用域下
- **mod**：之前也提到了过mod实际上是在定义模块的从属关系
- **extern**：可以通过`extern crate [crateName];`的方式告诉Rust我们需要编译和链接一个外部的元素，不过在2018版本之后我们一般都[不需要再这样显示申明了](https://doc.rust-lang.org/edition-guide/rust-2018/module-system/path-clarity.html)，目前我们一般通过和`#[macro_use]`特性结合来导入外部的宏

由于模块之间的层次关系，我们导入时可能会有多层路径，为了方便表示在rust中支持绝对路径和相对路径两种方式，绝对路径会从crate root开始起算，而相对路径则是从当前作用域所在的模块开始。我们可以通过以下的路径前缀来找到我们想要的元素：

- **crate**：绝对路径的表示方法，会从crate root所在的模块开始起算
- **self**：相对路径的表示方法，指代当前作用域所在的模块
- **super**：相对路径的表示方法，指代当前作用域所在模块的上一级模块
- **模块名**：相对路径的表示方法，当我们可以在当前作用域直接访问到其他模块时，上面的self关键词可以省略

接下来我们在一个包的lib.rs文件里写一个简单的例子加深理解，可以看到我们使用了四种方式访问到了结构体A：

```rust
mod aa {
    pub mod aa_inner {
        pub struct A;
    }

    // 通过super访问
    pub use super::aa::aa_inner::A as B;
}

#[test]
fn test() {
    // 通过crate访问
    let obj1 = crate::aa::aa_inner::A;
    // 通过self访问
    let obj2 = self::aa::aa_inner::A;
    // 直接通过模块名访问
    let obj3 = aa::aa_inner::A;
}
```

说完了前缀我们在看看“后缀”吧。后缀实际上是在限定导入的范围，以下是常用到的后缀：

- **模块名**：将一个模块导入到当前作用域下，可以直接通过该模块名来访问其内部的公共元素
- **类型**：直接讲一个类型导入到当前作用域下，这样就可以直接使用
- **`*`**：会将一个模块下的所有元素都导入到当前作用域，导入后的元素也可以直接使用。注意这样可能会导入很多用不到的元素，这会导致我们编译后的文件膨胀所以要慎用
- **self**：当导入表达式中存在多项时，可以通过self表示当前模块

老规矩我们再看一个例子体会一下区别吧：

```rust
mod aa {
    pub mod aa_inner {
        pub struct A1;
        pub struct A2;
    }

    pub struct A3;
}

mod bb {
    // 导入模块名aa
    use super::aa;

    #[test]
    fn test() {
        let a = aa::aa_inner::A1;
    }
}

mod cc {
    // 导入aa_inner子包下的A1
    use super::aa::aa_inner::A1;

    #[test]
    fn test() {
        let a = A1;
    }
}

mod dd {
    // 导入aa_inner包下的所有公共元素：A1和A2
    use super::aa::aa_inner::*;

    #[test]
    fn test() {
        let a1 = A1;
        let a2 = A2;
    }
}

mod ee {
    // 导入aa包本身，以及aa_inner子包下的A1
    use super::aa::{self, aa_inner::A1};

    #[test]
    fn test() {
        let a1 = A1;
        let a3 = aa::A3;
    }
}

```

------

到这里关于rust包的说明告一段落，希望换一种思路说明能对大家能有所帮助。如果有什么疏漏和错误也希望大家能够留言指正，我们一起学习一起进步。

