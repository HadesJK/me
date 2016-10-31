---
title: java.net.URL
date: 2016-10-29 15:20:22
categories:
- java 网络编程
---

URL(Uniform Resource Locator) 统一资源定位符。

# 形式

**protocol://userInof@host:port/path?query#fragment**

protocol需要是java支持的协议，如http、https、ftp等

userInfo是可选的

port是可选的

authority=userInof@host:port

file=/path?query
**file在java.net.URL中必须以/开头！**

如：
```java
url = new URL("http://username:password@www.example.com:123/path/data?key=value&key2=value2#fragid1");
// 各个部分：
protocol:	http
userInfo:	username:password
host:		www.example.com
port:		123
authority:	username:password@www.example.com:123
path:		/path/data
query:		key=value&key2=value2
file:		/path/data?key=value&key2=value2
ref:		fragid1
```

# 创建URL

```java
// spec=url
public URL(String spec) throws 
	MalformedURLException

// 相对URL，移除context的file，将spec接到最后
public URL(URL context, String spec)

// port 采用协议的default port值
public URL(String protocol, String host, String file) 
	throws MalformedURLException

public URL(String protocol, String host, int port, String file) 
	throws MalformedURLException

public URL(URL context, String spec, URLStreamHandler handler) 
	throws MalformedURLException

public URL(String protocol, String host, int port, String file, URLStreamHandler handler) 
	throws MalformedURLException
```

```java
// http://www.jinqiliang.me/index.html
url = new URL("http://www.jinqiliang.me/index.html");
System.out.println(url);

// http://www.jinqiliang.me/hello1
url = new URL("http", "www.jinqiliang.com", "/hello1");
System.out.println(url);

// http://www.jinqiliang.me:9527/hello2
url = new URL("http", "www.jinqiliang.com", 9527, "/hello2");
System.out.println(url);

// http://www.jinqiliang.me/about
url = new URL(url, "about");
System.out.println(url);
```

除了以上方法之外还有一些方法可以获取URL。

java classLoader.getResource() 可以返回URL

```java
// 静态方法
url = ClassLoader.getSystemResource("hei.properties");
System.out.println(url);

// 静态方法
Enumeration<URL> urls =  ClassLoader.getSystemResources("hei.properties");
while(urls.hasMoreElements()) {
	url = urls.nextElement();
	System.out.println(url);
}

// 实例方法
url = Main.class.getClassLoader().getResource("hei.properties");
System.out.println(url);
```

# get方法

URL类提供了一些get方法来获取url中的特定部分

- public String getPath()
返回URL路径部分。
- public String getQuery()
返回URL查询部分。
- public String getAuthority()
获取此 URL 的授权部分。
- public int getPort()
返回URL端口部分
- public int getDefaultPort()
返回协议的默认端口号。
- public String getProtocol()
返回URL的协议
- public String getHost()
返回URL的主机
- public String getFile()
返回URL文件名部分
- public String getRef()
获取此 URL 的锚点（也称为"引用"）

# 获取数据

URL定义了如何定位一个资源，因此可以通过url来获取资源。

```java
// 1
// 为url打开一个socket，并返回URLConnnect对象
// 如果打开失败，抛出异常IOException
// 这个对象还包括了连接的信息，如协议等
public URLConnection openConnection() 
		throws java.io.IOException
// 2
public URLConnection openConnection(Proxy proxy)
        throws java.io.IOException

// 3
// 调用该方法客户端和服务器会进行连接
// 返回的InputStream对象可以读取资源，但不包括连接信息
public final InputStream openStream() 
		throws java.io.IOException {
	return openConnection().getInputStream();
}

// 4
// 这个方法直接获取url的数据（openStream需要自己去获取数据）
public final Object getContent() throws java.io.IOException {
	return openConnection().getContent();
}
// 5
public final Object getContent(Class[] classes)
    	throws java.io.IOException {
	return openConnection().getContent(classes);
}
```
从以上5个方法最重要的是**URLConnection openConnection()**

方法3， 4， 5都是通过1中返回的URLConnection来进一步获取结果的。

# toString 和 toExternalForm

这两个方法其实是一样的，toString()内部调用的是toExternalForm()。

# toURI
可以将一个URL转化为一个URI，URI将在java.net.URI中再介绍。

```java
public URI toURI() throws URISyntaxException
```

# 参考：

1. 维基百科：https://en.wikipedia.org/wiki/Uniform_Resource_Identifier
2. 《java网络编程》（第四版）


