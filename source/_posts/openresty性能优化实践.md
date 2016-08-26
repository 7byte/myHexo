---
title: openresty性能优化实践
date: 2016-08-20 14:07:06
categories: openresty
tags: [openresty, nginx]
---
## 测试机配置
- CPU： 1核 Intel(R) Xeon(R) CPU E5-2650 v2 @ 2.60GHz 
- 内存： 1024 MB
- 操作系统： CentOS 7.2 64位
- 带宽： 1Mbps
没错，就是阿里云最低配的服务器。

## OpenResty
> [OpenResty][1] ™ 是一个基于 [Nginx][2] 与 Lua 的高性能 Web 平台，其内部集成了大量精良的 Lua 库、第三方模块以及大多数的依赖项。用于方便地搭建能够处理超高并发、扩展性极高的动态 Web 应用、Web 服务和动态网关。

### 安装OpenResty
因为测试机安装的是CentOS系统，所以我选择了OpenResty官方Yum资源库提供的PRM包。
首先在CentOS系统中添加openresty资源库，创建一个名为/etc/yum.repos.d/OpenResty.repo 的文件，内容如下:
```
[openresty]
name=Official OpenResty Repository
baseurl=https://copr-be.cloud.fedoraproject.org/results/openresty/openresty/epel-$releasever-$basearch/
skip_if_unavailable=True
gpgcheck=1
gpgkey=https://copr-be.cloud.fedoraproject.org/results/openresty/openresty/pubkey.gpg
enabled=1
enabled_metadata=1
```
然后就可以通过yum指令安装openresty了
``` shell
$ sudo yum install openresty
```
其它操作系统和安装方式，可以参考OpenResty官网说明。

### 创建自己的应用目录
为了避免污染/usr/local/openresty/下的OpenResty安装内容，我们最好创建自己的工作目录：
```
$ mkdir /home/7byte/openresty-test /home/7byte/openresty-test/logs/ /home/7byte/openresty-test/conf/
```
在刚刚创建的 /home/7byte/openresty-test/conf/ 目录下新建配置文件nginx.conf，可以从安装目录拷贝一份过来：
``` shell
$ cp /usr/local/openresty/nginx/conf/nginx.conf /home/7byte/openresty-test/conf/
```
修改nginx.conf，返回经典的“Hello, world!”
```
worker_processes  1;        #nginx worker 数量

events {
    worker_connections 1024;
}

http {
    server {
    listen 80;              #监听端口
        
    error_log /home/7byte/openresty-test/logs/error.log info;
    access_log /home/7byte/openresty-test/logs/access.log;

    location / {
            default_type text/html;
            content_by_lua_block {
                ngx.say("Hello, world!")
            }
        }
}
```
为了能够使用service对openresty-test执行start、stop、reload等操作，我们还需要在 /etc/init.d/ 目录下创建一个启动脚本openresty-test，内容参考默认的 /etc/init.d/openresty
```
#!/bin/sh
#
# openresty - this script starts and stops the nginx daemon of OpenResty
#
# chkconfig:   - 85 15
# description: OpenResty is a scalable web platform by extending
#              NGINX with Lua
# processname: openresty
# config:      /usr/local/openresty/nginx/conf/nginx.conf
# config:      /etc/sysconfig/openresty
# pidfile:     /usr/local/openresty/nginx/logs/nginx.pid

# Source function library.
. /etc/rc.d/init.d/functions

# Source networking configuration.
. /etc/sysconfig/network

# Check that networking is up.
[ "$NETWORKING" = "no" ] && exit 0

nginx="/usr/local/openresty/nginx/sbin/nginx"
prog=$(basename $nginx)
pidfile=/home/7byte/openresty-test/logs/nginx.pid

NGINX_CONF_FILE="/home/7byte/openresty-test/conf/nginx.conf"

[ -f /etc/sysconfig/openresty ] && . /etc/sysconfig/openresty

lockfile=/var/lock/subsys/openresty

start() {
    [ -x $nginx ] || exit 5
    [ -f $NGINX_CONF_FILE ] || exit 6
    echo -n $"Starting $prog: "
    daemon $nginx -c $NGINX_CONF_FILE
    retval=$?
    echo
    [ $retval -eq 0 ] && touch $lockfile
    return $retval
}

stop() {
    echo -n $"Stopping $prog: "
    killproc $prog -QUIT
    retval=$?
    echo
    [ $retval -eq 0 ] && rm -f $lockfile
    return $retval
}

restart() {
    configtest || return $?
    stop
    sleep 1
    start
}

reload() {
    configtest || return $?
    echo -n $"Reloading $prog: "
    killproc $nginx -HUP
    RETVAL=$?
    echo
}

force_reload() {
    restart
}

configtest() {
    $nginx -q -t -c $NGINX_CONF_FILE
}

rh_status() {
    status $nginx
}

rh_status_q() {
    rh_status >/dev/null 2>&1
}

case "$1" in
    start)
        rh_status_q && exit 0
        $1
        ;;
    stop)
        rh_status_q || exit 0
        $1
        ;;
    restart|configtest)
        $1
        ;;
    reload)
        rh_status_q || exit 7
        $1
        ;;
    force-reload)
        force_reload
        ;;
    status)
        rh_status
        ;;
    condrestart|try-restart)
        rh_status_q || exit 0
        ;;
    *)
        echo $"Usage: $0 {start|stop|status|restart|condrestart|try-restart|reload|force-reload|configtest}"
        exit 2
esac
```
保存退出后，下一步启动openresty-test。需要注意的是，如果开启了防火墙，需要放开nginx默认监听的80端口，或是其它自定义的端口
```
$ sudo service openresty-test start
Starting openresty-test (via systemctl):                   [  OK  ]
```
看到上面的结果，说明启动成功了。然后通过curl在试着访问一下
```
$ curl http://127.0.0.1
Hello, world!
```
访问 http://127.0.0.1 返回了“Hello, world!”，搭建完成。

## 火焰图
（待完善）
## wrk压测
[wrk][3]是一个开源的http性能测试工具，项目在github维护
wrk的使用非常简单，首先需要已经安装了git，gcc这两个基础工具，然后依次执行下面的3个命令：
``` shell
$ git clone https://github.com/wg/wrk.git  
$ cd wrk  
$ make
```
编译成功会在目录下生成执行文件wrk，准备工作完成。
参考github上给出的例子，做一下简单的使用说明。
``` 
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

## mysql连接池
``` lua
local MYSQL = require("resty.mysql")
local JSON  = require("cjson")

local mysql_server_ip                       =   "127.0.0.1"
local mysql_server_port                     =   3306
local mysql_databases                       =   "mysql"
local mysql_user_name                       =   "7byte"
local mysql_user_pass                       =   "abcdefg"
local mysql_max_packet_size                 =   1024 * 1024

function getMysql()
    local mysql = MYSQL:new()
    mysql:set_timeout(1000)
    local ok, err, errno, sqlstate = mysql:connect{
        host = mysql_server_ip,
        port = mysql_server_port,
        database = mysql_databases,
        user = mysql_user_name,
        password = mysql_user_pass,
        max_packet_size = mysql_max_packet_size,
        charset=utf8 }
    if not ok then
        return nil, err, errno, sqlstate
    end
    return mysql,nil,nil,sqlstate
end

function releaseConn(mysqlConn)
    local ok = true
    local db_type = 0
    local err = ""
    if mysqlConn then
        local res, err, errno, sqlstate = mysqlConn:read_result()
        while err == "again" do
            res, err, errno, sqlstate = mysqlConn:read_result()
        end
        local ok, err = mysqlConn:set_keepalive(0, 100)
        if not ok then
            mysqlConn:close()
            ok = false
            db_type = db_type + mysql_type
            err = "MySQL.Error ( "..(err or "null").." ) "
        end
    end 
    return ok, db_type, err
end

local mysql, err = getMysql()
if not mysql then
    ngx.say(err)
else
    local sql = "SELECT * FROM help_topic"
    local res1, err, errno, sqlstate = mysql:query(sql)
    -- ngx.log(ngx.ERR, "get_reused_times: "..mysql:get_reused_times())
    if not res1 then
        ngx.say(JSON.encode({"query failed!"}))
    else
        ngx.say(JSON.encode(res1))
    end
    releaseConn(mysql)
    ngx.exit(ngx.HTTP_OK)
    return
end
```
本机压测结果：
```
$ ./wrk -t10 -d30s -c200 http://127.0.0.1/testMysql
Running 30s test @ http://127.0.0.1/testMysql
  10 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    93.98ms  113.50ms   1.81s    99.24%
    Req/Sec    57.05     31.40   111.00     47.95%
  2673 requests in 30.09s, 1.87GB read
  Socket errors: connect 0, read 18, write 0, timeout 30
Requests/sec:     88.84
Transfer/sec:     63.71MB
```
## nginx gzip
> The [ngx_http_gzip_module][4] module is a filter that compresses responses using the “gzip” method. This often helps to reduce the size of transmitted data by half or even more.

根据官网文档说明，通过开启nginx gzip压缩，数据传输量可以减少一半甚至更多。在网络带宽成为瓶颈的情况下，gzip带来的提升无疑是相当有诱惑力的。

**未开启gzip：**
```
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
```
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


  [1]: http://openresty.org/cn/
  [2]: http://nginx.org
  [3]: https://github.com/wg/wrk
  [4]: http://nginx.org/en/docs/http/ngx_http_gzip_module.html