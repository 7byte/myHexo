---
title: golang环境搭建
date: 2016-11-14 14:33:04
categories: golang
tags: [golang]
---

最近由于工作上的原因开始接触golang，虽然早前就已听说golang的大名，但是也仅仅只有一个大概的印象，比如其对C语言系程序员来说别扭的声明、集其它程序语言优秀设计于一身的大杂烩式语法、天生的并发机制、谷歌亲爹的强大背景。毫无疑问这是一门实用至上的语言，非常值得学习。
本文记录Linux和Windows下golang环境的搭建过程，以便将来查阅。内容参照官方安装说明文档（[安装golang][1]），以及安装包下载页面（[下载安装包][2]）。
本人目前还买不起Mac，故无法验证Mac上安装流程的可行性，等将来剁手了再来补充相关内容。

***温馨提示：部分步骤可能需要科学上网。***

## Linux环境

1. 下载压缩包，例如当前最新的版本是1.7.3：
```
$ wget https://storage.googleapis.com/golang/go1.7.3.linux-amd64.tar.gz
```
2. 提取压缩包内容到 /usr/local 目录：
```
$ sudo tar -C /usr/local -xzf go1.7.3.linux-amd64.tar.gz
```
3. 将 /usr/local/go/bin 添加到PATH环境变量，将此行添加到你的 /etc/profile（全系统安装）或 $HOME/.bash_profile 文件中：
```
export PATH=$PATH:/usr/local/go/bin
```
   然后让新添加的配置生效：
```
$ source /etc/profile
```
4. 添加自己的工作目录，将此行添加到 $HOME/.bash_profile 文件中：
```
GOPATH=/home/7byte/code/gowork
GOBIN=$GOPATH/bin
PATH=$PATH:$GOBIN

export GOPATH
export GOBIN
export PATH
```
   同样的让新添加的配置生效，gowork目录如果不存在则创建：
```
$ mkdir $HOME/gowork
$ source $HOME/.bash_profile
```
5. 验证安装。使用`go env`命令查看golang环境：
```
$ go env
GOARCH="amd64"
GOBIN="/home/7byte/code/gowork/bin"
GOEXE=""
GOHOSTARCH="amd64"
GOHOSTOS="linux"
GOOS="linux"
GOPATH="/home/7byte/code/gowork"
GORACE=""
GOROOT="/usr/lib/golang"
GOTOOLDIR="/usr/lib/golang/pkg/tool/linux_amd64"
GO15VENDOREXPERIMENT="1"
CC="gcc"
GOGCCFLAGS="-fPIC -m64 -pthread -fmessage-length=0"
CXX="g++"
CGO_ENABLED="1"
```
  在 $GOPATH/src/ 创建 hello 目录，新建 helloworld.go 文件并输入以下代码：
``` go
package main

import "fmt"

func main() {
    fmt.Printf("hello, world\n")
}
```
  运行我们的hello程序，如果得到以下输出说明配置成功。
```
$ go run helloworld.go 
hello, world
```

## Windows环境
Windows环境下golang环境搭建和Linux类似，官网上提供了msi安装包，安装过程和普通PC软件安装一致，基本只要“下一步”就可以了。安装程序会设置默认的`%GOROOT%\bin`（c:\Go\bin）目录到`Path`环境变量，如果自定义安装目录，需要手动修改`GOROOT`和`Path`环境变量。
同样的，我们需要添加自己的工作目录。在环境变量中添加`GOPATH`，变量值即工作目录，例如`E:\code\gowork`。
设置GOBIN环境变量`%GOPATH%\bin`，并添加到Path环境变量
验证安装的方式与Linux中相同，参考上一小节第5条。

## 安装git
必须安装git客户端，否则 go get 将无法正常执行。git的安装过程作为程序员的基本技能不在此详细记录。

## 安装golint和goimports
什么是[golint][4]：
> Golint is a linter for Go source code. 

wiki对[lint][5]的解释：
> 在计算机科学中，lint是一种工具程序的名称，它用来标记源代码中，某些可疑的、不具结构性（可能造成bug）的段落。它是一种静态程序分析工具，最早适用于C语言，在UNIX平台上开发出来。后来它成为通用术语，可用于描述在任何一种计算机程序语言中，用来标记源代码中有疑义段落的工具。

安装golint：
```
go get -u github.com/golang/lint/golint
```
安装完成后会在GOPATH\bin目录下生成一个名为`golint`的执行文件。golint支持对Go源文件、目录或包做静态分析，例如要检查当前目录下的所有Go源文件：
```
golint ./...
```

什么是[goimports][8]：
> Command goimports updates your Go import lines, adding missing ones and removing unreferenced ones.

安装goimports
```
go get golang.org/x/tools/cmd/goimports
```
后文对golint和goimports这两个工具在编辑器中的配置将做进一步说明。

## 测试工具GoConvey
完善的测试用例对提高代码质量的帮助不言而喻，在github上随手翻几个golang开源项目，可以发现每一个项目都带有详细的测试用例，可见为自己的golang代码写测试case已经是一项约定俗成的圈内规范。Go语言中自带有一个轻量级的测试框架testing和自带的go test命令来实现单元测试。但是go test的结果不够直观，并且每次都要手动敲命令也比较麻烦，所以我用到了[GoConvey][3]这个开源测试工具。
GoConvey使用起来非常简单：
```
$ go get github.com/smartystreets/goconvey
$ $GOPATH/bin/goconvey
```
然后在浏览器打开http://127.0.0.1:8080/ ，就能看到当前工程目录下所有*_test.go的测试情况了。
添加一个测试文件helloworld_test.go
``` go
package main

import (
    "fmt"
    "testing"
)

func TestHello(t *testing.T) {
    fmt.Println("test OK!")
}

```
保存之后可以看到浏览器页面几秒后自动刷新，测试通过效果图：
<div align=center>
![](/images/20161207214519.png)
</div>

修改TestHello的代码：
``` go
func TestHello(t *testing.T) {
    fmt.Println("test OK?")
    t.Error("test not OK!")
}
```
再次刷新后测试不通过：
<div align=center>
![](/images/20161207220023.png)
</div>

## 配置Sublime Text
[Sublime Text][6]是一款非常好用的编辑器，自带很多强大的文本编辑功能，并且有丰富的插件扩展，完全可以胜任golang日常实际开发。

[GoSublime][7]插件提供了很多类golang IDE的功能，例如自动代码补全等等，能够很好地提高开发效率。

Sublime的插件一般都是基于Python开发，所以测试机上Python环境是必不可少的。Linux系统默认自带Python，但如果是Windows则需要自己安装。与git安装相同，Python安装过程不在此详细记录。

Python环境搭建完成，然后我们首先需要安装Sublime插件管理器Package Control，借助该工具我们可以非常方便地搜索、安装、更新Sublime上的众多插件。依据官方安装说明的建议，我在这里只贴出原始安装文档页面链接，请到官方页面获取安装代码：https://packagecontrol.io/installation
<div align=center>
![](/images/20170113230448.jpg)
</div>

Package Control安装完成，使用快捷键`ctrl+shift+p`或者打开菜单栏 Preferences > Package Control 调出Package Control对话框，输入`install`，在下拉选项中选择`Package Control: Install Package`，然后输入`GoSublime`，Package Control将会安装对应插件。
接下来是对GoSublime插件的基本设置，打开菜单栏 Preferences > Package Settings > GoSublime > Settiongs-User 修改用户设置：
```
{
    "env":{
        "GOPATH":"E:/code/gowork"
    },
    "fmt_cmd": [
      "goimports"
    ],
    "on_save":[
        {
            "cmd":"gs9o_open",
            "args":{
                "run":[
                    "sh",
                    "go vet"
                ],
                "focus_view":false
            }
        },
        {
            "cmd":"gs9o_open",
            "args":{
                "run":[
                    "golint",
                    "."
                ],
                "focus_view":false
            }
        }
    ]
}
```
该配置参考了这篇文章：https://www.goinggo.net/2016/05/installing-go-and-your-workspace.html

至此，golang环境搭建基本完成，可以尽情地写代码了！

  [1]: https://golang.org/doc/install
  [2]: https://golang.org/dl/
  [3]: http://goconvey.co/
  [4]: https://github.com/golang/lint
  [5]: https://zh.wikipedia.org/wiki/Lint
  [6]: https://www.sublimetext.com/
  [7]: https://github.com/DisposaBoy/GoSublime
  [8]: https://godoc.org/golang.org/x/tools/cmd/goimports