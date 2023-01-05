---
layout: post
title: 为docker私有registry配置nginx反向代理
date: 2016-01-05 11:30:21
author: 任怀林
categories:
- blog
- docker
thumb: docker.png
isCJKLanguage: true
---


公司的docker私有registry已经搭建好了，用官方的registry image很容易就搭建好了。现在就是要用nginx的反向代理把它放出来，以便在外网可以访问。
我的[上一篇blog](http://www.renhl.com/blog/docker/nginx-ssl-reverse-proxy/) 讲了如何配置nginx反向代理。所以本文主要是讲我在使用中遇到的问题及解决方法。


这是我最初的nginx配置

```nginx
upstream my_docker_registry  {
    server 192.168.100.48:8443; # registry.renhl.com
}

## START hub.renhl.com ##
server {
    server_name registry.renhl.com;

    listen 80;
    listen 443 ssl;

    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

    root        /usr/local/nginx/html;
    index       index.html;

    allow 111.206.238.12;
    allow 111.206.238.94;
    deny all;

   location / {
        proxy_pass  https://my_docker_registry;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        proxy_redirect off;
        proxy_buffering off;
        proxy_set_header        X-Real-IP       $remote_addr;
        proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
## END hub.renhl.com  ##
```

然后我开始push image到这个registry上，发现报错了：

```
The push refers to a repository [hub.renhl.com/mediawiki] (len: 1)
unable to ping registry endpoint https://hub.renhl.com/v0/
v2 ping attempt failed with error: Get https://hub.renhl.com/v2/: x509: certificate is valid for renhl.com, not hub.renhl.com
 v1 ping attempt failed with error: Get https://hub.renhl.com/v1/_ping: x509: certificate is valid for renhl.com, not hub.renhl.com
```
我已经把这个私有registry的ssl证书放在/etc/docker/certs.d下，应该不会出错呀。仔细看了这个配置后，我发现nginx的没有使用私有registry的ssl证书，而是使用了自己的证书`/etc/nginx/ssl/nginx.crt`。问题应该出在这儿,把nginx的ssl证书换成私有registry的ssl证书。

``` nginx
# 使用私有registry的ssl证书
ssl_certificate /opt/renhl_com_docker_registry/certs/registry_renhl_com.crt;
ssl_certificate_key /opt/renhl_com_docker_registry/certs/registry_renhl_com.key;
```

好，重启nginx再Push一下试试，又报错了：

```
The push refers to a repository [registry.renhl.com/mediawiki] (len: 1)
846b3100eaa8: Buffering to Disk
dial tcp: lookup my_docker_registry: no such host
```

原因很清楚，反向代理把my_docker_registry，做为host发到了客户端了，要让反向代理设置正确的host。把下面的一行加到nginx配置里。

``` nginx
proxy_set_header        Host            $host;
```

重启nginx push一下试试，还报错：

```
Error parsing HTTP response: invalid character '<' looking for beginning of value: "<html>\r\n<head><title>413 Request Entity Too Large</title></head>\r\n<body bgcolor=\"white\">\r\n<center><h1>413 Request Entity Too Large</h1></center>\r\n<hr><center>nginx/1.4.6 (Ubuntu)</center>\r\n</body>\r\n</html>\r\n"
```

我push的imgage太大，被nginx拒绝了。问了Google以后，在nginx的配置加入下面的两行：

```nginx
client_max_body_size 0; # disable any limits to avoid HTTP 413 for large image uploads
# required to avoid HTTP 411: see Issue #1486 (https://github.com/docker/docker/issues/1486)
chunked_transfer_encoding on;
```

再push image，成功了！这是最后的配置：

```nginx
upstream my_docker_registry  {
    server 192.168.100.48:8443; # registry.renhl.com
}

## START registry.renhl.com ##
server {
    server_name registry.renhl.com;

    listen 80;
    listen 443 ssl;

    # 使用私有registry的ssl证书
    ssl_certificate /opt/renhl_com_docker_registry/certs/registry_renhl_com.crt;
    ssl_certificate_key /opt/renhl_com_docker_registry/certs/registry_renhl_com.key;


    root        /usr/local/nginx/html;
    index       index.html;

    allow 111.206.238.12;
    allow 111.206.238.94;
    deny all;


    client_max_body_size 0; # disable any limits to avoid HTTP 413 for large image uploads

    # required to avoid HTTP 411: see Issue #1486 (https://github.com/docker/docker/issues/1486)
    chunked_transfer_encoding on;

   location / {
        proxy_pass  https://my_docker_registry;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        proxy_redirect off;
        proxy_buffering off;
        proxy_set_header        Host            $host;
        proxy_set_header        X-Real-IP       $remote_addr;
        proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
## END registry.renhl.com  ##
```

如果你在配置docker的私有registry时碰到了同样的问题，希望这篇博客能帮到你，：）


**参考资料：**
http://stackoverflow.com/questions/31730319/how-to-push-docker-images-through-reverse-proxy-to-artifactory
