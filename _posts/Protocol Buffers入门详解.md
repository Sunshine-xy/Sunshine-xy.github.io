# Protocol Buffers 入门详解
## 1. 概念
### 1.1 What?（什么是Protocol Buffers？）
&emsp;&emsp;Protocol Buffers(后面简称protobuf)是google团队开发的一种语言中立，平台无关，可扩展的数据压缩编码方式（序列化），其很适合做数据存储或RPC数据交换格式。可用于通讯协议、数据存储等领域的语言无关、平台无关、可扩展的序列化结构数据格式。目前提供了 C++、Java、Python 三种语言的 API。

### 1.2 Why?（为什么使用Protocol Buffers？）
&emsp;&emsp;Protobuf的存在有两个原因，一个是为了从数据存储的角度来加速站点间数据传输速度，另一个是为了解决站点间数据传输的协议不规范问题。
传输数据的大小无疑是影响传输速度的关键因素，当前流行的数据传输协议（json、xml等）会携带一些“结构化”的数据（如标签等），另外，它们对数据的压缩也没有很极致，所以需要有一个对数据存储很“紧凑”，对数据压缩很高效的工具来进行革新。

&emsp;&emsp;最初的数据传输协议是request/response形式的，没有 protocol buffers 之前，google 已经存在了一种 request/response 格式，用于手动处理 request/response 的编码和反编码。但是这种协议往往没有很明确的格式，所以开发人员经常会遇到新旧版协议不兼容的问题。因此，急需一个协议不需要了解所有业务字段还能灵活地应对各种改动的需求。

### 1.3 How?（Protocol Buffers 是怎么做到的？）
&emsp;&emsp;protobuf对传输的数据采取一种最简单的key-value形式的存储方式（但其中有一种类型的数据不是k-v形式，后面会讲），这钟存储方式极大的节省了空间。除此之外protobuf还采取了varint(变长编码)形式来压缩数据，对体积较小的字段分配较少的空间，由此使得压缩后的文件非常“紧凑”。

&emsp;&emsp;protobuf定义了一种后缀名为“.proto”的描述型文件为待传输的结构化数据作为数据协议，待传输的数据必须符合“.proto”文件中的相关定义。“.proto”文件在简洁易读的同时很好地保留了原数据的结构信息，并且还给出了一些实用的关键字，灵活地让开发者对数据中的字段做选取，给了开发人员很大的发挥空间，基本解决了其他协议出现的被需求牵着走的局面。“.proto”文件可以看作一种数据传输的协议，需要开发人员按照语法编写，其还可以通过protobuf工具编译成C++/Java/Python对象。

## 2.protobuf文件中的语法规范
### 2.1 message结构
&emsp;&emsp;在“.proto”文件中protobuf将一种结构称为一个message，类似Java中的一个Entity对象，我们以电话簿中的数据为例：
![avatar](https://thumbnail0.baidupcs.com/thumbnail/2d3e1e124fb8a3aea2d8807ab1fa797a?fid=1209439364-250528-786409188237192&time=1567328400&rt=sh&sign=FDTAER-DCb740ccc5511e5e8fedcff06b081203-td2ZmnVDpzG9lkHCkWrNQpLgM00%3D&expires=8h&chkv=0&chkbd=0&chkpc=&dp-logid=5652514853055495734&dp-callid=0&size=c710_u400&quality=100&vuk=-&ft=video)
&emsp;&emsp;其中Person代表这个结构体的名字，name、id、email都是结构体中的字段，我们称之为**field**。要注意的是，等号右边赋值的不是value，而是字段field对应的**field id**。Field最前面的required、optional、repeated是这个Filed的规则，分别表示该数据结构中这个Filed有且只有1个，可以是0个或1个，可以是0个或任意个。optional后面可以加default默认值，如果不加，数据类型的默认为0，字符串类型的默认为空串。repeated后面加[packed=true]会使用新的更高效的编码方式。

### 2.2 enum类型
&emsp;&emsp;其意义和Java中的enum一致，在 message 中可以嵌入枚举类型

```
message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
  enum Corpus {
    UNIVERSAL = 0;
    WEB = 1;
    IMAGES = 2;
    LOCAL = 3;
    NEWS = 4;
    PRODUCTS = 5;
    VIDEO = 6;
  }
  Corpus corpus = 4;
}
```

&emsp;&emsp;枚举类型需要注意的是，一定要有 0 值。

&emsp;&emsp;枚举为 0 的是作为零值，当不赋值的时候，就会是零值。

&emsp;&emsp;为了和 proto2 兼容。在 proto2 中，零值必须是第一个值。

### 2.3 Service接口
&emsp;&emsp;果要使用 RPC（远程过程调用）系统的消息类型，可以在 .proto 文件中定义 RPC 服务接口，protocol buffer编译器将使用所选语言生成服务接口代码和stubs。所以，例如，如果你定义一个 RPC 服务，入参是 SearchRequest 返回值是 SearchResponse，你可以在你的 .proto 文件中定义它，如下所示：
```
service SearchService {
  rpc Search (SearchRequest) returns (SearchResponse);
}
```
### 2.4 命名规范
&emsp;&emsp;message、enum、Service都采用驼峰命名法。即首字母大写开头，其中message和enum中的字段名采用下划线分隔法命名。示例如下：
```
message SongServerRequest {
  required string song_name = 1;
}

enum Foo {
  FIRST_VALUE = 0;
  SECOND_VALUE = 1;
}

service FooService {
  rpc GetSomething(FooRequest) returns (FooResponse);
}

```
### 2.5 特殊情况
#### 2.5.1 为已弃用的field保存field id
&emsp;&emsp;之前说过，每个field对应唯一一个field id，但是在工作中可能会出现这么一种情况：“*即在项目中某field被弃用，之后的版本不再使用该field了，但是仍有一部分用户使用之前的开发版本，此时若完全将对应的field id不小心应用到新的field中，一定会引起混乱*”，所以此时应该将一些field id设置为**保留**类型，即不能被用来定义新的field，如果以后有人使用，protobuf编译器会提出出错。示例如下：
```
message Person {
  reserved 2, 15, 9 to 11;   #9 to 11表示9、10、11三个field id
  reserved "samples", "email";
}
```
注意一个reserved字段不能既有标签数字又有名字

## 3. Protobuf存储原理和压缩原理
### 3.1 存储原理
&emsp;&emsp;protobuf对不同对数据类型进行分类，分别为其选择不同对存储格式，不过多数数据对存储格式都是键值对的key-value形式。protobuf将所有数据类型分为了一下几类：
type|meaning| used for|
--|:--:|---:
 0|varint|int32,int64,uint32,uint64,sint32,sint64,bool,enum
 1|64-bit|fixed64,sfixed64,double
 2|Length-delimited|string,bytes,embedded message,packed repeated fields
 5|32-bit|fixed32,sfixed32,float
 &emsp;&emsp;其中类型3、4已经在proto3中被弃用，故不列出。除了类型2的数据之外的三个类型全部是由k-v形式存储，类型2数据由于长度不定，所以在存储的时候还需要加入一个length指标，以**k-l-v**的形式存储。

 &emsp;&emsp;对于message这种结构体数据，protobuf把message中所有字段的数据表示成k-v/k-l-v形式后拼接在一起，减少分隔符所占用的空间。这样一来有个问题出现了，那就是如何在一长串的拼接起来的二进制数据中找到对应的field？，并且还要确定该field是k-v存储还是k-l-v存储的呢？

&emsp;&emsp;这个问题的答案在存储field的“key”中，“key”由两个值决定，分别是field id和该field的类型编号，其计算方法是`key = （field_id << 3） | type`，其中|代表两个二进制类型的拼接，示例如下图所示：
![avatar](https://thumbnail0.baidupcs.com/thumbnail/24ab0d79f00ab5eaaf9c0ad604fb6bdf?fid=1209439364-250528-246421905780110&time=1567332000&rt=sh&sign=FDTAER-DCb740ccc5511e5e8fedcff06b081203-t%2F3d6qN0D0myZm5PCbCGvYDcWtA%3D&expires=8h&chkv=0&chkbd=0&chkpc=&dp-logid=5653697480621319797&dp-callid=0&size=c710_u400&quality=100&vuk=-&ft=video)
&emsp;&emsp;要注意的是，key在某些时地方也会被称作tag。

#### 3.1.1 varint类型存储（type=0）
&emsp;&emsp;这里需要特别说明一下的是，在type=0的数据类型中，除了正数整型外还存在负数整型，一个负数整型一般会被表示为一个很大的整数，因为计算机定义负数的符号位为数字的最高位。如果采用[Varints](https://blog.csdn.net/mweibiao/article/details/83036427)这种编码方式表示一个负数，那么一定需要 10 个byte长度。

&emsp;&emsp;然而protobuf定义了一种新的sint32/64类型，其采用[ZigZag](https://www.cnblogs.com/mler/p/10252886.html)编码方式，首先将所有整数映射成无符号整数（*即-1 将会被编码成 1，1 将会被编码成 2，-2 会被编码成 3，以此类推...*），然后再采用Varint编码方式编码。这样一来绝对值小的整数，编码后也会有一个较小的Varint编码值，无缝对接。

#### 3.1.2 32/64-bit类型存储（type=1、type=5）
&emsp;&emsp;&type=1时，在解析时告诉解析器，该类型的数据需要一个 64 位大小的数据块即可。
&emsp;&emsp;type=5时，float 和 fixed32 的 wire_type 为5，给其 32 位数据块即可。

#### 3.1.2 Length-delimited类型存储（type=2）
&emsp;&emsp;type=2时，field以k-l-v格式存储，现在假设一个message中有一个string类型的field,名字为b，如下所示：
```
message Test2 {
  optional string b = 2;
}
```
&emsp;&emsp;现在我们假设b被赋上的值为“testing”，7个英文字符，每个字符分别对应的十六进制码为“*74 65 73 74 69 6e 67*”。

&emsp;&emsp;以字符t为例，其二进制表示为*0111 0100*，将其右三位后为*0000 1110*，再与type=2的二进制*0000 0010*做“或”运算得到t字符最后的key/tag为*0000 1110*

&emsp;&emsp;确定了key/tag之后，接着是length，其为7——>“*0000 0111*”。length确定后接着跟value，即“testing”的7个字符的二进制表示。由此一个字符串类型的protobuf的k-l-v存储就结束了。

### 3.2 压缩原理
&emsp;&emsp;protobuf采取的压缩算法是[Varints](https://blog.csdn.net/mweibiao/article/details/83036427)，Varint是一种紧凑的、不定长的表示数字的算法，它用一个或多个字节来表示一个数字，其中值越小的数字使用越少的字节数，这样可以节省表示较小数字的空间。

## 4. Protobuf的特点
### 4.1 优点
&emsp;&emsp;**操作性简单**：无复杂的文档对象模型，自动生成类，语义清晰

&emsp;&emsp;**安全性强**：没有获得.proto文件就无法破解

&emsp;&emsp;**兼容性强**：具有向后兼容的特性，更新数据结构以后，老版本依旧可以兼容跨语言：支持C++、Python、Java


&emsp;&emsp;**压缩性能好**：压缩后的文件体积小（由于varint压缩和option等关键字的原因，也不像json、xml一样有<>等符号）

### 4.2 缺点
&emsp;&emsp;**可读性差**：在没有.proto文件的基础下无法人为读懂传递过来的数据

&emsp;&emsp;**表达能力差**：无法描述数据结构，无法对标记文档（html等）建模

## 5. 与其他序列化协议比较
&emsp;&emsp;|protobuf| JSON|XML|Lua
--|:--:|---:|---:|---:
数据结构支持|较复杂结构|简单结构|复杂结构|复杂结构
数据存储方式|二进制|文本|文本|文本
数据压缩大小|小|一般|大|一般
编码解码效率|快|一般|慢|稍快
编程语言支持|C++/Python/Java|很多|很多|很多
开发难度|简单|简单|繁琐|相对繁琐
学习成本|低|低|低|高
适用范围|数据交换|数据交换|数据交换|数据保存及脚本处理