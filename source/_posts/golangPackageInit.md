---
title: golang包初始化
date: 2017-05-20 17:16:27
categories: golang
tags: [golang]
---

相比于C/C++等语言，Golang提供了一个非常重要的特性：包。包可以提供类似其它编程语言中的库或者模块的功能，这在Java和Python等语言中已经很常见，但是golang的包自有其特点，比如禁止循环引用、通过首字母大小写区分可见性而不是使用`public`和`private`这样的关键字。

另外还有一个被大量使用的特性：init函数机制。

* 每个包的go文件都能包含任意数量的init函数；
* init函数不能被用户调用或引用；
* 多个init函数在程序启动时按照声明的顺序（执行main函数之前）自动调用。

<div align=center>![golang初始化顺序](/images/golang_init.png)</div>

（图片来自beego官方文档：https://beego.me/docs/quickstart/router.md ）

### 包内的初始化顺序

1. `const`常量首先被初始化。
2. 接着是包内的全局变量，初始化的顺序为变量的声明的顺序。但是当变量之间存在依赖关系时，优先初始化被依赖的变量。
3. 最后执行init函数。

当包内有多个go源文件时，`go tool`按照文件名排序后依次送入编译器初始化。

###  依赖包的初始化

在初始化一个包之前，必须完全初始化它所依赖的包。例如上图中的pkg1引入了pkg2，那么在初始化pkg1之前必须先初始化pkg2，而pkg2引入了pkg3，所以在初始化pkg2之前需要先初始化pkg3，pkg3没有引入任何包，所以直接执行包内初始化。由上图也可以发现，main总是最后一个被初始化的包。

另外还有一个很重要的特性：每个包只会初始化一次。例如，即使`fmt`被多个包引入，也只会在第一次时被初始化。

用下面的测试代码来验证一下以上特性。

`testInit/pkg2/pkg2.go`

```go
package pkg2

import "fmt"

var (
    X = 10
)

func init() {
    X++
    fmt.Printf("x init to %d in pkg2\n", X)
}
```

pkg2声明了一个可导出的变量`X`，初始值是10。init函数将`X`的值加`1`，并打印出`X`当前的值。

`testInit/pkg1/pkg1.go`

```go
package pkg1

import (
    "fmt"
    "testInit/pkg2"
)

func init() {
    pkg2.X += 2
    fmt.Printf("x init to %d in pkg1\n", pkg2.X)
}

func PrintX() {
    fmt.Printf("x = %d in pkg1\n", pkg2.X)
}
```

pkg1引入了pkg2，并且在inti函数中将`X`的值加2，然后打印出`X`当前的值。

`testInit/main.go`

```go
package main

import (
    "fmt"
    "testInit/pkg1"
    "testInit/pkg2"
)

func init() {
    pkg2.X += 3
    fmt.Printf("x init to %d in main\n", pkg2.X)
}

func main() {
    fmt.Printf("x = %d in main\n", pkg2.X)
    pkg1.PrintX()
}
```

main同时引入了pkg1和pkg2，并且在inti函数中将`X`的值加3，然后打印出`X`当前的值。

输出：

```
[ `go run main.go` | done: 541.4397ms ]
  x init to 11 in pkg2
  x init to 13 in pkg1
  x init to 16 in main
  x = 16 in main
  x = 16 in pkg1
```

不考虑`fmt`包，以上代码中有两条引用关系`main->pkg1->pkg2`和`main->pkg2`，但是从输出结果可以看到，pkg2只被初始化了一次，符合我们的预期。
