---
title: Disruptor 源码分析一：Sequence 
date: 2017-03-31 21:48:58
categories:
- disruptor
---

#  Sequence

看懂这个类需要一下几个基础知识：
1. CPU 缓存， 伪共享
2. java 的 Unsafe 类


# CPU 缓存

通常CPU不直接访问内存，而是通过每一级缓存访问。
最上的是L1缓存，访问速度最快，当然存储容量也最小。
其次是L2缓存，速度慢一些，存储容量也大一些。
然后是L3缓存，速度再慢一些，存储容量再大一些。
最后才是内存，速度最慢。
当CPU从L1中获取不到，就会去L2中取，如果命中，则加载到L1中。
但是加载过程是整一个缓存行替换，通常的CPU的缓存行占用64字节，也就是说L1不命中时，会直接替换64个字节。

伪共享（False Sharing）的问题时，多线程之间各自读取相互独立的几个变量，
但是如果在内存中，这几个相互独立的变量是相邻的，那么每个线程在修改自己的变量时，
可以会时另外一个线程的变量失效，需要从内存中获取，从而互相影响性能。
虽然这几个变量互相独立，

这个类是 Disruptor 的核心，主要就一个字段：value，前后都用了56个字节填充（7个long），主要考虑到缓存行填充来消除伪共享的问题。
前后都用56字节填充的好处是，只要读取/写入value这个字段，不会因为其它数据的读写而影响这个字段需要失效。

# sun.misc.Unsafe

这是 jdk 提供的一个不安全类，程序员很少会去使用这个类。

从源码中看到很多 native 方法，底层由虚拟机实现。

这里主要分析用到的几个方法。

获取 Unsafe 实例

虽然 Unsafe 提供了一个静态方法 getUnsafe()，然而在代码中无法直接使用这个方法，会抛出异常 SecurityException

但是 通过 jdk 的反射方法却可以获取

```java
// 获取 Unsafe 的私有字段 theUnsafe
Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
// 取得私有字段 theUnsafe 的访问权限
theUnsafe.setAccessible(true);
return (Unsafe) theUnsafe.get(null);
```

获取类中一个字段的偏移量
```java
public native long objectFieldOffset(Field var1);
```

原子的写long，但是值不立即刷入内存
```
public native void putOrderedLong(Object var1, long var2, long var4);
```

long 字段原子写，值立即刷入内存
```
public native void putLongVolatile(Object var1, long var2, long var4);
```

# Sequence 源码分析

Sequence 是 Disruptor 的核心，这里称为序号
生产者利用序号获取 RingBuffer 当前可用的位置
消费者利用序号获取 RingBuffer 当前可以消费的最大位置，同时消费者自身需要记录当前已经消费到的位置

首先使用前后56字节填充关键字段 
**protected volatile long value;**，
即缓存行填充来避免伪共享的问题，
典型的空间获取时间。

```java
class LhsPadding {
    protected long p1, p2, p3, p4, p5, p6, p7;
}

class Value extends LhsPadding {
    protected volatile long value;
}

class RhsPadding extends Value {
    protected long p9, p10, p11, p12, p13, p14, p15;
}
```

然后是计算 value 在类中的偏移量
```java
// 初始值
static final long INITIAL_VALUE = -1L;
// Unsafe 对象
private static final Unsafe UNSAFE;
// 偏移量
private static final long VALUE_OFFS
static {
    UNSAFE = Util.getUnsafe();
    try {
    	// 计算 value 字段在类 Value 中的偏移量
    	// 按 java 的规则，父类的字段在前面，因此这个也是 value 字段在 Sequence 中的位置
        VALUE_OFFSET = UNSAFE.objectFieldOffset(Value.class.getDeclaredField("value"));
    } catch (final Exception e) {
        throw new RuntimeException(e);
    }
}

public Sequence() {
    this(INITIAL_VALUE);
}

// 初始化，但是这个值不立即刷入内存中，也就是说其它线程不一定立即可见
public Sequence(final long initialValue) {
    UNSAFE.putOrderedLong(this, VALUE_OFFSET, initialValue);
}

// 读取 volatile 变量
public long get() {
    return value;
}

// 原子的写一个 long 变量，但是不立即刷入内存中
public void set(final long value) {
    UNSAFE.putOrderedLong(this, VALUE_OFFSET, value);
}

// 写 volatile 变量
public void setVolatile(final long value) {
    UNSAFE.putLongVolatile(this, VALUE_OFFSET, value);
}

// 对 value 值的 cas 操作
public boolean compareAndSet(final long expectedValue, final long newValue) {
    return UNSAFE.compareAndSwapLong(this, VALUE_OFFSET, expectedValue, newValue);
}

// 类似 AtomicLong.incrementAndGet 操作
public long incrementAndGet() {
    return addAndGet(1L);
}

// 类似 AtomicLong.addAndGet 操作
public long addAndGet(final long increment) {
    long currentValue;
    long newValue;
    do {
        currentValue = get();
        newValue = currentValue + increment;
    }
    while (!compareAndSet(currentValue, newValue));
    return newValue;
}

@Override
public String toString() {
    return Long.toString(get());
}
```

Sequence 有2个子类 FixedSequenceGroup 和 SequenceGroup
FixedSequenceGroup 从命名上看出是一个固定数量的 Sequence 组，内部利用数组实现
SequenceGroup 则是一个长度可变的 Sequence 组，可能很容易想到的是 List 实现，然而作者仍然采用数组实现。

# FixedSequenceGroup

这个类比较简单

```java
public final class FixedSequenceGroup extends Sequence {
    private final Sequence[] sequences;

    // 初始化 Sequence 数组，一旦初始化，这个数组变不可变化，但数组持有的是引用，引用指向的内容是可变的
    public FixedSequenceGroup(Sequence[] sequences) {
        this.sequences = Arrays.copyOf(sequences, sequences.length);
    }

    // 获取 Sequence 组的最小序号值
    @Override
    public long get() {
        return Util.getMinimumSequence(sequences);
    }

    @Override
    public String toString() {
        return Arrays.toString(sequences);
    }

    // ====== 因为是不可变 Sequence 组，对这个组的写操作都不支持 ======
    // ====== 感觉这个类没啥用 ======
    @Override
    public void set(long value) {
        throw new UnsupportedOperationException();
    }

    @Override
    public boolean compareAndSet(long expectedValue, long newValue) {
        throw new UnsupportedOperationException();
    }

    @Override
    public long incrementAndGet() {
        throw new UnsupportedOperationException();
    }

    @Override
    public long addAndGet(long increment) {
        throw new UnsupportedOperationException();
    }
}
```

# SequenceGroups

这里先介绍工具 SequenceGroups
不得不吐槽一下作者，类的命名真是没考虑中国攻城狮的感受，这个类可以取成 SequenceGroupUtil 多好

```java
class SequenceGroups {
    // 添加一个序号到序号组，支持并发添加
    // holder 持有序号组的对象，即 SequenceGroup，可能以后还有其它对象
    // updater 更新器 AtomicReferenceFieldUpdater
    // cursor 当前的序号位置，即 SequenceGroup 的 value 字段（SequenceGroup 也是 Sequence 对象）
    // sequencesToAdd 需要添加的 Sequence 数组
    static <T> void addSequences(final T holder, 
                                final AtomicReferenceFieldUpdater<T, Sequence[]> updater, 
                                final Cursored cursor, 
                                final Sequence... sequencesToAdd) {
        long cursorSequence;
        Sequence[] updatedSequences;
        Sequence[] currentSequences;
        do {
            // 获取 SequenceGroup 持有的 Sequence 数组
            currentSequences = updater.get(holder);
            updatedSequences = copyOf(currentSequences, currentSequences.length + sequencesToAdd.length);
            cursorSequence = cursor.getCursor();
            // 将需要添加的 Sequence 数组添加到末尾，并初始化 value 为当前 Sequence 的value
            int index = currentSequences.length;
            for (Sequence sequence : sequencesToAdd) {
                sequence.set(cursorSequence);
                updatedSequences[index++] = sequence;
            }
        } while (!updater.compareAndSet(holder, currentSequences, updatedSequences));

        // TODO: 添加后，为什么还需要重新设置被添加的 Sequence 数组的序列号值？
        cursorSequence = cursor.getCursor();
        for (Sequence sequence : sequencesToAdd) {
            sequence.set(cursorSequence);
        }
    }

    static <T> boolean removeSequence(
            final T holder,
            final AtomicReferenceFieldUpdater<T, Sequence[]> sequenceUpdater,
            final Sequence sequence) {
        int numToRemove;
        Sequence[] oldSequences;
        Sequence[] newSequences;

        do {
            oldSequences = sequenceUpdater.get(holder);
            // 需要移除出序号组的数量
            numToRemove = countMatching(oldSequences, sequence);
            if (0 == numToRemove) {
                break;
            }
            final int oldSize = oldSequences.length;
            newSequences = new Sequence[oldSize - numToRemove];

            for (int i = 0, pos = 0; i < oldSize; i++) {
                final Sequence testSequence = oldSequences[i];
                if (sequence != testSequence) {
                    // 如果引用不等，表示不需要移除
                    newSequences[pos++] = testSequence;
                }
            }
        } while (!sequenceUpdater.compareAndSet(holder, oldSequences, newSequences));
        return numToRemove != 0;
    }

    // 计算出需要移除的数量
    private static <T> int countMatching(T[] values, final T toMatch) {
        int numToRemove = 0;
        for (T value : values) {
            if (value == toMatch) {
                numToRemove++;
            }
        }
        return numToRemove;
    }
}
```

# SequenceGroup

SequenceGroup 是 Sequence 的子类，这里称为 序号组 。

```java
public final class SequenceGroup extends Sequence {
    private static final AtomicReferenceFieldUpdater<SequenceGroup, Sequence[]> SEQUENCE_UPDATER =
            AtomicReferenceFieldUpdater.newUpdater(SequenceGroup.class, Sequence[].class, "sequences");
    // 这个是 SequenceGroup，初始值为长度等于 0 的数组
    private volatile Sequence[] sequences = new Sequence[0];

    public SequenceGroup() {
        super(-1);
    }

    // 获取 SequenceGroup 中的最小的序号值，无并发控制
    @Override
    public long get() {
        return Util.getMinimumSequence(sequences);
    }

    // 设置 SequenceGroup 中的每个序号为 value 值，无并发控制
    @Override
    public void set(final long value) {
        final Sequence[] sequences = this.sequences;
        for (int i = 0, size = sequences.length; i < size; i++) {
            sequences[i].set(value);
        }
    }

    // 原子的添加一个序号到序号组，cas实现乐观锁
    // 这个方法支持并发，但不能在运行中添加，否则 
    public void add(final Sequence sequence) {
        Sequence[] oldSequences;
        Sequence[] newSequences;
        do {
            oldSequences = sequences;
            final int oldSize = oldSequences.length;
            newSequences = new Sequence[oldSize + 1];
            System.arraycopy(oldSequences, 0, newSequences, 0, oldSize);
            newSequences[oldSize] = sequence;
        }
        while (!SEQUENCE_UPDATER.compareAndSet(this, oldSequences, newSequences));
    }

    // 支持动态得从序号组中移除一个序号
    public boolean remove(final Sequence sequence) {
        return SequenceGroups.removeSequence(this, SEQUENCE_UPDATER, sequence);
    }

    public int size() {
        return sequences.length;
    }

    // 支持动态得从序号组中添加一个序号
    public void addWhileRunning(Cursored cursored, Sequence sequence) {
        SequenceGroups.addSequences(this, SEQUENCE_UPDATER, cursored, sequence);
    }
}
```


