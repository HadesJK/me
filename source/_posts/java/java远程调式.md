---
title: java远程调式
date: 2016-12-12 14:04:37
categories:
---

# 困境

工作中遇到了本地环境能调用测试环境的服务(本地拨了VPN)，但是测试环境不能调用本地服务(dubbo服务)

因此想到使用远程调式来解决这个问题，所有服务都部署在远程测试环境中，本地使用intellij idea调试代码

# 远程调式

使用intellij远程调式需要准备2个工作：
1. 服务器开启允许远程调式
2. intellij配置远程地址和端口，关联本地代码

## 服务器

如果使用命令行运行的jar，即通过 **java -jar xxx.jar** 的方式，那么需要加入启动参数

```bash
java -Xdebug -Xnoagent -Djava.compiler=NONE -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=9527 -jar xxx.jar
```

看到网上说，这种方式要注意的一个点(未亲测)：-jar 参数写在最后。

**address=9527** 表示端口是9527

其它几个参数主要是：

**server=y/n** VM是否需要作为调试服务器执行

**suspend=y/n** 是否在调试客户端建立连接之后启动 VM

## intellij

intellij 的配置比较简单，如一下几个图所示：

**1. intellij 配置**

![](/images/java/远程调式/1.jpg)

**2. intellij 配置**

![](/images/java/远程调式/2.jpg)

**3. intellij 配置**

![](/images/java/远程调式/3.jpg)

**4. 服务器启动程序，并允许远程调式**

![](/images/java/远程调式/4.jpg)

**5. intellij 启动远程调式**

![](/images/java/远程调式/5.jpg)

**6. 输入url，访问服务器**

![](/images/java/远程调式/6.jpg)

**7. 断点进入intellij中**

![](/images/java/远程调式/7.jpg)

# 环境隔离

让人讨厌的是，在一些大型企业中经常需要各种权限的VPN才能登陆，并且出于安全考虑，对机器做了限制，比如一些特殊的服务只能被部署在本机上的应用访问，对外部进行了隔离。

这种情况下远程debug时，就需要用ssh进行端口转发了。

当然首先是服务器端开始了debug模式，假设端口还是9527.

```bash
ssh -L <local port>:localhost:<remote port> user@host -p <port> [-v -v]
```
按上面的例子，假设远程端口是9527，那么可以设置本地端口也为9527，user@port -p <port> 就是ssh的登录

-v -v 可以不加。

完整的命令可以是：

```bash
ssh -L 9527:localhost:9527 yifan@10.13.81.180 -p 10086 
```

在intellij中配置的host为 **localhost**， port为 **9527**















