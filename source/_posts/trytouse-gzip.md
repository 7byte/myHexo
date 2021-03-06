---
title: nginx gzip
date: 2017-01-17 15:21:27
categories: openresty
tags: [openresty, nginx]
---

## nginx gzip
> The [ngx_http_gzip_module][1] module is a filter that compresses responses using the “gzip” method. This often helps to reduce the size of transmitted data by half or even more.

根据官网文档说明，通过开启nginx gzip压缩，数据传输量可以减少一半甚至更多。在网络带宽成为瓶颈的情况下，gzip带来的提升无疑是相当有诱惑力的。

**未开启gzip：**
``` shell
$ ./wrk -t10 -c200 -d30s http://120.25.176.131/testMysql
Running 30s test @ http://120.25.176.131/testMysql
  10 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     0.00us    0.00us   0.00us    -nan%
    Req/Sec     0.00      0.00     0.00      -nan%
  0 requests in 30.01s, 4.04MB read
Requests/sec:      0.00
Transfer/sec:    137.73KB
```
惨不忍睹，基本上是把服务器压死了。

**开启gzip：**
按照官网示例，在nginx.conf添加下面的配置
```
gzip            on;
gzip_min_length 1000;
gzip_proxied    expired no-cache no-store private auth;
gzip_types      text/plain application/xml;
```
压测结果
``` shell
$ ./wrk -t10 -c200 -d30s http://120.25.176.131/testMysql
Running 30s test @ http://120.25.176.131/testMysql
  10 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     0.00us    0.00us   0.00us    -nan%
    Req/Sec     0.00      0.00     0.00      -nan%
  0 requests in 30.02s, 4.02MB read
Requests/sec:      0.00
Transfer/sec:    137.12KB
```
好吧，从压测结果来看，并没有什么明显的改善。。。没办法，这台服务器配置实在够差。
但是，通过chrome浏览器提供的调试工具，可以看到API的返回数据两压缩了6倍左右，从用户的角度来说就是接口响应变快了，用户体验当然更好。

|-| 数据长度 | Waiting(TTFB) | Content Download |
|--|--------:|-----:|----:|
|关闭gzip| 734KB|20.67ms|4.56s|
|开启gzip| 229KB|29.58ms|699.19ms|


  [1]: http://nginx.org/en/docs/http/ngx_http_gzip_module.html
