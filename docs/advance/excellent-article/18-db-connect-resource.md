---
sidebar: heading
title: 为什么说数据库连接很消耗资源
category: 优质文章
tag:
  - 实践经验
head:
  - - meta
    - name: keywords
      content: 数据库连接
  - - meta
    - name: description
      content: 优质文章汇总
---

# 为什么说数据库连接很消耗资源

相信有过工作经验的同学都知道数据库连接是一个比较耗资源的操作。那么资源到底是耗费在哪里呢？

本文主要想探究一下连接数据库的细节，尤其是在`Web`应用中要使用数据库来连接池，以免每次发送一次请求就重新建立一次连接。对于这个问题，答案都是一致的，建立数据库连接很耗时，但是这个耗时是都多少呢，又是分别在哪些方面产生的耗时呢？

本文以连接`MySQL`数据库为例，因为`MySQL`数据库是开源的，其通信协议是公开的，所以我们能够详细分析建立连接的整个过程。

> 在本文中，消耗资源的分析主要集中在网络上，当然，资源也包括内存、`CPU`等计算资源，使用的编程语言是`Java`，但是不排除编程语言也会有一定的影响。

首先先看一下连接数据库的`Java`代码，如下：

```java
Class.forName("com.mysql.jdbc.Driver");

String name = "xttblog2";
String password = "123456";
String url = "jdbc:mysql://xxx:3306/xttblog2";
Connection conn = DriverManager.getConnection(url, name, password);
// 之后程序终止，连接被强制关闭
```

然后通过`Wireshark`,分析整个连接的建立过程，如下：

![](http://img.topjavaer.cn/img/db-connect-1.png)

## **Wireshark抓包**

在上图中显示的连接过程中，可以看出`MySQL`的通信协议是基于`TCP`传输协议的，而且该协议是二进制协议，不是类似于`HTTP`的文本协议，其中建立连接的过程具体如下：

- 第1步：建立`TCP`连接，通过三次握手实现；
- 第2步：服务器发送给客户端「握手信息」 ，客户端响应该握手消息；
- 第3步：客户端「发送认证包」 ，用于用户验证，验证成功后，服务器返回`OK`响应，之后开始执行命令；

用户验证成功之后，会进行一些连接变量的设置，**比如字符集、是否自动提交事务等，其间会有多次数据的交互**。完成了这些步骤后，才会执行真正的数据查询和更新等操作。

在本文的测试中，只用了`5`行代码来建立连接，但是并没有通过该连接去执行任何操作，所以在程序执行完毕之后，连接不是通过`Connection.close()`关闭的，而是由于程序执行完毕，导致进程终止，造成与数据库的连接异常关闭，所以最后会出现`TCP`的`RST`报文。

在这个最简单的代码中，没有设置任何额外的连接属性，所以在设置属性上占用的时间可以认为是最少的（其实，虽然我们没有设置任何属性，但是驱动仍然设置了字符集、事务自动提交等，这取决于具体的驱动实现），所以整个连接所使用的时间可以认为是最少的。

但从统计信息中可以看出，在不包括最后`TCP`的`RST`报文时（因为该报文不需要服务器返回任何响应），但是其中仍需在客户端和服务器之间进行往返「`7`」 次，「也就是说完成一次连接，可以认为，数据在客户端和服务器之间需要至少往返`7`次」 ，从时间上来看，从开始`TCP`的三次握手，到最终连接强制断开为止（不包括最后的`RST`报文），总共花费了：

```java
10.416042 - 10.190799 = 0.225243s = 225.243ms！！！
```

这意味着，建立一次数据库连接需要`225ms`，而这还是还可以认为是最少的，当然「花费的时间可能受到网络状况、数据库服务器性能以及应用代码是否高效的影响」 ，但是这里只是一个最简单的例子，已经足够说明问题了！

由于上面是程序异常终止了，但是在正常的应用程序中，连接的关闭一般都是通过`Connection.close()`完成的，代码如下：

```java
Class.forName("com.mysql.jdbc.Driver");

String name = "shine_user";
String password = "123";
String url = "jdbc:mysql://xxx:3306/clever_mg_test";
Connection conn = DriverManager.getConnection(url, name, password);
conn.close();
```

这样的话，情况发生了变化，主要体现在与数据库连接的断开，如下图：

![](http://img.topjavaer.cn/img/db-connect-2.png)

## **网络抓包**

- 第1步：此时处于`MySQL`通信协议阶段，客户端发送关闭连接请求，而且不用等待服务端的响应；
- 第2步：`TCP`断开连接，`4`次挥手完成连接断开；

这里是完整地完成了从数据库连接的建立到关闭，整个过程花费了：

```
747.284311 - 747.100954 = 0.183357s = 183.357ms
```

这里可能也有网络状况的影响，比上述的`225ms`少了，但是也几乎达到了`200ms`的级别。

那么问题来了，想象一下这个场景，对于一个日活`2万`的网站来说，假设每个用户只会发送`5`个请求，那么一天就是`10万`个请求，对于建立数据库连接，我们保守一点计算为`150ms`好了，那么一天当中花费在建立数据库连接的时间有（还不包括执行查询和更新操作）：

```
100000 * 150ms = 15000000ms = 15000s = 250min = 4.17h
```

也就说每天花费在建立数据库连接上的时间已经达到「`4个小时`」 ，所以说数据库连接池是必须的，而且当日活增加时，单单使用数据库连接池也不能完全保证你的服务能够正常运行，还需要考虑其他的解决方案：

- 缓存
- SQL 的预编译
- 负载均衡
- ……



总之，数据库连接真的很耗时，所以不要频繁的**建立连接**。

