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
当NGINX代理请求时，NGINX把请求发送给指定的代理服务器——获取到请求结果——把结果返回给客户端。请求可能被发送给HTTP服务器（另一个NGINX服务或者其它HTTP服务器），也可能通过特定的协议发送给非HTTP服务（运行基于特定框架开发的应用服务，例如PHP、Python）。支持的协议包括[FastCGI][1]、[uwsgi][2]、[SCGI][3]和[memcached][4]。

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

请注意，在这些情况下，指定地址的规则可能会有所不同。你可能还需要将附加参数传递到服务器（请参见[参考文档][11]的更多细节）。
[proxy_pass][5]指令也可以指向一组[命名服务器][12]。在这种情况下，请求按照[指定的方式][13]被分发到组中的服务器。

## 转发请求头

NGINX默认会重新定义代理请求中的两个header字段：Host和Connection，并且去除值为空字符串的字段。Host设置为**$proxy_host**变量，Connection设置为**close**。

如果要修改这些设置，或者修改其它header字段，就需要用到[proxy_set_header][14]指令。这条指令可以放在[location][6]或更高一级的位置，此外也可以放在特定的[server][15]上下文或者[http][16]块中，例如：

```
location /some/path/ {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_pass http://localhost:8000;
}
```

在这个配置中，Host字段被设置成[$host][17]变量。

为了防止一个header字段被传递给代理服务器，可以将其设置为空字符串：

```
location /some/path/ {
    proxy_set_header Accept-Encoding "";
    proxy_pass http://localhost:8000;
}
```

## 配置缓存

在默认情况下，NGINX会缓存来自代理服务器的响应。响应会存储在内部缓冲区中，直到接收到完整的数据才将响应内容发送给客户端，这样就有助于优化与慢客户端之间的交互体验，反之，如果响应以同步的方式从NGINX服务器发送给客户端，将会对代理服务器造成浪费。当启用缓存，NGINX允许代理服务器迅速地处理响应（而无须等待客户端接收），与此同时NGINX存储响应数据以保证客户端有充足的时间下载。

负责启用和禁用缓存的指令是[proxy_buffering][18]。默认情况下它被设置为**on**：启用缓存。

[proxy_buffers][19]指令控制分配给一个请求的缓冲区大小和数量。来自代理服务器的第一份响应数据被存储在单独的缓冲区，这块缓冲区的大小由[proxy_buffer_size][20]指令指定。该缓冲区通常包含相对较小的响应header以及剩余可以可以被塞进缓冲区的响应数据。

在下面的例子中，缓冲区的默认数量被加大了，并且第一份响应数据的缓冲区大小小于默认的缓冲区大小。

```
location /some/path/ {
    proxy_buffers 16 4k;
    proxy_buffer_size 2k;
    proxy_pass http://localhost:8000;
}
```

如果缓存被禁用，NGINX一旦接收到来自代理服务器的响应将会同步地发送给客户端。对要求尽可能实时的快速交互式客户端来说这种方式是可取的。

如果要禁用特定location中的缓存，则将location中的[proxy_buffering][18]设置为**off**，例如：

```
location /some/path/ {
    proxy_buffering off;
    proxy_pass http://localhost:8000;
}
```

在这种情况下，NGINX仅使用由[proxy_buffer_size][20]配置的缓存区来存储当前响应数据。

反向代理的一个常见用途是提供负载平衡。阅读电子书[ Five Reasons to Choose a Software Load Balancer][21]，了解如何提高性能、专注快速部署你的应用。

## 选择输出IP

如果你的代理服务器有多个网络接口，有时候你可能需要选择一个特定的源IP用于连接到代理服务器或上游服务器（upstream）。当NGINX后端的代理服务器配置为接受来自特定IP网络或IP地址范围的连接时，这样是设置是很有用处的。

指定[proxy_bind][22]指令和必要的网络接口的IP地址：

```
location /app1/ {
    proxy_bind 127.0.0.1;
    proxy_pass http://example.com/app1/;
}

location /app2/ {
    proxy_bind 127.0.0.2;
    proxy_pass http://example.com/app2/;
}
```

也可以用一个变量来指定IP地址。例如，[$var_server_addr][23]变量表示接收客户端请求的网络接口的IP地址：

```
location /app3/ {
    proxy_bind $server_addr;
    proxy_pass http://example.com/app3/;
}
```

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
  [11]:http://nginx.org/en/docs
  [12]:http://nginx.org/en/docs/http/load_balancing.html?#algorithms
  [13]:https://www.nginx.com/resources/admin-guide/load-balancer/
  [14]:http://nginx.org/en/docs/http/ngx_http_proxy_module.html?#proxy_set_header
  [15]:http://nginx.org/en/docs/http/ngx_http_core_module.html?#server
  [16]:http://nginx.org/en/docs/http/ngx_http_core_module.html?#http
  [17]:http://nginx.org/en/docs/http/ngx_http_core_module.html?#variables
  [18]:http://nginx.org/en/docs/http/ngx_http_proxy_module.html?#proxy_buffering
  [19]:http://nginx.org/en/docs/http/ngx_http_proxy_module.html?#proxy_buffers
  [20]:http://nginx.org/en/docs/http/ngx_http_proxy_module.html?#proxy_buffer_size
  [21]:https://www.nginx.com/resources/library/five-reasons-choose-software-load-balancer/
  [22]:http://nginx.org/en/docs/http/ngx_http_proxy_module.html?#proxy_bind
  [23]:http://nginx.org/en/docs/http/ngx_http_core_module.html?#var_server_addr