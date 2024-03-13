---
title: Rust中derive宏的作用及常用trait
date: 2024-02-20T22:42:15+08:00
summary: 在Rust代码中经常可以看到在struct的上面会有#[derive(Clone, Debug)]这样的代码，本文会解释这段代码的作用及与derive宏配合使用的常见trait。
slug: rust-dervice-trait
categories:
  - Rust
tags:
  - Rust
---
在Rust代码是经常可以看到在struct的上面，会有一行类似`#[derive(Clone, Debug)]`这样的代码。`dervice`是Rust的内置宏，可以自动为struct或是enum实现某些的trait.

##  Debug trait
在下面的代码中，Book struct 通过derive宏自动实现了Debug、Clone和PartialEq这三个trait。
```rust
/// Defines a Column for an Entity
#[derive(Debug, Clone, PartialEq)]
pub struct Book {
    pub title: String,
    pub isbn : String,
    pub price: i32,
    pub author: String,
}
```
所谓自动实现，就是不用您自己写实现代码。
我们以Debug这个trait为例，它包含一个方法`fmt`，是使用指定的Formatter来格式struct或enum中的值。
```rust
fn fmt(&self, f: &mut Formatter<'_>) -> Result;
```
当使用derive自动`Debug`实现时，Rust的编译器会自动生成实现`Debug`trait的代码，可以减少代码编写的工作量。

与`Debug`trait相似，Rust还为标准库中常用的trait定义了相关的宏，这些宏都可以与`dervice`配合使用，简化代码，下面一一介绍这几个常用的trait。


##  Clone trait
看名字大家也就可以猜到，这个trait是用来复制实例的。在Rust中什么情况下需要clone一个实例呢？为什么默认为实例实现这个trait呢？

1.  **显式控制** ：Rust强调显式性和安全性。所以默认并没有为所有的类型实现这个trati，它确保开发人员知道并明确允许克隆行为。这有助于防止由于不加选择地克隆而导致的意外性能问题或意外行为。
    
2. **避免借用**：在Rust中，通过引用（borrowing）传递值是避免不必要复制并维护所有权语义的默认方式，然而，在需要创建具有自己所有权的新实例的情况下，`Clone`提供了一种无需借用的方法。这种情况在新手刚使用Rust的时候经常会碰到，常会碰到编译器提示，所有权已经在某处move了，提示需要clone这个实例。
    
3. **深拷贝**：如果定义的结构体中不仅包含基本类型，还包含其它结构体，则在Clone的时候 ，通常希望创建深拷贝，这意味着不仅复制顶层结构，还要复制所有嵌套数据。`Clone` trait提供了一种类型定义如何克隆的方法，允许它们实现自定义的深度复制行为。
    
4. **定制克隆**：有时候我们Clone的时候，并不希望Clone原实例的所有的值，可能只希望部分数值被Clone到新实例（具体场景当用到的时候自然就会知道）。
    
上面的4种场景，除了场景4其它都可以直接用devive宏来实现。


## PartialEq  trait
在Rust里 `PartialEq`和`Eq`这两个trait也挺让人迷惑的。

`PartialEq`，故名思义，是部分相等。这个trait有两个方法：
```rust
pub trait PartialEq<Rhs = Self>where
    Rhs: ?Sized,{
    // Required method
    fn eq(&self, other: &Rhs) -> bool;

    // Provided method
    fn ne(&self, other: &Rhs) -> bool { ... }
}
```
只需要实现`eq`这个方法即可。那么Partial体现在哪儿呢？比如有个Book结构体，包含`isbn`和`format`两个字段，只要`isbn`相等，就可以认为两个Book是相等的，从这个意义上看，只有部分字段相等就可以认为相等，所以称`Partial`。

```rust
enum BookFormat {
    Paperback,
    Hardback,
    Ebook,
}

struct Book {
    isbn: i32,
    format: BookFormat,
}

impl PartialEq for Book {
    fn eq(&self, other: &Self) -> bool {
        self.isbn == other.isbn
    }
}

let b1 = Book { isbn: 3, format: BookFormat::Paperback };
let b2 = Book { isbn: 3, format: BookFormat::Ebook };
let b3 = Book { isbn: 10, format: BookFormat::Paperback };
```

另外，跟Eq trait相比，`PartialEq`不满足自反性。
所谓自反性，就是一个类型的所有实例应该跟它自己相等，如果不是这个类型就不满足Eq,它就是`PartialEq`。这样说比较抽象，举个例子来说明。
```rust
fn main() {
    let f1 = 3.14;
    is_eq(f1);
    is_partial_eq(f1)
}

fn is_eq<T: Eq>(f: T) {}
fn is_partial_eq<T: PartialEq>(f: T) {}
```
运行上面的代码，会发现float并没有实现`Eq`，这很奇怪吧？
```
 is_eq(f1);
       ----- ^^ the trait `Eq` is not implemented for `{float}`
```

这是因为浮点数有一个特殊的值 `NaN`，它是无法进行相等性比较的，也就是`NaN == NaN`是不成立的。如果你的struct也有类似的情况，那就不能实现`Eq` trait。

这两个trait都可以用`derive`宏来自动实现。当用`derive`来实现时，如果要实现`Eq` trait，那么所有的字段都必须实现Eq，如果包含浮点数这样没有实现`Eq`的字段，那么是无法实现`Eq`的。比如下面的代码是无法编译的：
```
#[derive(Debug, PartialEq, Eq)]
struct Book {
	isbn: String,
	price: f32,
}
```
编译器会建议你把price改成i32这种实现了`Eq`的类型。

## PartialOrd, Ord
`PartialOrd`和`Ord`这对Trait的应用场景跟`PartialEq`和`Eq`非常相似。
`PartialOrd`用于类型只能部分进行比较的场景，`Ord`则要求类型所有的部分都能进行比较。
比如上面例子中的浮点类型中的`NaN`,是不能比较的，所以包含浮点数的类型，就不能实现`Ord`，只能实现`PartialOrd`。

```rust
pub trait PartialOrd<Rhs = Self>: PartialEq<Rhs>
where
    Rhs: ?Sized,
{
    // Required method
    fn partial_cmp(&self, other: &Rhs) -> Option<Ordering>;

    // Provided methods
    fn lt(&self, other: &Rhs) -> bool { ... }
    fn le(&self, other: &Rhs) -> bool { ... }
    fn gt(&self, other: &Rhs) -> bool { ... }
    fn ge(&self, other: &Rhs) -> bool { ... }
}
```
 `lt`、`le`、 `gt` 和 `ge`可能分别通过 `<`、`<=`、 `>`,和 `>=` 这些操作符来调用。可见Rust是通过trait来支持操作符重载的。

总结一下，上述的traits在rust里被称为`Derivable Traits`，中文叫`可派生的 trait`。这些trait是标准库中定义的，可以通过derive在类型上实现。这些trait具有默认行为，因此可以通过简单的derive语法来自动生成对应的实现代码。Derivable Traits允许程序员轻松地为他们的类型自动生成一些常见trait的实现代码，提高了开发效率并确保了一致性。