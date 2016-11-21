---
title: golang环境搭建
date: 2016-11-14 14:33:04
categories: golang
tags: [golang]
---

最近由于工作上的原因开始接触golang，虽然早前就已听说golang的大名，但是也仅仅只有一个大概的印象，比如其对C语言系程序员来说别扭的声明、集其它程序语言优秀设计于一身的大杂烩式语法、天生的并发机制、谷歌亲爹的强大背景。毫无疑问这是一门实用至上的语言，非常值得学习。
本文记录Linux和Windows下golang环境的搭建过程，以便将来查阅。内容参照官方安装说明文档（[安装golang][1]），以及安装包下载页面（[下载安装包][2]）（温馨提示：可能需要科学上网）。
本人目前还买不起Mac，故无法验证Mac上安装流程的可行性，等将来剁手了再来补充相关内容。

## Linux环境

1. 下载压缩包，例如当前最新的版本是1.7.3：
```
$ wget https://storage.googleapis.com/golang/go1.7.3.linux-amd64.tar.gz
```
2. 提取压缩包内容到 /usr/local 目录：
```
$ sudo tar -C /usr/local -xzf go1.7.3.linux-amd64.tar.gz
```
3. 将 /usr/local/go/bin 添加到PATH环境变量，将此行添加到你的 /etc/profile（全系统安装）或 $HOME/.profile 文件中：
```
export PATH=$PATH:/usr/local/go/bin
```
   然后让新添加的配置生效：
```
$ source /etc/profile
```
4. 添加自己的工作目录，将此行添加到 $HOME/.profile 文件中：
```
export GOPATH=$HOME/gowork
```
   同样的让新添加的配置生效，gowork目录如果不存在则创建：
```
$ mkdir $HOME/gowork
$ source $HOME/.profile
```
5. 验证安装。在 $GOPATH/src/ 创建 hello 目录，新建 hello.go 文件并输入以下代码：
``` go
package main

import "fmt"

func main() {
    fmt.Printf("hello, world\n")
}
```
  运行我们的hello程序，如果得到以下输出说明配置成功。
```
$ go run hello.go 
hello, world
```

## Windows环境
Windows环境下golang环境搭建和Linux类似，官网上提供了msi安装包，安装过程和普通PC软件安装一致，基本只要“下一步”就可以了。安装程序会设置默认的 GOROOT\bin（c:\Go\bin）目录到 PATH 环境变量，如果自定义安装目录，需要手动修改 GOROOT 和 PATH 环境变量。
同样的，我们需要添加自己的工作目录。在环境变量中添加GOPATH，变量值即工作目录，例如 E:\GOPATH。
验证安装的方式与Linux中相同，参考上一小节第5条。

  [1]: https://golang.org/doc/install
  [2]: https://golang.org/dl/