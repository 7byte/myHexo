---
title: openresty中连接池的应用
date: 2018-03-05 15:16:27
categories: openresty
tags: [openresty, nginx]
---

在各种形式的数据交互中，创建连接、传输数据、销毁连接这三个步骤都是必不可少的。但是，当并发量持续增加时，耗费在创建和销毁连接上的时间将越来越不容忽视，所以理想中的情况应该是这样：创建连接-传输数据-……-传输数据-销毁连接。连接池正是为了解决这种问题而产生的技术。
在 OpenResty 中，所有具备 set_keepalive 的类、库函数，说明他都是支持连接池的。这里举个mysql连接池的例子，其它类型的连接池比如redis连接池、线程池、内存池思路是一样的。

``` lua
local MYSQL = require("resty.mysql")
local JSON  = require("cjson")

local mysql_server_ip                       =   "127.0.0.1"
local mysql_server_port                     =   3306
local mysql_databases                       =   "mysql"
local mysql_user_name                       =   "7byte"
local mysql_user_pass                       =   "abcdefg"
local mysql_max_packet_size                 =   1024 * 1024

-- 获取mysql连接，mysql:connect先在连接池中查找是否有可用的连接，没有才创建一个新连接
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

-- 释放mysql连接，放入连接池
function releaseConn(mysqlConn)
    local ok = true
    local db_type = 0
    local err = ""
    if mysqlConn then
        local res, err, errno, sqlstate = mysqlConn:read_result()
        while err == "again" do
            res, err, errno, sqlstate = mysqlConn:read_result()
        end
        -- setkeepalive(maxidletimeout, poolsize)
        local ok, err = mysqlConn:set_keepalive(0, 1000)
        if not ok then
            mysqlConn:close()
            ok = false
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

wrk本机压测结果。
不使用连接池
```
$ ./wrk -t10 -d30s -c200 http://127.0.0.1/testMysql 
Running 30s test @ http://127.0.0.1/testMysql
  10 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.16s   386.87ms   1.99s    62.06%
    Req/Sec    19.50     15.57    90.00     79.12%
  2454 requests in 30.08s, 1.72GB read
  Socket errors: connect 0, read 0, write 0, timeout 100
Requests/sec:     81.59
Transfer/sec:     58.51MB
```
使用连接池
``` shell
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