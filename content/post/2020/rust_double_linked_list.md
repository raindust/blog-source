---
title: "Rust双向链表分析之旅"
date: 2020-07-11T15:09:28+08:00
lastmod: 2020-07-11T15:09:28+08:00
keywords: ["rust"]
description: ""
tags: ["rust"]
categories: ["rust"]
---

对rust初学者而言，写算法是很痛苦的事情。一个双向链表就能折磨得人死去活来，不少老外都在抱怨”Why writing a linked list in safe Rust is so damned hard ...“，后来看到一位大神的[playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2015)感觉很过瘾，自己照着实现了一下，接下来我会按自己的实现思路进行拆解。

## 节点定义

我们知道双向链表的基本元素是一个节点（Node)，典型的定义是有一个数据字段以及两个指针字段，分别指向前一个节点和下一个节点（我们后面分别称为前驱节点和后继节点）。

那么写一个rust style的节点定义需要如何写呢？我们可能第一反应是用`Box`，如果按直觉写下来可能是这样的：

```rust
struct Node<T> {
  data: T,
  prev: Box<Node<T>>,
  next: Box<Node<T>>,
}
```

因为我们是要写一个通用的链表，所以这里直接给出的是泛型的定义。

我们知道`Box`是不能分享所有权的（类似于C++中的`auto_ptr`），而双向列表中一个节点会有两个指针指向它（前驱节点的next和后继节点的prev），这样的场景使用Box就不合适了。不过Box可以做单向链表，有兴趣的话看完之后大家也可以尝试用Box做一个简单的单项链表。

顺着这个思路我们会想到`Rc`（类似于C++中的shared_ptr），接下来我们就用Rc来改造一下节点的定义：

```rust
use std::rc::Rc;

struct Node<T> {
  data: T,
  prev: Rc<Node<T>>,
  next: Rc<Node<T>>,
}
```

这样一来节点没多处引用的问题解决了，不过感觉有点做的太过了，因为这样很可能会造成指针循环引用。用过C++智能指针的同学对指针的循环引用可能不陌生，最简单的例子比如：两个对象A和B，它们都有一个智能指针的成员分别指向对方，这样A和B都无法先于对方释放，从而造成内存泄露。

两个对象之间的循环引用比较容易发现，不过多个对象之间的循环引用就不那么明显了。再举个例子，我们在上述循环引用的两个对象A和B之间再加入C，即A引用C，C引用B，B反过来又引用A，这样就构成了三个对象之间的循环引用。如果在这个引用环中不断地加元素，变成1000个元素的循环，还会那么容易发现吗？而且这样涉及的元素更多，危害也会更大。这样的循环引用环现实中也是有的，最常见的就是循环链表。所以当有环形引用关系时，我们一定要留心潜在的内存泄露。

回到我们的代码中，我们看看相邻两个节点的引用关系：我们假设A的next指向B，也就是说A的后继节点是B，那么反过来说B的前驱节点是A，即B的prev指向A。这和我们描述的两个对象间的循环引用现象一致。因此，我们需要改进数据结构打断这种循环引用关系，接下来需要引入的是**弱引用**指针。

我们知道类似于Rc这种智能指针是通过引用计数来实现的，每当增加一个Rc的引用，引用计数就会加1，当引用计数减为0时会自动释放其指向的内存。循环引用的情况引用计数会至少维持引用计数为1，因此总没有机会自动释放。而弱引用指针不同的是，它并不会增加引用计数（或者是不会增加**强引用**计数，即二者的引用计数系统时分开的）。我们接下来就借助rust标准库中的弱引用指针——Weak来改造节点的数据结构如下：

```rust
use std::rc::{Rc, Weak};

struct Node<T> {
  data: T,
  prev: Weak<Node<T>>,
  next: Rc<Node<T>>,
}
```

到这里节点定义已经很接近我们想要的了，但是我们回想一下（非循环）链表的边界——首尾节点的情况：头结点因为是第一个节点，所以是没有前驱的；尾节点因为是最后一个节点，所以是没有后继的。“没有”在编程中我们一般会用空来表达，而在rust中有一个更人性化的泛型枚举来帮我们处理，这个枚举就是Option。

在我们看看刚才的定义，实际上是没有表达为空的能力的。因此，我们需要为prev和next字段加上Option。那么问题来了，我们需要在哪一层加上Option呢，最外层？Node？还是类型T呢？我们定义prev和next字段是为了对应一个节点的引用，因此在Node这里或者最外层会比较合理。另外，我们希望的是可以简单置空的效果，比如`node.prev = None;`这样，所以最外层比较合适。所以加上Option之后会变成这样：

```rust
use std::rc::{Rc, Weak};

struct Node<T> {
  data: T,
  prev: Option<Weak<Node<T>>>,
  next: Option<Rc<Node<T>>>,
}
```

这样已经很接近了，我们来做一个小的测试吧：

```rust
let mut a = Node {
  data: 1,
  prev: None,
  next: None,
};
let b = Node {
  data: 2,
  prev: None,
  next: None,
};

a.next = Some(Rc::new(b));
a.next.as_ref().unwrap().next = None;
```

上面代码我们构建了两个节点a和b，我们先让b成a的后继节点：`a.next = Some(Rc::new(b));`，这一步还挺顺利，但是下一步我们让a后继的后继节点变为空时会报"cannot assign to data in an `Rc`"这样的错误。我们可以先把`as_ref().unwrap()`这些东东忽略，简单理解最后一行就是`a.next.next = None`这样。类似于后继的后继赋值这种操作在双向链表中还是挺常见的，我们还是回过头分析一下为什么会报错吧。

回忆我们刚开始学习rust时，会比较在意rust语言中默认变量是不可变的，因此我们需要显式在声明时加上`mut`关键字让该变量可变。而我们在定义节点前驱和后继节点时并没有指明其内部引用的节点是可变的，因此当我们尝试更改改节点应用的时候就会报错。我们再来看一个很简单的例子验证一下刚才的推断：

```rust
use std::rc::Rc;

let a: Rc<i32> = Rc::new(1);
*a = 5;
```

可以看到我们申明了一个类型为`Rc<i32>`的变量，然后尝试更改a的值，编译器报了同样的错误。这时我们最容易想到的改进方法是在Rc声明内部类型时指定可变性，将`Rc<i32>`改为`Rc<&mut i32>`，然后试着更改成如下形式：

```rust
use std::rc::Rc;

let mut b: i32 = 1;
let a: Rc<&mut i32> = Rc::new(&mut b);
**a = 5;
```

编译发现还是报了同样的错误。这条路看来是走不通了，我们试着将跳出目前的困境看看智能指针相关的类型还有什么，这时就会发现两个其他语言中找不到的类型：`Cell`和`RefCell`。看两者的说明都是用来改变内部可变性的，这正是我们想要的！接下来我们先使用`Cell`尝试让上面的例子编译通过吧：

```rust
use std::rc::Rc;
use std::cell::Cell;

let a: Rc<Cell<i32>> = Rc::new(Cell::new(1));
a.set(5);
```

编译之后发现终于被放行了：）通过`a.get()`查看发现值已经成功修改成了`5`，可喜可贺。

回过头来看看`Cell`和`RefCell`的区别：通过阅读文档我们发现`Cell`用于实现了`Copy trait`的类型（可以简单理解为值类型），而其他类型则使用`RefCell`。我们的Node结构体不打算实现`Copy trait`，所以我们通过`RefCell`修改节点的定义如下：

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

struct Node<T> {
  data: T,
  prev: Option<Weak<RefCell<Node<T>>>>,
  next: Option<Rc<RefCell<Node<T>>>>,
}
```

大功告成…实际上还没有，我们只是实现了节点的定义。不过我们已经渐入佳境了；）

也许你会觉得类似这样的`Option<Rc<RefCell<Node<T>>>>`定义看着头晕，也可以通过`type NodePtr<T> = Option<Rc<RefCell<Node<T>>>>;`这种方式把细节影藏起来。不过我们这里还是需要探究一下细节，所以还是保持原样吧。

## 链表定义

完成了节点结构的定义之后，我们也可以用类似的方式完成双向链表的结构。我们为双向链表结构中添加头指针和尾指针两个成员。我们总结这两个成员的一些特性吧：

- 它们一般会指向某个节点
- 当链表为空时它们为空
- 释放链表时会首先去除他们引用，然后触发链上节点释放的多米诺骨牌
- 当在链表首部或者尾部插入时会改变他们指向的节点

这样分析下来，头指针和尾指针和Node中的next成员很像，参照next的定义我们实现如下：

```rust
struct List<T> {
  first: Option<Rc<RefCell<Node<T>>>>,
  last: Option<Rc<RefCell<Node<T>>>>,
}
```

通过分析节点的定义，这里实现链表定义水到渠成，理解之后就很简单了是吧。

## 单元测试

限于篇幅我们在文中只实现双向链表的插入行为，其他行为大家可以参考我们接下来的分析来尝试实现，rustc是我们耐心且忠实的老朋友，多问问她一定没问题的：）

好了，接下来我们用单元测试描绘一下想要实现的行为吧。我们想要实现的是`append`行为，每次操作会追加到链表的末尾。因此我们会先创建一个链表，往链表中`append`几个节点，然后验证一下这些节点是否都成功加入到链表中，并且节点的顺序是否符合预期。

按照rust单元测试惯例，我们在实现上面逻辑的文件底部添加单元测试的定义：

```rust
#[cfg(test)]
mod tests {
  use super::{List};
    
  #[test]
  fn test_list_append() {
    // 实现测试逻辑
  }
}
```

接下来我在`test_list_append`方法中添加测试逻辑。按照上面的分析我们需要先创建一个链表，我们当然可以直接地这样创建：

```rust
let mut a = List {
  first: None,
  last: None,
};
```

不过这样不利于后期的使用，大体说来有下面的问题：

- 每次都定义太繁琐
- 定义的细节暴露出来，看起来不直观，会造成“视觉污染”
- 如果构造逻辑后续发生变化（比如要增加一些额外的处理加工），我们每一处创建都要再加上类似的逻辑，不符合我们一贯偷懒的作风

所以我们决定我们需要为我们的List添加一个名为`new`的构造方法：

```rust
let mut l: List<i32> = List::new();
```

注意这我们为变量`l`的后面显式地指定了类型`List<i32>`，这是因为我们的链表是泛型的，而这里没法自动推导出类型所以就直接告诉编译器“我想要生成这个结果”。这句话还有另一种实现：

```rust
let mut l = List::<i32>::new();
```

这里是就相当于告诉编译器“我知道节点的类型是什么”。也许有人觉得`List::<i32>::new()`这种表达方式很奇怪，其实这和`<List<i32>>::new()`是等价的（熟悉C++的同学对这种表达方式一定不陌生），是rust中一种减少泛型`<>`嵌套的表达方式。

构造的话题我们就说到这里吧，上面几种表达方式大家可以按喜好来用，不过在实际生产开发中为了便于阅读最好保持风格的一惯性。

接下来我们为我们的链表中添加几个元素：

```rust
for i in 1..6 {
  l.append(i);
}
```

我们依次给链表中添加了1~5几个数字，接下来我们就验证一下添加后的结果是否正确。为此我们需要申明一个指针的引用：

```rust
let mut cur = l.first.clone();
```

我们看到这里我们定义了一个`cur`的变量，它最开始指向链表的头指针的位置。那这里为什么需要`clone`呢？`clone`的过程到底做了什么呢？

我们先看看后一个问题吧。`clone`方法实际上是对`Clone` trait的实现。我们知道这里的first类型是`Option<Rc<RefCell<Node<T>>>>`，因此这里调用`clone`会首先调用Option的`clone`方法，我们看看Option中`clone`方法的实现吧：

```rust
#[inline]
fn clone(&self) -> Self {
  match self {
    Some(x) => Some(x.clone()),
    None => None,
  }
}
```

我们看到这里实际上只是调用Option内部类型的clone方法，由于这里的内部类型是`Rc<RefCell<Node<T>>>`，因此我们在往下看下Rc中的`clone`方法实现：

```rust
#[inline]
fn clone(&self) -> Rc<T> {
  self.inc_strong();
  Self::from_inner(self.ptr)
}
```

通过这两个方法名，我们就能大概猜测到这里增加了引用计数，并且返回了一个自己的指针拷贝，而指针实际上就是一个`RefCell<Node<T>>`。因此调用`clone`实际上是以增加引用计数的方式返回了一个相同的指针，而这正是我们想要的获取指针的方式。

再来看看为什么要调用`clone`。如果找到Option的定义，我们会发现Option是实现了`Copy` trait的，我们知道实现了`Copy`之后可以看做是“值类型”，因此当我们像`let mut cur = l.first;`这样赋值时不会触发`move`操作，而是会进行拷贝操作。

那么如何定义拷贝操作呢？找到`Copy`的定义，我们会发现它实际上是继承了`Clone` trait：

```rust
#[lang = "copy"]
pub trait Copy: Clone {
  // Empty.
}
```

所以`let mut cur = l.first;`这句在句末加上`clone`并不是必须的，因为默认的拷贝操作也会调用`clone`方法的。不过为了让这里的操作用意看起来更明显一些，我们还是在这里显式地加上了`clone`。

好了，接下来我们通过循环来依次比对值。我将剩下的整个逻辑都贴出来吧：

```rust
let mut cur_value = 1;
while let Some(node) = cur {
  assert_eq!(cur_value, node.as_ref().borrow().data);

  cur_value += 1;
  cur = node.as_ref().borrow().next.clone();
}
```

可以看到我们在这里定义了一个当前比对的值，然后通过循环判断是否符合预期。这一段整体都好理解，我们比较在意的是`as_ref().borrow()`这样的表达式，我们就重点看一下这个好了。

我们通过`let Some`表达式来定义node变量，他的类型则是`Rc<RefCell<Node<T>>>`，所以这里的`as_ref`是对Rc的操作，我们再看看Rc关于`as_ref`的实现：

```rust
impl<T: ?Sized> AsRef<T> for Rc<T> {
  fn as_ref(&self) -> &T {
    &**self
  }
}
```

可以看到as_ref实际上是将内部的类型`T`转换为`&T`，因此对node执行`as_ref`操作后的类型是`&RefCell<Node<T>>`。

接下来的再看看`borrow`方法吧，borrow方法在RefCell中实现，rust中的定义如下：

```rust
pub fn borrow(&self) -> Ref<'_, T> {
  self.try_borrow().expect("already mutably borrowed")
}
```

可以看到最终返回的是一个`Ref<'_, T>`类型，第一个参数是生命周期参数，通过`_`忽略后实际的类型就是`Ref<Node<T>>`。

我们大概明白了`as_ref`和`borrow`的作用之后，现在回过头来想想为什么要这样做。上面我通过循环比对的时候有在第3行和第6行两个地方调用了这两个方法，而这两个地方实际上都是想要访问`Node`结构体中的成员（分别是`data`和`next`）。可是我们知道node类型是`Rc<RefCell<Node<T>>>`，为此我们需要像剥洋葱一样一层层地退去外面包裹的类型，因此才会通过`as_ref`和`borrow`方法将Rc和RefCell褪去。

可是等等，`borrow`之后的类型是`Ref<Node<T>>`，为什么就可以直接用`.`操作符来访问`Node`的成员了呢？

我们通过`Ref`这个类型来发现线索吧，找到`Ref`的定义我们会看到它实现了`Deref` trait：

```rust
impl<T: ?Sized> Deref for Ref<'_, T> {
  type Target = T;
  
  #[inline]
  fn deref(&self) -> &T {
    self.value
  }
}
```

`Deref`是一个很有意思的trait，如果类型`Y`实现了这个trait，那对于类型`Y`的一个变量y，当我们执行`*y`解引用操作时，rust事实上在底层运行了`*(y.deref())`操作。事实上所有的智能指针也是实现了`Deref` trait之后才会变得能够[当做常规引用处理](https://kaisery.github.io/trpl-zh-cn/ch15-02-deref.html)。

这样最后一个谜题也解开了，我们的单元测试分析也终于到最后啦。不过我们这里只是添加了一个简单的正常测试，在实际开发中我们还需要添加边界值、异常测试等保障我们逻辑的健壮性。

虽然我们还没有实现具体的逻辑，不过人性化的rust也帮我们准备了`cargo test --no-run`这样的只保障编译通过的测试方式。不过遗憾的是即便如此也会报错，因为我们还没有定义测试中用到的方法。事不宜迟我们立马补上：

```rust
impl<T> List<T> {
  pub fn new() -> Self {
    List {
      first: None,
      last: None,
    }
  }

  pub fn append(&mut self, data: T) {
    // todo: complete me
  }
}
```

这样`cargo test --no-run`终于通过啦。

## 节点实现

这里我们还是参考[playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2015)的实现在Node中添加一个构造方法和一个append方法，这里的append会递归查找自己的后继节点，直到找到最后一个节点。整个逻辑比较简单我们就直接贴出来吧：

```rust
impl<T> Node<T> {
  pub fn new(data: T) -> Self {
    Node {
      data,
      prev: None,
      next: None,
    }
  }

  pub fn append(node: &mut Rc<RefCell<Node<T>>>, data: T) {
    let is_last = node.borrow_mut().next.is_none();
    if is_last {
      let mut new_node = Self::new(data);
      new_node.prev = Some(Rc::downgrade(node));
      node.borrow_mut().next = Some(Rc::new(RefCell::new(new_node)));
    } else {
      if let Some(ref mut next) = node.borrow_mut().next {
        Self::append(next, data);
      }
    }
  }
}
```

我们先看看比较简单的new方法，该方法中的data初始化和其他两个变量看起来不太一样，这是因为在rust的结构体初始化时，如果成员变量名和用来初始化的名称相同就可以用一个变量代替。

然后再看重头戏append吧，我们会发现append的第一个参数类型是`&mut Rc<RefCell<Node<T>>>`，需要声明为可变引用`&mut`的原因是我们可能会在内部改变node成员变量的值。我们会看到和`Node`以及`List`中的引用计数指针比起来缺少了Option，这是因为我们希望传入的都是非空的值，为`None`的情况就自然地被我们过滤掉了。

实现方法的逻辑可以分为两个部分：如果已经到最后一个节点（尾节点），我们就创建一个节点作为新的尾节点；否则我们就找到该节点的下一个节点来递归调用该方法。

实现逻辑比较简单我们就不再做进一步说明了。我们接下来看一下几处不好理解的语法点吧。

我们会发现代码中多处用到了`borrow_mut`方法，它和我们前面说明的`borrow`类似，用来返回内部类型的可变引用，具体实现是这样的：

```rust
pub fn borrow_mut(&self) -> RefMut<'_, T> {
  self.try_borrow_mut().expect("already borrowed")
}
```

可以看到返回的类型是`RefMut`，该类型和`Ref`也是相似的类型，而且它也同样实现了`Deref` trait，这意味着我们可以当做常规引用一样访问`Node`结构体的内部成员了。

我们在定义`Node`的时候将前驱指针的类型设置成弱引用指针`Weak`的形式，因此在上面代码的14行初始化的时候，我们需要通过`Rc::downgrade`方法把`Rc<T>`转换为`Weak<T>`。

15行中为类型为`Option<Rc<RefCell<Node<T>>>>`的后继类型指针next设置新值的时候，我们使用了`Some(Rc::new(RefCell::new(new_node)));`这样的表达式。最外层的`Some`是Option枚举类型的非空成员，Some内部我们连续调用多个类型的构造器`new`，可以看到`new`构造器也是rust语言中约定俗成的构造器方法名。

接下来我们将视线下移到17、18行，我们通过`node.borrow_mut().next`得到的是`Option<Rc<RefCell<Node<T>>>>`类型，不过我们在调用的时候期望的类型是`&mut Rc<RefCell<Node<T>>>`，我们要怎样剥掉外面的Option类型并且变成可变引用呢？

我们第一反应可能会是通过Option的常用方法`unwrap`，不过这个方法如果为空的时候会抛出panic，所以除非这就是你想要的错误处理方式，或者你可以百分之百肯定这里不为空的时候再考虑使用它。

考虑之后我们还是通过模式匹配`let Some`的方式，不过我们不加任何修饰得到的变量是`Rc<RefCell<Node<T>>>`，怎样才能得到`&mut Rc<RefCell<Node<T>>>`呢？在匹配表达式中加上`mut`关键字行得通，不过要加上取地址附`&`编译器就会找我们的麻烦了。

这个时候我们需要借助一个关键：`ref`。它和`&`在很多时候可以等价使用的：

```rust
let ref a = 1;
let a = &1;
```

上面两个变量都是`&i32`类型。不过也有互相不能替换的时候，比如上面我们遇到的绑定（模式匹配）情况，就只能借助`ref`；而如果需要对一个表达式取地址，那就只能用`&`而不能用`ref`。如果要加深理解的话，我觉得可以再换个角度思考一下使用场景：左值（left value）情况下只能用`&`；右值情况下都可以用`ref`，并且有部分情况（模式匹配）是只能用`ref`的。

节点实现的说明就到这里啦，总体来说还是挺顺利的。

## 链表实现

终于到最后了，还记得我们之前我们还没有实现的`append`方法吗？我接下来就来搞定它。

其实我们已经在节点里已经有一个`append`方法了，它会通过递归找到一个给定节点的后续节点，直到最后一个节点的时候新增一个节点作为新的末尾节点。这个方法已经涵盖了追加值的核心逻辑，不过在节点中的方法只负责与节点相关的添加操作。而当我们通过链表新增一个节点的时候，链表层面也是有变化的：追加之后原来的尾结点已经不再是尾结点了，因此我们要将`last`成员指向新的尾结点。

此外，我们还需要考虑一下整个链表为空时的新增情况，这时候我们甚至都不用调用节点的append方法了。

好了，我们贴上代码吧：

```rust
if let Some(ref mut next) = self.first {
  Node::append(next, data);
  let v = self.last.as_ref().unwrap().borrow().next
    .as_ref().unwrap().clone();
  self.last = Some(v);
} else {
  let f = Rc::new(RefCell::new(Node::new(data)));
  self.first = Some(f.clone());
  self.last = Some(f);
}
```

经过我们之前的披荆斩棘，这段代码对我们来说就是一马平川了。注意我们在第3行和第4行都用到了`unwrap`，不过当第二句插入成功后，这两处都是确定不会为空的情况。

这下我们去掉`--no-run`的参数用`cargo test`运行一下单元测试，大功告成🎉

不过目前为止的实现还有很多可以改进的地方，比如在节点的`append`方法里，递归调用的情况我们只处理了非空的情况，如果能将空值通过Result方式抛出也是极好的。再在比如链表的`append`方法里，既然我们定义了尾指针`last`，那也不需要从头指针`first`开始去找它了。更重要的是，我们只做了`append`，还有插入、删除、检索等等功能没有实现，这些功能都后续交给大家来继续探索啦。

------

rust语言学习曲线很陡峭，不过除了类型系统、生命周期这些大块的理论知识点（学习理论强烈推荐张汉东老师的《Rust编程之道》，深入浅出娓娓道来）之外，主要还是一些细碎的知识点，我们建立起Rust整体语法认知框架之后，通过不断练习掌握零散的知识点，再不断地建立起与已有知识的关联，相信不久的将来一定会开花结果。

我目前也在探索的途中，有时也会着急也会因进步太小而沮丧。不过最近很喜欢胡适先生的一句话，贴在这里作为结束语与大家一起共勉：	

> 怕什么真理无穷，进一寸有一寸的欢喜