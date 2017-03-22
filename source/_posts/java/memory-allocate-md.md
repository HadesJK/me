---
title: java 对象内存计算
date: 2017-03-22 10:24:52
categories:
- java
---

3月19号去见了一个之前实习公司的同事，当他问我disruptor的时候，我突然懵逼了。
这个还是之前我和他布道的，真是惭愧，我和他的差距体现在，他听到这个东西后，回去体验他，而我仅仅是听说过而已。
当然作为一个非资深程序猿攻城狮，心服口难服(劳资不服)，因此回来后，马上拉了disruptor的源码。
悲剧是一连串发生的，看到第一个类是  RingBuffer，里面有个RingBufferPad，7个字段，其它什么都没有。
稍微搜了一下，发现和前几天我自己吹逼的cpu L1，L2，L3缓存有关，和jvm有关，因此才有了这篇文章。

这篇文章讲的是如何计算一个java对象的内存，请听小仙细细道来。

# java 内存计算

java.lang.instrument.Instrumentation

# java 对象的内存布局

这里抄一下 周志明 兄弟的《深入理解java虚拟机 jvm高级特性与最佳实践》（俺的jvm入门书籍）

** http://www.infoq.com/cn/articles/jvm-hotspot **

```
HotSpot虚拟机中，对象在内存中存储的布局可以分为三块区域：对象头（Header）、实例数据（Instance Data）和对齐填充（Padding）。

HotSpot虚拟机的对象头包括两部分信息，
第一部分用于存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等等，
这部分数据的长度在32位和64位的虚拟机（暂不考虑开启压缩指针的场景）中分别为32个和64个Bits，官方称它为“Mark Word”。
对象头的另外一部分是类型指针，即是对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。
并不是所有的虚拟机实现都必须在对象数据上保留类型指针，换句话说查找对象的元数据信息并不一定要经过对象本身。

如果对象是一个Java数组，那在对象头中还必须有一块用于记录数组长度的数据，
因为虚拟机可以通过普通Java对象的元数据信息确定Java对象的大小，但是从数组的元数据中无法确定数组的大小。

第三部分对齐填充并不是必然存在的，也没有特别的含义，它仅仅起着占位符的作用。
由于HotSpot VM的自动内存管理系统要求对象起始地址必须是8字节的整数倍，换句话说就是对象的大小必须是8字节的整数倍。
对象头部分正好似8字节的倍数（1倍或者2倍），因此当对象实例数据部分没有对齐的话，就需要通过对齐填充来补全。
```

那么根据这个理论知识，马上开始实践。

之前俺对于数据库的原始对象，使用 int 和 Integer 很好奇，有什么规范可以遵循，以这个问题为由，查了一些知识
其中讲到一点：Integer 使用的内存是 int 的 4 倍！

java中，int 是 4 字节的，而 Integer 呢？ 在32位虚拟机下，8(对象头)+4(域)+4(对齐填充)=16

然后使用代码跑了一下，果然是 16 ：

![](/images/java/对象内存计算/ErrorIntegerSize.png)

果然，哈哈哈！

辣嚒，java.lang.Double呢？应该是 8(对象头)+8(域)=16，结果：

![](/images/java/对象内存计算/ErrorDoubleSize.png)

雪崩啊，竟然是24，为毛！

# 32bit & 64bit

又是一顿搜索，再结合周老师的理论，发现自己蠢了，理论上说32bit的和64bit的对象头是**不一样**的
查自己的机器：
![](/images/java/对象内存计算/jvm.png)

既然是64bit的虚拟机，那么计算方式是：16 + 8 = 24，咦，没问题！没问题？

那么再来看 java.lang.Integer
内存应该是 16(对象头)+4(域)+4(对齐填充)=24，不是之前的结果

完蛋了，问题粗在哪里呢。

再查查查

原来，64bit的虚拟机默认开启了指针压缩（周老师的理论也提到了，“暂不考虑开启压缩指针的场景”）
这个时候，对象头中的类型指针是4字节的

OK，那么重新计算64bit下的java.lang.Integer和java.lang.Double的内存：

**默认情况：开始指针压缩**

-XX:+UseCompressedOops

java.lang.Integer 12 + 4 + 0 = 16

java.lang.Double  12 + 8 + 4 = 24

![](/images/java/对象内存计算/XX+UseCompressedOops.png)

**不开启指针压缩**

-XX:-UseCompressedOops

java.lang.Integer 16 + 4 + 4 = 24

java.lang.Double  16 + 8 + 0 = 24

![](/images/java/对象内存计算/XX-UseCompressedOops.png)

# 计算java对象内存

理论知识：
周老师提到了

接下来实例数据部分是对象真正存储的有效信息，
也既是我们在程序代码里面所定义的各种类型的字段内容，
无论是从父类继承下来的，
还是在子类中定义的都需要记录起来。
这部分的存储顺序会受到虚拟机分配策略参数（FieldsAllocationStyle）和字段在Java源码中定义顺序的影响。
HotSpot虚拟机默认的分配策略为
1. longs/doubles、
2. ints、
3. shorts/chars、
4. bytes/booleans、
5. oops（Ordinary Object Pointers），
从分配策略中可以看出，相同宽度的字段总是被分配到一起。
**在满足这个前提条件的情况下，在父类中定义的变量会出现在子类之前。**
**如果 CompactFields 参数值为true（默认为true），那子类之中较窄的变量也可能会插入到父类变量的空隙之中**


前面只是提了很简单的情况，那么来计算下其它情况

首先当实例数据很复杂的时候【其实我要开始挖坑了】

3个类 A, B, C

```java
public class A {
    protected long a1;
    protected int a2;
    protected short a3;
    protected byte a4;
}
```

```java
public class B extends A {
    private int b1;
    private byte b2;
}
```

```java
public class C {
    private long a1;
    private int a2;
    private short a3;
    private byte a4;
    private int b1;
    byte b2;
}
```

首先手工计算一下：
A:	12 + [8 + 4 + 2 + 1] + padding = 32字节
B:	12 +  [8 + 4 + 2 + 1] + 4 + 1 + padding = 32字节
C: 12 + 8 + 4 + 2 + 1 + 4 + 1 + padding = 32字节

然后看程序的输出：
![](/images/java/对象内存计算/output1.png)

什么！B的内存怎么不对？

这里还有一个坑，周老师讲的比较含糊，
虽然64位机器开始了指针压缩和CompactFields=true，但是只有相等长度的数据可以被放一起的，
否则按操作系统的要求会发生4字节对齐（我记得C语言就是这个样子的）

这样子分析，再看B的内存计算
head + a2 + a1 + a3 + padding/4 + a4 + b2 + padding/4 + b1 + padding/8
B: 12 + [4 + 8 + 2 + 2(padding)  + 1 + 1 + 2(padding) + 4] + 4(padding) = 40字节

不行吗？那么通过java中的Unsafe类来看下各个字段的位置偏移信息：

```java
long	a1: 16
int		a2: 12
short	a3: 24
byte	a4: 26
int		b1: 28
byte	b2: 32
```

**还是对不上！**
**网上搜到这样子一段话：(https://www.liaohuqiu.net/cn/posts/caculate-object-size-in-java/)**

```
在32bit的虚拟机上：
如果子类首个成员变量是 long 或者 double 等 8 字节数据类型，
而父类结束时没有 8 位对齐。
会把子类的小于 8 字节的实例成员先排列，
直到能 8 字节对齐。
```

根据这一点，就解释了上面的例子了，哈哈哈，也是醉了。













