---
title: "在Rust代码中使用保留字做为标识符"
date: 2024-01-23T11:41:13+08:00
summary: "今天在看Rocket的例子时，看到一个struct里有这样一段代码：r#type: &'r str, 我不懂r# 的意思，原来是叫原生标识符的东西。"
slug: rust_use_reserved_keywords_as_identifiers
categories:
- Rust
tags:
- Rust
---

今天在看Rocket的例子时，看到一个struct里有这样一段代码:

```rust
use rocket::form::Form;

#[derive(FromForm)]
struct Task<'r> {
    complete: bool,
    r#type: &'r str,
}
```

其中的`'r`我知道是lifetime，但是`r#type`是什么意思？

我让ChatGPT帮我解释这段代码，它回答这是Rust语言中使用关键字做为标识符的方法。

我又查询了《Rust语言圣经》，在Rust里叫[原生标识符 Raw identifiers](https://course.rs/appendix/keywords.html?highlight=r%23#%E5%8E%9F%E7%94%9F%E6%A0%87%E8%AF%86%E7%AC%A6)。

在编程语言中，通常是不能使用关键字keyword做为标识符的，比如不能使用`int`做为函数名。
这些关键字在编程语言中是有特殊意义的，编译器也是要做特殊处理的，不能随便用。但是有些情况使用关键字是最合理的，比如上面的Task这个结构体，任务类型使用`type`做为字段名是最合理的，换成其它的都不合适。 这时就需要在`type`前面加上`r#`,告诉编译器我就是要使用`type`做为标识符。

