---
title: nginx反向代理（翻译）
date: 2016-12-11 11:16:27
categories: openresty
tags: [nginx,反向代理]
---

> 英文原文：https://www.nginx.com/resources/admin-guide/reverse-proxy/

本文描述了代理服务器的基本配置。你将学会怎样使用各种协议把一个请求从NGINX转发到代理服务器、怎样修改发送给代理服务器的客户端请求头，以及怎样为来自代理服务器的请求响应配置缓存。

## 目录
* 介绍
* 转发请求到代理服务器
* 转发请求头
* 配置缓存
* 选择输出IP

## 介绍
代理通常用来把负载分发到若干服务器，由不同的网站无缝地提供内容，或者经由非HTTP的其它协议转发请求到应用服务器做处理。

## 转发请求到代理服务器
当NGINX代理请求时，NGINX把请求发送给指定的代理服务器、获取到请求结果、把结果返回给客户端。请求可能被发送给HTTP服务器（另一个NGINX服务或者其它HTTP服务器），也可能通过特定的协议发送给非HTTP服务（运行有基于特定框架开发的应用服务，例如PHP、Python）。支持的协议包括[FastCGI][1]、[uwsgi][2]、[SCGI][3]和[memcached][4]。

为了把一个请求转发到HTTP代理服务器，将[proxy_pass][5]指令添加到[location][6]里面。例如：
```
location /some/path/ {
    proxy_pass http://www.example.com/link/;
}
```
这个配置示例会使所有由该location处理的请求转发到指定地址的代理服务器。这里的地址可以指定为一个域名或是一个IP，地址也可以带上端口：
```
location ~ \.php {
    proxy_pass http://127.0.0.1:8000;
}
```
注意到在第一个示例中，代理服务器的地址后面是一个URI：**/link/**。如果地址中指定了该URI，那么它会替换掉原始请求URI中与location参数匹配的部分。例如，这里的原始请求URI包含**/some/path/page.html**，转发的请求会替换成**http://www.example.com/link/page.html**。如果地址中没有指明URI，或者无法判断需要替换请求URI的哪部分，则会转发完整的请求（可能被修改过）。


为了把一个请求转发到非HTTP代理服务器，必须使用适当的****_pass**指令：
* [fastcgi_pass][7] 转发请求到FastCGI服务器
* [uwsgi_pass][8] 转发请求到uwsgi服务器
* [scgi_pass][9] 转发请求到SCGI服务器
* [memcached_pass][10] 转发请求到memcached服务器

## 转发请求头

## 配置缓存

## 选择输出IP

  [1]:http://nginx.org/en/docs/http/ngx_http_fastcgi_module.html
  [2]:http://nginx.org/en/docs/http/ngx_http_uwsgi_module.html
  [3]:http://nginx.org/en/docs/http/ngx_http_scgi_module.html
  [4]:http://nginx.org/en/docs/http/ngx_http_memcached_module.html
  [5]:http://nginx.org/en/docs/http/ngx_http_proxy_module.html?#proxy_pass
  [6]:http://nginx.org/en/docs/http/ngx_http_core_module.html?#location
  [7]:http://nginx.org/en/docs/http/ngx_http_fastcgi_module.html?#fastcgi_pass
  [8]:http://nginx.org/en/docs/http/ngx_http_uwsgi_module.html?#uwsgi_pass
  [9]:http://nginx.org/en/docs/http/ngx_http_scgi_module.html?#scgi_pass
  [10]:http://nginx.org/en/docs/http/ngx_http_memcached_module.html?#memcached_pass