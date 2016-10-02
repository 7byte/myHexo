---
title: 初尝openresty
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
``` shell
$ mkdir ~/openresty-test ~/openresty-test/logs/ ~/openresty-test/conf/
```
在刚刚创建的 ~/openresty-test/conf/ 目录下新建配置文件nginx.conf，可以从安装目录拷贝一份过来：
``` shell
$ cp /usr/local/openresty/nginx/conf/nginx.conf ~/openresty-test/conf/
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
``` shell
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
``` shell
$ sudo service openresty-test start
Starting openresty-test (via systemctl):                   [  OK  ]
```
看到上面的结果，说明启动成功了。然后通过curl在试着访问一下
``` shell
$ curl http://127.0.0.1
Hello, world!
```
访问 http://127.0.0.1 返回了“Hello, world!”，搭建完成。

  [1]: http://openresty.org/cn/
  [2]: http://nginx.org
  