---
title: Rust必须引入trait才能编译的原因
date: 2024-01-25T12:13:31+08:00
summary: 为什么在rust里，有时必须引入相关的trait代码才能编译，具体原因请看正文。
slug: rust_have_to_use_traits_in_order_to_compile?
categories:
  - Rust
tags:
  - Rust
---
今天在写代码时发现，我必须引入一个trait代码才能编译。
```rust
buf.write_all(b"hello")?; 
```
编译器在`write_all`这儿报错，提示我可能要引入`std::io::Write`。我引入`use std::io::Write`后代码编译成功了

```rust
use std::io::Write

buf.write_all(b"hello")?; 
```
做为一个Java和Python程序员我懵了。为什么呢？

因为Rust允许在任何时候为任何类型实现任何Trait。

这会导致什么呢？比如有两个trait，都有叫`fn log(&self);`的方法，在一个crate里完全有可能在同一个类型上实现这两个trait，请看下面的代码。

```rust
mod module1 {
    pub trait Logger {
        fn log(&self);
    }
}

mod module2 {
    pub trait MyLogger {
        fn log(&self);
    }

    pub struct Console;
    impl MyLogger for Console {
        fn log(&self) {
            println!("MyLogger:: hello!")
        }
    }
}

mod module3 {

    use super::module1::Logger;
    use super::module2::Console;

    impl Logger for Console {
        fn log(&self) {
            println!("Logger:: hello!")
        }
    }

    #[test]
    fn test() {
        let console = Console {};
        console.log();
    }
}

mod module4 {
    use super::module2::Console;

    #[test]
    fn test() {
        let console = Console {};
        console.log();
    }
}
```

在上面的代码里，我在module1和module2里分别定义了两个trait：`Logger`和`MyLogger`,这两个trait里都包含`fn log(&self);`方法的定义。在`module4`里我尝试调用`log()`方法，编译器会报错。

因为`Console`在module2和module3里分别实现了MyLogger和Logger这两个trait，而这两个trait都包含`fn log(&self);`方法，编译器不知道你用的是哪个，所以你必须引入具体的trait。

```rust
mod module4 {
	use super::module1::Logger;
    use super::module2::Console;
    #[test]
    fn test() {
        let console = Console {};
        console.log(); // 输出："Logger:: hello!"
    }
}
```

在引入了`module1::Logger`后，代码就可以编译运行了，输出结果是`Logger:: hello!`，这部分代码实现是在`module3`里。

接下来我试着引入`module2`看看。

```rust
mod module4 {
	use super::module2::{Console, MyLogger};
    #[test]
    fn test() {
        let console = Console {};
        console.log(); // 输出："MyLogger:: hello!"
    }
}
```

输出结果变成了`MyLogger:: hello!`,可见引入不同的trait，输出的结果也不同。

为什么会如此呢？还是因为**Rust允许在任何时候为任何类型实现任何Trait**。

在Rust里，一个Type(如struct)的实现代码可能分布在不同的地方。这跟Java、python是不一样的，Java、python里类型的实现代码都在同一个地方，是不可能出现在一个类里面包含两个完全一样的方法的。而在Rust里就完全可能，因此为了明确方法的出错，必须引入相关的trait。


