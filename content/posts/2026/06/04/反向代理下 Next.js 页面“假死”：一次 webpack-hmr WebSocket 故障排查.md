---
title: 反向代理下 Next.js 页面“假死”：一次 webpack-hmr WebSocket 故障排查
date: 2026-06-04T22:28:57+08:00
draft: true

layout: post
author: 任怀林
thumb: docker.png
isCJKLanguage: true

slug: nextjs-nginx-reverse-proxy-hmr-websocket-troubleshooting
categories:
  - 开发
tags:
  - 开发
---




最近我在开发一个 Next.js 项目，为了在微信内置浏览器中调试，我把应用部署到了一台公网服务器上，并通过 Nginx 做反向代理。域名是 `mp.long9.net`，Next.js dev server 跑在本地 `3001` 端口。

配置好代理后，页面能正常打开，但很快就发现了一些诡异的现象。

---

##  navigator.userAgent 变成了 Node.js/24

我是通过UA来判断当前是不是微信浏览器，如果是，则显示用微信登录。但是我发现微信登录怎么着都不显示，我明明是在微信里打开的。于是我在登录页里写了一段调试代码，看一下UA是什么。

```tsx
<div>{navigator.userAgent}</div>
```

结果页面上显示的不是熟悉的 `Mozilla/5.0 (iPhone...)`，而是：

```
Node.js/24
```
这是什么鬼？？
我尝试在本地访问了一下服务，使用`localhost:3001`访问的时候是正常的。

我第一反应是：这肯定跟 Nginx 反向代理有关。因为直接访问 `localhost:3001` 时显示的是正常的浏览器 UA，走代理后就变成了 `Node.js/24`。

我怀疑是不是 Nginx 把 User-Agent 给改写或者丢弃了。

**此时我的判断是：Nginx 代理导致了请求头异常。**

---

##  所有 JS 都不执行了

继续排查时，我发现了一个更严重的问题：页面上的按钮点击没反应，`useEffect` 不触发，所有的客户端 JavaScript 仿佛都失效了。

但直接访问 `localhost:3001` 时一切正常。这就更加深了我的怀疑——Nginx 一定动了什么手脚。

我检查了 Nginx 配置，怀疑：
- 是不是 `_next/static` 静态资源路径被拦截了？
- 是不是 MIME 类型返回错了？
- 是不是 `try_files` 把 JS 请求都 fallback 到了 HTML？

这些猜测都有一定的道理，但修改后问题依旧。

---

##  有了一点端倪了： WebSocket 连接失败

打开浏览器 DevTools 的 Console，我看到了一条关键的报错：

```
WebSocket connection to 'wss://mp.long9.net/_next/webpack-hmr?id=...' failed:
```

原来页面一直在尝试连接一个 WebSocket 地址，但每次都失败。

通过和 AI 助手交流，我了解到这个 `/_next/webpack-hmr` 是 Next.js **开发模式**下的热更新通道。它的作用是：

> **Next.js 在开发时会启动一个 WebSocket 服务，当代码发生变更时，通过 `/_next/webpack-hmr` 把更新推送到浏览器。浏览器收到更新后，触发 React 的 hydration（水合）过程，让服务端渲染的 HTML 具备完整的客户端交互能力。**
>
> 如果这个 WebSocket 连接不上，浏览器端的 React 就无法正确接管页面，导致按钮点不了、事件不触发、`useEffect` 不执行——表现出来的现象就是"JavaScript 完全不执行"。

这就解释了为什么 SSR 阶段 `navigator.userAgent` 会显示 `Node.js/24`：因为 Next.js 在服务端预渲染时确实运行在 Node 环境里，正常情况下浏览器加载完 JS 后会通过 hydration 覆盖掉这个结果。但 HMR WebSocket 连不上，hydration 被卡住了，页面上就永远停留着 SSR 阶段的输出。

---

## 尝试过的解决方案

### 方案一：在 Nginx 中添加 WebSocket 转发头

AI 助手建议我在 Nginx 的 `location` 中添加：

```nginx
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
proxy_http_version 1.1;
```

这是 WebSocket 代理的标准配置。我加上了，重启了 Nginx，但问题没有解决，浏览器依然报同样的 WebSocket 连接失败。

**结论：必要的配置，但单独使用不能解决此问题。**

### 方案二：为 `/_next/webpack-hmr` 单独配置 location

我怀疑通用的 `location /` 没有正确匹配到 `/_next/webpack-hmr`，于是单独写了一个 location：

```nginx
location /_next/webpack-hmr {
    proxy_pass http://127.0.0.1:3001/_next/webpack-hmr;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 86400;
    proxy_buffering off;
}
```

reload 后，问题依然存在。

**结论：不是 Nginx 路径匹配的问题。**

### 方案三：查看 Nginx error.log

我查看了 `/var/log/nginx/error.log`，发现了这样的错误：

```
upstream sent no valid HTTP/1.0 header while reading response header from upstream
```

AI 助手分析这个日志时指出：Nginx 默认用 HTTP/1.0 与 upstream 通信，但 WebSocket 握手需要 HTTP/1.1 的 `101 Switching Protocols` 响应。不过我已经加了 `proxy_http_version 1.1`，所以这个日志本身有些误导性。

**结论：日志反映了症状，但不是真正的根因。**

---

## 最终的根因

就在我反复尝试 Nginx 配置无果时，我偶然在终端控制台（Next.js dev server 的输出）里看到了一段提示：

```
⚠ Blocked cross-origin request to Next.js dev resource /_next/webpack-hmr from "mp.long9.net".
Cross-origin access to Next.js dev resources is blocked by default for safety.

To allow this host in development, add it to "allowedDevOrigins" in next.config.js and restart the dev server:

// next.config.js
module.exports = {
  allowedDevOrigins: ['mp.long9.net'],
}
```

原来，**Next.js 15+ 在开发模式下默认会阻止跨域请求访问 dev 资源**。当通过 Nginx 代理后，浏览器请求的 `Host` 变成了 `mp.long9.net`，而 Next.js dev server 认为这是一个跨域访问，直接拒绝了 `/_next/webpack-hmr` 的 WebSocket 连接。

Nginx 本身已经把请求正确转发到了 Next.js，但 **Next.js 应用在应用层拒绝了这个请求**。

---

## 最终解决方案

修改 `next.config.ts`：

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
    output: 'standalone',
    experimental: {
        authInterrupts: true,
    },
    allowedDevOrigins: ['mp.long9.net'],
};

export default nextConfig;
```

**重启 dev server 后，WebSocket 连接成功，HMR 恢复正常，页面上的按钮、事件、`useEffect` 全部恢复正常工作。**

---

## Nginx 最终配置（供参考）

虽然根因在 Next.js，但 Nginx 的 WebSocket 转发配置依然是必需的。我目前使用的完整配置如下：

```nginx
server {
    listen 443 ssl;
    server_name mp.long9.net;

    # SSL 证书配置...

    location / {
        proxy_pass http://127.0.0.1:3001;
        proxy_http_version 1.1;
        
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 总结

这个问题的排查路径很曲折，表面现象是 "JS 不执行" 和 "UA 异常"，中间一度完全怀疑是 Nginx 配置问题，实际上却卡在 Next.js 15 新增的 `allowedDevOrigins` 安全机制上。

记录几点教训：

1. **`navigator.userAgent` 在 SSR 阶段返回 `Node.js/24` 是正常的**，不要因此怀疑代理。真正的问题是 hydration 没完成。
2. **Next.js dev 模式的 HMR 依赖 WebSocket**，如果它连不上，React 客户端就接管不了页面。
3. **Nginx 的 `proxy_http_version 1.1` 和 Upgrade header 是 WebSocket 代理的必要条件**，但仅此不够，应用层可能还有自己的安全限制。
4. **遇到代理后的开发环境异常，一定要同时查看浏览器 Console、Nginx error.log、以及 Next.js 服务端的终端输出**。我就是在服务端输出里找到答案的。

希望这篇文章能帮到同样在折腾 Next.js + Nginx 反向代理开发环境的朋友。