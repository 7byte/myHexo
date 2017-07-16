---
title: protobuf编码（翻译）
date: 2017-07-09 12:21:27
categories: 算法
tags: []
---

> 英文原文：https://developers.google.com/protocol-buffers/docs/encoding



本文描述了protocol buffer 消息的二进制格式。当你在你的应用中使用protocol buffer 时无需了解这些细节。但是，要想理解不同的protocol buffer 格式对最终编码生成的消息大小有何影响，了解这些将非常有帮助。

## 一个简单消息

假设你有如下简单的消息定义：

```protobuf
message Test1 {
  required int32 a = 1;
}
```

在应用中，创建一个名为`Test1`的消息并且对`a`赋值150。然后将这个消息序列化到一个输出流。如果查看编码出的消息内容，你将看到如下的三个字节：

```protobuf
08 96 01
```

只有几个数值——这些东西代表什么？且看下文……

## Base 128 Varints

在理解上面的简单消息是如何编码之前，你需要先了解什么是Varints。Varints是使用一个或多个字节对整型数字序列化的方法。数值越小，序列化后所占的字节数越少。

除了最后一个字节，Varints中的每个字节设置了最高有效位（msb）用来标示后续的字节也是该数字的一部分。一个以补码表示的数字按7位一组的方式分成若干组，每组存储在字节的低7位，低位数字在前面（即小端序）。

例如，数字1只有一个字节，所以不需要设置msb：

```protobuf
0000 0001
```

数字300要稍微复杂一些：

```protobuf
1010 1100 0000 0010
```

你如何知道这是300？首先去掉每个字节中的msb，因为msb的作用只是告诉我们是否到达了数字的末尾（正如你所看到的，第一个字节设置了msb，表示后续一个字节也是varint的一部分）：

```protobuf
1010 1100 0000 0010
→ 010 1100  000 0010
```

反转两组7位数值，因为varints将数字的低位放在前面。然后把两组数值拼接起来：

```protobuf
000 0010  010 1100
→  000 0010 ++ 010 1100
→  100101100
→  256 + 32 + 8 + 4 = 300
```

## 消息结构

正如你所知道的，protocol buffer消息就是一系列的键值对。一个消息的二进制版本使用域的编号作为键——域的名字和声明类型只有在解码结束后参考消息类型定义（比如`.proto`文件）才能确定。

消息编码时，所有的键值被连接在一起写入字节流。消息解码时，分析器需要跳过无法识别的域。通过这样的机制，即可在消息中加入新的域时无须破坏老程序。为了这个目的，每个键值对的“key”实际上是由两个值组成——`.proto`文件中域的编号和一个*wire type*，wire type提供了“value”的长度信息。

可用的wire type如下：

| 类型   | 含义               | 用途                                       |
| ---- | ---------------- | ---------------------------------------- |
| 0    | Varint           | int32, int64, uint32, uint64, sint32, sint64, bool, enum |
| 1    | 64-bit           | fixed64, sfixed64, double                |
| 2    | Length-delimited | string, bytes, embedded messages, packed repeated fields |
| 3    | Start group      | groups (deprecated)                      |
| 4    | End group        | groups (deprecated)                      |
| 5    | 32-bit           | fixed32, sfixed32, float                 |

消息流里的每个键是一个varint值：`(field_number << 3) | wire_type`，也就是说数字的低3位用来记录wire type。

再次回到我们的简单示例，你现在知道了字节流的第一个数字永远是一个varint类型的键值，即示例中的08，也就是（去掉msb后）：

```protobuf
000 1000
```

取最后3位可得wire type（0），然后右移3位可得域编号（1）。现在你知道了域序号是1，域的值是varint类型。用上一节中学到的varint解码相关知识，你将会看到后面两个字节确实存储了150这个数字。

```protobuf
96 01 = 1001 0110  0000 0001
       → 000 0001  ++  001 0110 (drop the msb and reverse the groups of 7 bits)
       → 10010110
       → 2 + 4 + 16 + 128 = 150
```

## 其它值类型

### 有符号整型

正如你在上一节里看到的，protocol buffer中所有wire type为0的类型都被编码成varint。然而，有符号int类型（`sint32`和`sint64`）和“标准”int类型（`int32`和`int64`）这两者在处理负数编码时有很大的区别。如果你使用`int32`或者`int64`作为一个负数的类型，varint编码后的结果*永远有10字节之长*，实际上就像是在处理一个非常大的无符号整型。如果你使用有符号int类型（`sint32`或`sint64`），varint将使用更高效的ZigZag（之字形）编码。

ZigZag编码将有符号整数映射到无符号整数，这样，具有较小绝对值的数字（例如-1）也具有较小的varint编码值。