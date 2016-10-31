---
title: java.net.URI
date: 2016-10-31 10:47:11
categories:
- java 网络编程
---

URI(Uniform Resource Identifier) 统一资源标识符

# 形式

URI的最常见的形式是统一资源定位符。

URL是一种URI，所以两者非常相似，但是还是有点区别。

uri最多包含3个部分：
scheme:scheme-specific-part#fragment

如：
```java
uri = new URI("abc://username:password@www.example.com:123/path/data?key=苹果&key2=肉#fragid1");
scheme:				abc
schemeSpecificPart:	//username:password@www.example.com:123/path/data?key=苹果&key2=肉
fragment:			fragid1
```

没有模式部分的成为相对URI
有模式部分的成为绝对URI
可以用方法**public boolean isAbsolute()**判断
```java
public boolean isAbsolute() {
   return scheme != null;
}
```

许多uri中（如URL），模式的特定部分是分层结构：
scheme-specific-part=authority/path?query
authority=userinfo@host:port
这种情况下uri形式仍然可以这样子表示：
**scheme://userInof@host:port/path?query#fragment**

一个层次uri是透明的，非层的uri是不透明的（opaque），可以用方法进行判断
```java
// 层次uri=透明的：isOpaque=false
// 非层次uri=不透明：isOpaque=true
public boolean isOpaque()
```

对于层次uri（透明uri），可以通过和url类似的方式获得模式特定部分的不同部分。
对于非层次uri，只能得到模式，模式的特定部分，片段（fragment）。


# 创建URI

## 绝对URI

```java
public URI(String str) throws URISyntaxException

public URI(String scheme,
	String userInfo, String host, int port,
	String path, String query, String fragment)
        throws URISyntaxException

public URI(String scheme,
	String authority,
	String path, String query, String fragment)
        throws URISyntaxException

public URI(String scheme, String host, 
	String path, String fragment)
        throws URISyntaxException

public URI(String scheme, String ssp, String fragment)
        throws URISyntaxException

public static URI create(String str)
```

与URL不同，URI不依赖底层协议处理器，所以只会检查URI语法。

## 相对URI

```java
// 相对URI构造绝对URI
public URI resolve(URI uri)
// 相对URI构造绝对URI
public URI resolve(String str)
// 绝对URI构造相对URi
public URI relativize(URI uri)
```

# get方法

```java
// scheme 没有 raw get 方法
public String getScheme()
public String getSchemeSpecificPart()
public String getFragment()
// ====== 层次uri可以获取值 ======
public String getAuthority()
public String getUserInfo()
// host 没有 raw get 方法
public String getHost()
// port 没有 raw get 方法
public int getPort()
public String getPath()
public String getQuery()
// ====== ======

// raw get 方法：用来获取decoder之前的值
public String getRawSchemeSpecificPart()
public String getRawFragment()
public String getRawAuthority()
public String getRawUserInfo()
public String getRawPath()
public String getRawQuery()
```

# 检查URI语法

创建URI时，虽然会抛出URISyntaxException异常，但是并不一定会去检查语法
可以通过parseServerAuthority()方法主动检查URI的语法。

```java
public URI parseServerAuthority()
	throws URISyntaxException
```

# toString 和 toASCIIString

toString()返回未转义之前的值
toASCIIString()返回转义之前的值

# toURL

如果是绝对的URI，转化为URL。

```java
public URL toURL() throws MalformedURLException {
	if (!isAbsolute())
		throw new IllegalArgumentException("URI is not absolute");
	return new URL(toString());
}
```

# 参考：

1. https://en.wikipedia.org/wiki/Uniform_Resource_Identifier
2. 《java网络编程》（第四版）
