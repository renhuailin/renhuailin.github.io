---
title: 在Rust单元测试中调用异步函数
date: 2024-01-24T12:56:13+08:00
summary: 在单元测试中直接调用异步函数是不行的，那要怎么调用呢？
slug: call_async_functions_in_rust_unit_test
categories:
  - Rust
tags:
  - Rust
---

最近写的一个网站，要抓取一些内容，为了保证抓取回来的内容能正确解析，我要先在单元测试中测试我的解析代码。抓取的代码如下。

```rust
async fn request_test() {
    let res = reqwest::get("http://httpbin.org/get").await.unwrap();
    println!("Status: {}", res.status());
    println!("Headers:\n{:#?}", res.headers());

    let body = res.text().await.unwrap();
    println!("Body:\n{}", body);
}
```

明眼人看得出来，这就是reqwest的示例代码。

下面是我的测试代码：
```rust
	#[test]
    async fn test_async() {
        async fn request_test() {
            let res = reqwest::get("http://httpbin.org/get").await.unwrap();
            println!("Status: {}", res.status());
            println!("Headers:\n{:#?}", res.headers());

            let body = res.text().await.unwrap();
            println!("Body:\n{}", body);
        }
		//尝试调用这个异步函数
        request_test().await();
    }

```
因为`request_test`是异步函数，所以要调用`await()`函数，这个跟Javascript是一样的。在同步函数里是不能调用`await()`的，这一点跟Javascript也一样。所以我给`test_async`函数加上了`async`，尝试运行这个测试。


结果无法编译，报错了：
```
error: async functions cannot be used for tests
```
这个报错的意思是，在异步函数不能用来做测试。也就是`#[test]`宏，不能修饰异步函数。

既然异步函数不能直接做为单元测试函数，那只能用同步函数了，去掉`test_async`前面的`async`,但是怎么调用异步函数`request_test`呢？

原来是用`block_on`这个函数。
```rust
use futures::executor::block_on;

#[test]
fn test_async() {
    async fn request_test() {
        let res = reqwest::get("http://httpbin.org/get").await.unwrap();
        println!("Status: {}", res.status());
        println!("Headers:\n{:#?}", res.headers());

        let body = res.text().await.unwrap();
        println!("Body:\n{}", body);
    }

    block_on(request_test());
}
```
但是这样调用还是不行。
```
thread 'test::test_async' panicked at /Users/harley/.cargo/registry/src/index.crates.io-6f17d22bba15001f/tokio-1.35.1/src/net/tcp/stream.rs:160:18:
there is no reactor running, must be called from the context of a Tokio 1.x runtime
```
要运行这个测试需要 tokio context， rust标准库里的`block_on`是没有tokio context的。

我想主要原因是rust只定义了支持异步的一些类型，并没有在标准库里实现异步支持。异步的支持交给了社区来实现，这是挺怪的。我个人认为Rust应该在标准库里实现异步支持，不过这几年Rust管理层变动较大，可能也没人有精力来解决这个问题。

Rust社区有两非常出名的 `async` 运行时：[`tokio`](https://github.com/tokio-rs/tokio) 和 [`async-std`](https://github.com/async-rs/async-std)。

在这里我就使用`tokio`这个异步库，把它加到项目的依赖里。

要调用request_test()则要使用`tokio`所提供的`block_on`函数。
```rust
#[test]
fn hello_world() {
    async fn request_test() {
        let res = reqwest::get("http://httpbin.org/get").await.unwrap();
        println!("Status: {}", res.status());
        println!("Headers:\n{:#?}", res.headers());

        let body = res.text().await.unwrap();
        println!("Body:\n{}", body);
    }

    let runtime = Builder::new_current_thread().enable_all().build().unwrap();
    runtime.block_on(request_test())
}
```
这样就能成功地调用异步函数了，不过也能看到在异步支持这方面rust还需要继续加强啊。












