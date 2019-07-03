---
title: 轻量级在线游戏服务器框架skynet
date: 2019-04-16 11:30:27
categories: 算法
tags: []
---

## skynet是什么

![](http://ou1s4jkow.bkt.clouddn.com/skynet.png)

按照作者的说法，[*skynet是一个轻量级的为在线游戏服务器打造的框架* ][1]。 

## skynet可以做什么



## skynet怎么用

### 常用Lua API

* **skynet.newservice(name, ...)** 启动一个名为 name 的新服务
* **skynet.start(func)** 用 func 函数初始化服务，并将消息处理函数注册到 C 层，让该服务可以工作
* **skynet.dispatch(type, func)** 为 type 类型的消息设定一个处理函数。
* **skynet.fork(func, ...)** 启动一个新的任务去执行函数 func 
* **skynet.call(addr, type, ...)** 用 type 类型发送一个消息到 addr ，并等待对方的回应
* **skynet.send(addr, type, ...)** 用 type 类型向 addr 发送一个消息

### 目录结构

```
.
├── bin
│   └── skynet
├── conf
│   └── config.example
├── lib
│   ├── cservice
│   ├── luaclib
│   ├── lualib
│   └── service
├── script
└── src
    └── main.lua
```

###  配置文件

启动 skynet 服务需要提供一个配置文件，以下是一个简单的配置文件示例：

```lua
root = "./"
thread = 8
harbor = 0
start = "main"	-- main script
luaservice = root .. "src/?.lua;" .. root .. "lib/service/?.lua"
lualoader = root .. "lib/lualib/loader.lua"
lua_path = root .. "lib/lualib/?.lua;" .. root .. "lib/lualib/?/init.lua;"
lua_cpath = root .. "lib/luaclib/?.so;"
cpath = root .. "lib/cservice/?.so"
```

* **root** 项目根目录
* **thread** 启动多少个工作线程。通常不要将它配置超过你实际拥有的 CPU 核心数
* **bootstrap** skynet 启动的第一个服务以及其启动参数。默认配置为`snlua bootstrap`，即启动一个名为 bootstrap 的 lua 服务。通常指的是 service/bootstrap.lua 这段代码
* **harbor** 可以是 1-255 间的任意整数。一个 skynet 网络最多支持 255 个节点。每个节点有必须有一个唯一的编号。harbor 为 0 时skynet 工作在单节点模式下
* **start** 这是 bootstrap 最后一个环节将启动的 lua 服务，也就是你定制的 skynet 节点的主程序。默认为 main ，即启动 main.lua 这个脚本。这个 lua 服务的路径由下面的 **luaservice** 指定
* **lualoader** 用哪一段 lua 代码加载 lua 服务。通常配置为 lualib/loader.lua ，再由这段代码解析服务名称，进一步加载 lua 代码。snlua 会将下面几个配置项取出，放在初始化好的 lua 虚拟机的全局变量中。具体可参考实现
* **luaservice** lua 服务代码所在的位置。可以配置多项，以 ; 分割。 如果在创建 lua 服务时，以一个目录而不是单个文件提供，最终找到的路径还会被添加到 package.path 中。比如，在编写 lua 服务时，有时候会希望把该服务用到的库也放到同一个目录下
* **lua_path** 将添加到 package.path 中的路径，供 require 调用
* **lua_cpath** 将添加到 package.cpath 中的路径，供 require 调用


* **cpath** 用 C 编写的服务模块的位置，通常指 cservice 下那些 .so 文件。如果你的系统的动态库不是以 .so 为后缀，需要做相应的修改

### 启动skynet节点

`main.lua`

```lua
local skynet = require "skynet"

skynet.start(function()
	print("hello, skynet!")
	skynet.exit()
end)
```

使用skynet启动一个节点：

```
$ ./bin/skynet ./conf/config.example
[:00000001] LAUNCH logger 
[:00000002] LAUNCH snlua bootstrap
[:00000003] LAUNCH snlua launcher
[:00000004] LAUNCH snlua cdummy
[:00000005] LAUNCH harbor 0 4
[:00000006] LAUNCH snlua datacenterd
[:00000007] LAUNCH snlua service_mgr
[:00000008] LAUNCH snlua main
hello, skynet!
[:00000008] KILL self
[:00000002] KILL self
```






[1]: https://github.com/cloudwu/skynet/wiki	"skynet官方wiki"