---
title: slf4j
date: 2016-10-28 13:29:16
categories:
- java
---

**之前的个人博客使用jekyll搭建的，换了个mac，发现在本地用gem install jekyll完全装不了。**

**在一只狗头的推荐下，开始使用hexo，很不错。**

**重新开始，那么从前几天我在团队中分享的slf4j开始吧。**

# java 日志

当前java中存在的日志方案有很多种，但是主流的用法是接口+实现。

Simple Logging Facade for Java (SLF4J) 是一款优秀的框架，相信很多软件攻城狮早已经使用了。

网站: http://www.slf4j.org/

但是除了slf4j之外，还有 Jakarta Commons-Logging（jcl） ，Spring就是使用这个的。

这篇文章主要分析在java工程中，如何接日志。

# 接口 or 实现

面向接口的编程是大家提倡的，因此代码中，日志一般使用接口，而具体什么实现，由使用方自己决定。

我所了解到的主流接法是 slf4j + logback 的组合。

slf4j，logback，log4j和log4j2 都是同一个作者。

# 多种接口和实现的统一

一个常见的工程，往往会使用到多种依赖，而这些依赖会使用不同的日志接口，甚至有些依赖本身就不是面向接口的编程。

因此如果不采用统一的日志，那么各种日志会以不同的形式出现，难以管理。

如常见的一个SpringMvc工程，如果使用者采用slf4j，这时候就会出现slf4j+jcl共存的情况。

slf4j的网站上给出了如何处理这种常见的多种接口和实现的统一。

假设一个工程中用了2个依赖，分别是A，B，而使用者想用slf4j+logback。

- A使用了jcl
- B使用了log4j（B在编写中，没有采用面向接口的编程）

这时候，应该拆除依赖A和B中的jcl和log4j，并且加入依赖slf4j和logback。

```xml
	<properties>
		<slf4j.version>1.7.21</slf4j.version>
		<logback.version>1.1.7</logback.version>
	</properties>

    <dependencies>
    	<!-- A排除jcl -->
        <dependency>
            <groupId>xxx.A.xxx</groupId>
            <artifactId>A</artifactId>
            <version>${A.version}</version>
            <exclusions>
                <exclusion>
                    <groupId>commons-logging</groupId>
                    <artifactId>commons-logging</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <!-- B排除log4j -->
        <dependency>
            <groupId>xxx.B.xxx</groupId>
            <artifactId>B</artifactId>
            <version>${B.version}</version>
            <exclusions>
                <exclusion>
                    <groupId>log4j</groupId>
                    <artifactId>log4j</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <!-- 加入slf4j -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>log4j-over-slf4j</artifactId>
            <version>${slf4j.version}</version>
        </dependency>

        <!-- 加入logback -->
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-core</artifactId>
            <version>${logback.version}</version>
        </dependency>
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>${logback.version}</version>
        </dependency>
    </dependencies>
```

现实中很少会遇到其它日志框架，如果遇到了jul，因为这是jdk的一部分，那么肯定不能在pom中将它排除掉。

需要在代码中加入：

```java
SLF4JBridgeHandler.removeHandlersForRootLogger();
SLF4JBridgeHandler.install();
```

在pom中加入转接依赖：

```xml
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>jul-to-slf4j</artifactId>
    <version>${slf4j.version}</version>
</dependency>
```

然后就可以将jul的日志转到slf4j了，但是性能不好，官网有提示。

# 总结

用一副图来总结用法。

![](/images/java/slf4j-1.jpg)


