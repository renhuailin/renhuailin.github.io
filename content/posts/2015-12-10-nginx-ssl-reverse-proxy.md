---
layout: post
title: Nginx自签名https和反向代理
date: 2015-12-10 17:58:00
author: 任怀林
categories:
- blog
- docker
thumb: openstack.png
isCJKLanguage: true
---
# 场景
公司的wiki服务器和docker private registry都在公司的桌面云里，由于公网IP资源紧张，无法为这些服务器每个都配上公网IP，
只能通过一个公网IP来访问，所以需要用Nginx做个反向代理来访问些服务器。另外，这些服务都要以https来访问。

|服务器|内网IP|
|-----|-----|
|wiki.renhl.com|172.168.100.47|
|hub.renhl.com|172.168.100.48|


# 生成自签名的证书
因为是自己公司用也就无需申请认证的证书了，自签名即可。

```
$ sudo mkdir -p /etc/nginx/ssl
$ sudo openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout /etc/nginx/ssl/nginx.key -out /etc/nginx/ssl/nginx.crt
```



# 配置反向代理

编辑`/etc/nginx/sites-available/default`,加入如下内容：

```nginx
upstream wiki  {
	server 172.168.100.47:80; # wiki.renhl.com
}

upstream hub  {
	server 172.168.100.48; # hub.renhl.com
}

## Start wiki.renhl.com ##
server {

    listen 80;

    listen 443 ssl;

    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

	server_name  wiki.ecloud.com.cn;

    access_log  /var/log/nginx/wiki.renhl.access.log;
    error_log  /var/log/nginx/wiki.renhl.error.log;
    root   /usr/share/nginx/html;
    index  index.html index.htm;

    ## send request back to apache1 ##
    location / {
     proxy_pass  http://wiki;
     proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
     proxy_redirect off;
     proxy_buffering off;
     proxy_set_header        Host            $host;
     proxy_set_header        X-Real-IP       $remote_addr;
     proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
   }
}
## End wiki.renhl.com ##

## START hub.renhl.com ##
server {
   	server_name hub.renhl.com;

	listen 80;
    listen 443 ssl;

    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

   access_log  /var/log/nginx/hub.renhl.access.log;
   error_log   /var/log/nginx/hub.renhl.error.log;
   root        /usr/local/nginx/html;
   index       index.html;

   location / {
        proxy_pass  https://hub;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        proxy_redirect off;
        proxy_buffering off;
        proxy_set_header        Host           $host;
        proxy_set_header        X-Real-IP       $remote_addr;
        proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
## END hub.renhl.com  ##

```

# IP限制
出于安全的考虑，要禁止公司以外的人访问这些服务，在nginx里设置只允许公司的IP访问。在上面的两个配置里加入下面的内容：

```nginx
allow 111.206.238.12;
allow 111.206.238.94;
deny all;
```

# 參考文献
* https://www.digitalocean.com/community/tutorials/how-to-create-an-ssl-certificate-on-nginx-for-ubuntu-14-04
* http://www.cyberciti.biz/tips/using-nginx-as-reverse-proxy.html
