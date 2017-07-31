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

你怎么知道这是300？首先，去掉每个字节中的msb，因为msb的作用只是告诉我们是否到达了数字的末尾（正如你所看到的，第一个字节设置了msb，表示后续字节也是varint的一部分）：

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

正如你所知道的，protocol buffer消息是一系列键值对。一个二进制消息使用字段的编号作为键——字段的名字和声明类型只有在解码结束后参考消息类型定义（比如`.proto`文件）才能确定。

编码消息时，所有的键和值被拼接在一起写入字节流。解码消息时，分析器需要能够跳过无法识别的字段。这样的话，就可以在消息中加入新的字段时无须破坏旧程序，即使旧程序不知道这些新字段。为了这个目的，每个键值对的“key”实际上是由两个值组成——`.proto`文件中字段的编号和一个*wire type*，wire type提供了“value”的长度信息。

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

取最后3位可得wire type（0），然后右移3位可得字段编号（1）。现在你知道了tag是1，并且字段的值是varint类型。用上一节中学到的varint解码相关知识，你将会看到后面两个字节存储了150这个数字。

```protobuf
96 01 = 1001 0110  0000 0001
       → 000 0001  ++  001 0110 (drop the msb and reverse the groups of 7 bits)
       → 10010110
       → 2 + 4 + 16 + 128 = 150
```

## 其它值类型

### 有符号整型

正如你在上一节里看到的，protocol buffer中所有wire type为0的类型都被编码成varint。然而，有符号int类型（`sint32`和`sint64`）和“标准”int类型（`int32`和`int64`）这两者在处理负数编码时有很重要的区别。如果你使用`int32`或者`int64`作为一个负数的类型，varint编码后的结果*永远有10字节之长*，实际上就像是在处理一个非常大的无符号整型。如果你使用有符号int类型（`sint32`或`sint64`），varint将使用更高效的ZigZag（之字形）编码。

ZigZag编码将有符号整数映射到无符号整数，这样，具有较小绝对值的数字（例如-1）也具有较小的varint编码值。它以“之字形”来回处理正负整数，所以-1被编码成1，1被编码成2，-2被编码成3，以此类推，如下表所示：

| 有符号原始数字     | 编码为        |
| ----------- | ---------- |
| 0           | 0          |
| -1          | 1          |
| 1           | 2          |
| -2          | 3          |
| 2147483647  | 4294967294 |
| -2147483648 | 4294967295 |

换句话说，对每个`sint32`类型的`n`值以如下方式编码：

```protobuf
(n << 1) ^ (n >> 31)
```

对64位：

```protobuf
(n << 1) ^ (n >> 63)
```

注意，第二个位移操作——`(n >> 31)`——是一个算数位移。也就是说，位移的结果要么所有位全是0（如果`n`是正数），要么全是1（如果`n`是负数）。

当解析到`sint32`或是`sint64`时，对应的值被解码为原始的、有符号形式。

### 非varint数字

非varint数字类型比较简单——`double`和`fixed64`使用wire type 1来告诉解析器有一块64位大小的数据，同理，`float`和`fixed32`使用wire type 5来告诉解析器有一块32位大小的数据。两种类型中的数值均以小端字节序编码。

### 字符串

wire type 2（长度分隔）表示值为一个包含长度信息、携带指定个数字节的varint。

```protobuf
message Test2 {
  required string b = 2;
}
```

对b赋值“testing”将会得到：

```protobuf
12 07 74 65 73 74 69 6e 67
```

红色的字节（`74 65 73 74 69 6e 67`）是“testing”的UTF8编码。这里的键是0x12→ tag = 2, type = 2。表示长度的varint值为7，你瞧，我们看到在它后面有七个字节——我们的字符串。

## 嵌套消息

这里有一个message定义，它以嵌套了我们的示例消息：

```protobuf
message Test3 {
  required Test1 c = 3;
}
```

同样对Test1中的`a`赋值为150，编码后：

```protobuf
1a 03 08 96 01
```

可以看到，最后三个字节与第一个例子中的完全相同（`08 96 01`）。在它们前面是数字3——嵌套消息与字符串（wire type = 2）的处理方式完全相同。

## 可选和repeated元素

如果一个proto2消息定义了`repeated`元素（没有设置`[packed]=true`），那么编码后的消息会有0个或多个拥有相同tag编号的键值对。这些重复值不必连续，它们可能和其它的字段交错在一起。在解析时，元素相对于彼此的顺序保持不变，但是相对于其它字段的顺序信息会丢失。在proto3中，repeated字段使用[packed 编码][1]，你可以阅读下面的内容。

对proto3中的任何一个non-repeated字段，或是proto2中的`optional`字段，编码后的消息可能有，也可能没有这个tag编号字段的键值对。

通常来说，编码的消息永远不会有一个non-repeated字段的多个示例。然而，真碰到这种情况时我们期望解析器也能够正常处理。对数字类型和字符串，如果同一个字段出现多次，解析器将接受它所看到的最后哪个值。对于嵌套消息字段，解析器会合并同一个字段的的多个实例，就像使用`Message::MergeFrom`方法——所有的歧义字段都会用后一个实例中的字段替换，歧义嵌套消息被合并，并且repeated字段会拼接起来。这些规则的效果就是，解析两个消息的串联，与你分别解析这两条消息然后合并的结果完全相同。即：

```C++
MyMessage message;
message.ParseFromString(str1 + str2);
```

等于：

```c++
MyMessage message, message2;
message.ParseFromString(str1);
message2.ParseFromString(str2);
message.MergeFrom(message2);
```

这个特性有时候很有用，因为它允许你在完全不知道两个消息的类型时合并它们。

### Packed repeated字段

2.1.0版本中引入了packed repeated字段，在proto2中它被声明成带有`[packed=true]`选项的repeated字段。在proto3中，repeated字段默认会按packed处理。这些功能与repeated字段很类似，但是有不一样的编码规则。编码生成的消息中不会出现包含零元素的packed repeated字段。否则，所有的元素都被打包成一个wire type为2（长度分隔）的键值对。每个元素都按正常的、相同的方式编码，除了前面没有tag。

例如，想象你有这样一个消息类型：

```protobuf
message Test4 {
  repeated int32 d = 4 [packed=true];
}
```

现在，假设你构造了一个`Test4`，给repeated字段`d`设值3、270和86942。然后，编码的形式将是：

```protobuf
22        // tag (field number 4, wire type 2)
06        // payload size (6 bytes)
03        // first element (varint 3)
8E 02     // second element (varint 270)
9E A7 05  // third element (varint 86942)
```

只有原始数字类型的repeated字段（使用varint、32-bit、或者64-bit类型）才能声明为“packed”。

请注意，尽管通常没有理由将多个键值对编码成一个packed repeated字段，但编码器必须做好接受多个键值对的准备。在这种情况下，所有载荷（payloads）应该拼接到一起。每一对都必须包含完整的元素。

Protocol buffer解析器必须能够解析以`packed`方式编译而成的repeated字段，就好像它们没有被打包一样，反之亦然。这样就能允许以向前和向后兼容的方式向现有字段添加`[packed=true]`。

## 字段顺序

虽然可以在`.proto`中以任意顺序使用字段号，但当消息被序列化时，已知字段应该按字段号顺序写入，正如所提供的C++、Java和Python序列化代码。这使得解析代码可以使用依赖于字段号的优化。但是，protocol buffer解析器必须能够以任意顺序解析字段，因为并非所有消息都是通过简单地序列化一个对象来创建的——例如，通过简单地拼接两个消息来合并它们有时候是很有用的。

如果一个消息具有[未知字段][2]，当前的Java和C++实现会在顺序排列已知字段之后按任意顺序写入未知字段。当前的Python实现不处理未知字段。

## 版权说明

除另有说明外，本页面的内容是根据[知识共享署名3.0许可证][3]授权的，代码示例根据[Apache 2.0许可证][4]授权。有关详细信息，请参阅我们的[网站政策][5]。Java是甲骨文和/或其子公司的注册商标。


[1]: https://developers.google.com/protocol-buffers/docs/encoding#packed
[2]: https://developers.google.com/protocol-buffers/docs/proto.html#updating
[3]: http://creativecommons.org/licenses/by/3.0/
[4]: http://www.apache.org/licenses/LICENSE-2.0
[5]: https://developers.google.com/terms/site-policies
