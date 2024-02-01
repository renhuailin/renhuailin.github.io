---
title: 跨域请求CORS和HTTP Option实现
date: 2024-02-01T17:01:20+08:00
summary: 在Web开发中，有时会碰到跨域请求，如何正确地处理跨域请求？我们如何控制某些网站才能请求你网站的内容？请看正文。
slug: cors
categories:
  - Rust
tags:
  - Rust
  - Web
---
写网站经常需要调用其它网站上的资源，写App时要调用后台的服务，这些情况下都涉及到跨域请求。

我最近在调试时就碰到了跨域的问题：
```
Access to fetch at 'http://127.0.0.1:8000/new_task_from_json' from origin 'https://renhl.com' has been blocked by CORS policy: Response to preflight request doesn't pass access control check: It does not have HTTP ok status.
```

以前也碰到过跨域的问题，找个插件就解决了。但是每次都是只知道解决方法，但没有搞明白到底为什么，今天我要搞清楚它。

先介绍一个概念CORS，([CORS](https://developer.mozilla.org/en-US/docs/Glossary/CORS)) is an [HTTP](https://developer.mozilla.org/en-US/docs/Glossary/HTTP)-header based mechanism that allows a server to indicate any [origins](https://developer.mozilla.org/en-US/docs/Glossary/Origin) (domain, scheme, or port) other than its own from which a browser should permit loading resources
跨域资源共享 （CORS） 是一种基于 HTTP  Header的机制，它允许服务器指示  除其自身来源以外的任何来源（域、方案或端口），浏览器应允许从中加载资源

跨源资源共享（CORS）是一种基于 HTTP Header的机制，它允许服务器告知浏览器应允许从除其自身来源以外的某些来源（域、方案或端口）加载资源。

有点绕，用白话来解释就是，服务器告诉浏览器能不能访问它提供的资源。   

服务器是根据什么判断浏览器能不能访问的呢？是根据HTTP Header 信息。浏览器发送的请求中包含 Header信息，服务器根据这些头信息来决定是否允许访问所请求的资源。

出于安全的考虑，浏览器会限制脚本发起跨域请求。在前端常用的`fetch()` 和 Ajax技术（背后是[`XMLHttpRequest`](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest)） 都遵循 [same-origin policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy)这个策略。也就是默认只能访问同个origin下的资源。在HTTP协议下，这个origin就是主机和端口的组合。

举个例子：http://www.renhl.com/a.html下的页面里的脚本，访问以下的资源时：
1. 访问`https://www.renhl.com/api`, 被禁止，因为https和http是两种协议，判定为不是同一origin。
2. 访问`http://api.renhl.com/api`, 被禁止，因为`www.renhl.com`和`api.renhl.com`域名不同，判定为不是同一origin。
3. 访问`http://www.renhl.com:81/api`, 被禁止，因为80和81端口号不同，判定为不是同一origin。

CORS允许跨origin的访问，下面分三种情况来介绍一下它是如何工作的。

## 一、 简单请求 Simple Requests 。
简单请求是不触发[CORS preflight](https://developer.mozilla.org/en-US/docs/Glossary/Preflight_request)CORS预检的。只所以叫简单请求，是在CORS规范(已过时)中这样称呼的，新的`Fetch规范`中不再使用这个术语。

简单请求要满足下面条件：

*  下面的方法之一
	- GET
	- HEAD
	- POST


* 除了用户代理设置的Header以外，唯一允许手动设置的Header是 `Fetch 规范`定义为 CORS 安全列表请求Header，它们是：
	- [`Accept`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept)
	-  [`Accept-Language`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Language)
	- [`Content-Language`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Language)
	- [`Content-Type`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Type)  （请注意以下附加要求）
	- [`Range`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Range) （仅使用简单的范围Header值;例如， `bytes=256-` 或 `bytes=127-255` ）

		`Content-Type` Header中指定的媒体类型唯一允许的类型/子类型组合是：
	- application/x-www-form-urlencoded
	- multipart/form-data
	- text/plain

* 如果请求是使用`XMLHttpRequest`对象发出的，没有调用任何代码来添加事件侦听器 `xhr.upload.addEventListener()` 来监视上传。
* 请求中未使用任何 `ReadableStream` 对象。

简单请求虽然不需要CORS预检，但仍然需要服务器端开启CORS支持。

##  二、预检请求(Preflighted requests)    
非简单请求，则需要执行CORS预检。浏览器会向所请求资源的URL发送一个OPTIONS请求，根据服务器的响应确定实际请求是否可以安全发送。   

假设要请求的资源是https://www.renhl.com/new_gpt_from_json,那浏览器会先向这个URL发送OPTIONS，确认服务器允许访问后，再发送实际的请求。
可以通过浏览器的开发工具看到这一过程。
![](/assets/img/2024/02/01/1527012450.png)

预检请求是浏览器自动发送的，开发者不用也无法干预。

目前，并非所有浏览器都支持在预检请求后进行重定向。如果在此类请求后发生重定向，某些浏览器当前会报告如下错误消息：
```
The request was redirected to '[https://example.com/foo](https://example.com/foo)', which is disallowed for cross-origin requests that require preflight. Request requires preflight, which is disallowed to follow cross-origin redirects.
```
所以，如果后端开发者在写OPTIONS方法时，请注意不要给浏览器返回302状态码。

##  三、 带凭据的请求 ([Requests with credentials](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#requests_with_credentials))

默认情况下，在跨域 `fetch()` 或 `XMLHttpRequest` 调用中，浏览器不会发送凭据。如果要 `fetch()`请求包含凭据，请将 `Request()` 构造函数中的 `credentials` 选项设置为 `"include"` 。
如果使用的是`XMLHttpRequest` ，请将 `XMLHttpRequest.withCredentials` 该属性设置为 `true`。

下面是使用`fetch`的例子。
```js
const url = "https://bar.other/resources/credentialed-content/";
const request = new Request(url, { credentials: "include" });
const fetchPromise = fetch(request);
fetchPromise.then((response) => console.log(response));
```

下面是交互示例
```
GET /resources/credentialed-content/ HTTP/1.1
Host: bar.other
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:71.0) Gecko/20100101 Firefox/71.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-us,en;q=0.5
Accept-Encoding: gzip,deflate
Connection: keep-alive
Referer: https://foo.example/examples/credential.html
Origin: https://foo.example
Cookie: pageAccess=2

HTTP/1.1 200 OK
Date: Mon, 01 Dec 2008 01:34:52 GMT
Server: Apache/2
Access-Control-Allow-Origin: https://foo.example
Access-Control-Allow-Credentials: true
Cache-Control: no-cache
Pragma: no-cache
Set-Cookie: pageAccess=3; expires=Wed, 31-Dec-2008 01:34:53 GMT
Vary: Accept-Encoding, Origin
Content-Encoding: gzip
Content-Length: 106
Keep-Alive: timeout=2, max=100
Connection: Keep-Alive
Content-Type: text/plain

[text/plain payload]
```

要注意的是,如果响应中没有包含`Access-Control-Allow-Credentials: true`,那么浏览器会丢弃响应的内容。


**预检请求和凭据**
CORS 预检请求**不得**包含凭据。对预检请求的响应必须指定 `Access-Control-Allow-Credentials: true` 告诉浏览器可以使用凭据发出实际请求。

**凭据请求和通配符**
服务器在响应带凭据的请求时：
* 一定不能设置响应的Header `Access-Control-Allow-Origin`为`*`，必须指定明确的origin。
* 一定不能设置响应的Header `Access-Control-Allow-Headers`为`*`，必须明确指定Header的名称。
* 一定不能设置响应的Header `Access-Control-Allow-Methods`为`*`，必须明确指定请求方法名。比如 `Access-Control-Allow-Methods: POST, GET`。
* 一定不能设置响应的Header `Access-Control-Expose-Headers`为`*`，必须明确指定Header的名称。

## 在Rocket中实现CORS
下面讲一下在Rocket中如何实现CORS。
此前我一直研究如何把`Access-Control-Allow-Origin`这Header里放入多个网站。我的想法是想让我的服务器端可以让多个网站访问，我以为是需要在`OPTIONS`的响应里把`Access-Control-Allow-Origin`设置为多个Origin，这样浏览器发现当前的网站在这个列表中就发起实际的请求了。在研究了MDN上的规范后，我发现`Access-Control-Allow-Origin`只能设置为一个Origin或是*。不能设置为多个Origin，比如把多个Origin放在数组里`['http://www.renhl.com','https://www.renhl.com']`,这样是不行的。      

后来我想明白了，这个控制逻辑应该是放在`OPTIONS`请求的实现代码里，而不是在这个Header里。在实现代码里维护一个可以访问的列表，然后从request里取出Origin，看看在不在这个列表里，如果在，就设置Access-Control-Allow-Origin为这个Origin就行了。

下面是实现的代码，请参考。
```rust
use rocket::fairing::{Fairing, Info, Kind};
use rocket::http::Header;

pub struct CORS;

#[rocket::async_trait]
impl Fairing for CORS {
    fn info(&self) -> Info {
        Info {
            name: "Add CORS headers to responses",
            kind: Kind::Response,
        }
    }

    async fn on_response<'r>(&self, _request: &'r Request<'_>, response: &mut Response<'r>) {
        let allow_origins = ["https://renhl.com", "https://renhl.com"];

        let origin = _request.headers().get_one("origin").unwrap();

        if allow_origins.contains(&origin) {
            response.set_header(Header::new("Access-Control-Allow-Origin", origin));
        } else {
            response.set_header(Header::new("Access-Control-Allow-Origin", "null"));
        }

        response.set_header(Header::new(
            "Access-Control-Allow-Methods",
            "POST, GET, PATCH, OPTIONS",
        ));
        response.set_header(Header::new("Access-Control-Allow-Headers", "*"));
        response.set_header(Header::new("Access-Control-Allow-Credentials", "true"));
    }
}


async fn rocket() -> _ {
    let db = match set_up_db().await {
        Ok(db) => db,
        Err(err) => panic!("{}", err),
    };

    rocket::build()
        .manage(db)
        .attach(CORS)
        .mount("/", FileServer::from(relative!("/static")))
        .mount(
            "/",
            routes![
                index,
            ],
        )
        .register("/", catchers![not_found])
        .attach(Template::fairing())
}
```
