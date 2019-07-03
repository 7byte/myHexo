---
title: wrk压力测试
date: 2017-01-12 15:12:27
categories: 性能测试
tags: [性能测试]
---

[wrk][1]是一个开源的http性能测试工具，项目在github维护
wrk的使用非常简单，首先需要已经安装了git，gcc这两个基础工具，然后依次执行下面的3个命令：
``` shell
$ git clone https://github.com/wg/wrk.git  
$ cd wrk  
$ make
```
编译成功会在目录下生成执行文件wrk，准备工作完成。
参考github上给出的例子，做一下简单的使用说明。
```  shell
$ wrk -t12 -c400 -d30s http://127.0.0.1:8080/index.html
```
-t 使用的线程个数
-c HTTP连接的最大个数
-d 压测时间

  Output:
```
Running 30s test @ http://127.0.0.1:8080/index.html
  12 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   635.91us    0.89ms  12.92ms   93.69%
    Req/Sec    56.20k     8.07k   62.00k    86.54%
  22464657 requests in 30.00s, 17.76GB read
Requests/sec: 748868.53
Transfer/sec: 606.33MB
```
Latency：响应时间
Req/Sec：每个线程每秒完成的请求数
Requests/sec：每秒完成的请求数
Transfer/sec：每秒数据传输量

[1]: https://github.com/wg/wrk
