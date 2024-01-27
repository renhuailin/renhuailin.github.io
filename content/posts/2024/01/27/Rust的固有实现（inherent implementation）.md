---
title: Rust的固有实现（inherent implementation）
date: 2024-01-27T20:44:34+08:00
summary: 在Rust语言规范中是这样描述固有实现的，固有实现定义为 `impl` 关键字、泛型类型声明、名义类型的路径、where 子句的序列和一组带括号的可关联项。
slug: rust_inherent_implementation
categories:
  - Rust
tags:
  - Rust
---

在Rust语言规范中是这样描述固有实现的：
```
An inherent implementation is defined as the sequence of the impl keyword, generic type declarations, a path to a nominal type, a where clause, and a bracketed set of associable items.
```
固有实现定义为 `impl` 关键字、泛型类型声明、名义类型的路径、where 子句的序列和一组带括号的可关联项。      
Rust 的固有实现（Inherent Implementation）可以通过以下方式来定义：

```rust
impl <P1..=Pn> Type<T1..=Tn> where clause {
    // associable items
}
```

用代码来解释一下。
```rust
pub struct Color(pub u8, pub u8, pub u8);

impl Color {
    pub fn white() -> Color {
        Color(255, 255, 255)
    }
}
```

函数`white`就是一个固有实现。
固有实现有一个特点，那就是在一个crate里的所有固有实现，不论你写在哪个module里，都能通过类型的路径来调用它。

```rust
pub mod color {
    pub struct Color(pub u8, pub u8, pub u8);

    impl Color {
        pub const WHITE: Color = Color(255, 255, 255);
    }
}

mod values {
    use super::color::Color;
    impl Color {
        pub fn red() -> Color {
            Color(255, 0, 0)
        }
    }
}

pub use self::color::Color;
fn main() {
    // Actual path to the implementing type and impl in the same module.
    color::Color::WHITE;

    // Impl blocks in different modules are still accessed through a path to the type.
    color::Color::red();

    // Re-exported paths to the implementing type also work.
    Color::red();

    // Does not work, because use in `values` is not pub.
    // values::Color::red();
}
```
上面的例子里，在`color`模块和`values`模块里分别为`Color`编写了固有实现的代码，定义了一个常量和一个方法，在main函数里都可以通过`Color`引用路径来调用。

上面的代码虽然是在一个文件里，你完全可以把这些实现分散在crate的多个文件里，我做过测试了，效果是一样的，也就是Rust编译器会把分散在不同位置的固有实现合在一起。

需要注意的是，只有固有实现才能这样，如果是特征实现(Trait implementation)就不行了，不能只通过类型的路径调用实现，必须引入相关的trait才可以。