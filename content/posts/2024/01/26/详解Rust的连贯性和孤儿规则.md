---
title: 详解Rust的连贯性和孤儿规则
date: 2024-01-26T16:02:54+08:00
summary: Rust的连贯性和孤儿规则很让人迷惑，本文结合实例详细解释了什么是连贯性和孤儿规则。
slug: explaining_rust_coherence_and_orphan_rules_in_detail
categories:
  - Rust
tags:
  - Rust
---

最近学习Rust时候看到两个术语：连贯性(Coherence)和孤儿规则(Orphan rules)，书上解释的不是很清楚，又没有给出具体的代码示例，让人很难理解。我在网上搜了好久，最后又查了Rust语言规范，算是搞明白了这两个概念，在这里尝试解释一下，如果有理解不对的地方，请各位同学留言指正。

连贯性(Coherence)和孤儿规则(Orphan rules)两者之间是有关联的。

我们先解释一下连贯性，**连贯性**指的是Trait实现的连贯性。    
rust语言规范是这样写的：“A trait implementation is considered incoherent if either the orphan rules check fails or there are overlapping implementation instances.”
如果孤儿规则检查失败或存在重叠的实现实例，则认为trait的实现是不连贯的。      

存在重叠的实现实例，这个很好理解，所以理解了`孤儿规则`，也就理解了`连贯性`。

下面是`孤儿规则`在rust语言规范中的定义：
**孤儿规则**
给定一个 `impl<P1..=Pn> Trait<T1..=Tn> for T0` ，仅当以下至少一项为真时，这个 `impl` 才有效：
- `Trait` 是一个 [本地 trait](https://github.com/rust-lang/reference/blob/master/src/glossary.md#local-trait)
- 满足以下所有条件：
    - `T0..=Tn`中**至少有一个**类型是[本地类型](https://github.com/rust-lang/reference/blob/master/src/glossary.md#local-trait) 。这里我们把第一个本地类型定义为`Ti`。
    - No [uncovered type](https://github.com/rust-lang/reference/blob/master/src/glossary.md#uncovered-type) parameters `P1..=Pn` may appear in `T0..Ti` (excluding `Ti`)。        `P1..=Pn`中的[未覆盖的类型](https://github.com/rust-lang/reference/blob/master/src/glossary.md#uncovered-type)参数不能出现在 `T0..Ti` （不包括 Ti ）中； 因为这一条比较难翻译，所以附上了英文。


我来解释一下。    
一、如果trait是本地trait，那就直接满足孤儿规则了。本地trait的意思是这个trait是在当前的crate里定义的。   
二、如果trait是不是本地trait。那就要同时满足两个条件：
1. 类型`T0..=Tn`要至少有一个类型是本地类型。当然，一个impl可能会有多个本地类型，我们第一个本地类型定义为`Ti`。
2. `P1..=Pn`中的[未覆盖的类型](https://github.com/rust-lang/reference/blob/master/src/glossary.md#uncovered-type)参数不能出现在 `T0..Ti` （不包括 Ti ）中。
	这一条是比较难理解的，如果理解了这一条，我们就理解了孤儿规则了。    
	什么是`未覆盖的类型`呢？    
	规范里的定义是：`A type which does not appear as an argument to another type`.  一个类型如果它不是做为其它类型的参数而出现，那它就是`未覆盖的类型`。      
	举个例子，类型`T`就是`未覆盖的类型`，但是`Vec<T>`中的`T`就不是`未覆盖的类型`了，就是覆盖了的类型，因为它做为`Vec`的参数出现了，不是单独出现的。
	
	我们再来看看规范中的说明：`P1..=Pn`中的`未覆盖的类型`参数不能出现在 `T0..Ti（不包括 Ti ）` 中。需要注意是 `T0..Ti`，规范中可没要求是`T0..=Tn`，这一点要注意。
	
但是这样我们还是难以理解，下面我用代码来解释一下。

请试着编译下面的代码：
```rust
struct Entry<K, V> {
    key: K,
    value: V
}

impl<K: PartialEq, V> PartialEq<Entry<K, V>> for K {
    fn eq(&self, other: &Entry<K, V>) -> bool {
        self.eq(&other.key)
    }
}
```

如果你在rust playground里编译，会报下面的错误：
```
   Compiling playground v0.0.1 (/playground)
error[E0210]: type parameter `K` must be covered by another type when it appears before the first local type (`Entry<K, V>`)
 --> src/lib.rs:6:6
  |
6 | impl<K: PartialEq, V> PartialEq<Entry<K, V>> for K {
  |      ^ type parameter `K` must be covered by another type when it appears before the first local type (`Entry<K, V>`)
  |
  = note: implementing a foreign trait is only possible if at least one of the types for which it is implemented is local, and no uncovered type parameters appear before that first local type
  = note: in this case, 'before' refers to the following order: `impl<..> ForeignTrait<T1, ..., Tn> for T0`, where `T0` is the first and `Tn` is the last

For more information about this error, try `rustc --explain E0210`.
error: could not compile `playground` (lib) due to previous error
```

请看编译器给出的信息，其中:
```
= note: implementing a foreign trait is only possible if at least one of the types for which it is implemented is local, and no uncovered type parameters appear before that first local type
```
这不就是孤儿规则嘛。

我们来详细分析一下。               
实现一个trait的语法是这样的`impl<P1..=Pn> Trait<T1..=Tn> for T0` 。           
在上面的例子里，实现的代码是这样的`impl<K: PartialEq, V> PartialEq<Entry<K, V>> for K {`           
我们可以得出：   
`P1 = K,P2=V`   
trait是PartialEq   
`T0 = K`   
`T1  = Entry<K, V>`   
Ti是第一个本地类型，所以Ti=T1。

规范是这样规定的： `P1..=Pn`中的`未覆盖的类型`参数不能出现在 `T0..Ti（不包括 Ti ）`      
在本例中：    
`P1..=Pn` 就是 [K,V]。
`T0..Ti` 就是[K,Entry<K, V>]    
注意`P1`也就是`K`，出现在了`T0..Ti`中，而`K`又是一个`未覆盖的类型`（uncovered type），所以违反了孤儿规则，代码编译报错。

通过这个例子，我想大家就彻底搞明白了`孤儿规则`，以后碰到也不会觉得奇怪了。当然，不明白也没问题，如果你违反了规则，编译器会报错的。