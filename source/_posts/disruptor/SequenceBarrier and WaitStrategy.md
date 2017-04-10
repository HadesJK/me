---
title: Disruptor 源码分析三：SequenceBarrier 和 WaitStrategy
date: 2017-04-10 10:12:35
categories:
- disruptor
---

之前分析了生产者，disruptor 对消费者的封装比较复杂

首先需要介绍 SequenceBarrier 和 WaitStrategy

SequenceBarrier 是给消费者独用的

WaitStrategy 有2个方法，分别给生产者和消费者使用

# SequenceBarrier

![SequenceBarrier](/images/disruptor/SequenceBarrier.png)

SequenceBarrier 是给消费者使用的，
通过 SequenceBarrier ，消费者可以实现一些复杂的序号操作

当前他只有一个实现类 ProcessingSequenceBarrier

```java
final class ProcessingSequenceBarrier implements SequenceBarrier {

    // 等待策略(下面分析)
    private final WaitStrategy waitStrategy;

    // 前置依赖序号
    private final Sequence dependentSequence;

    private volatile boolean alerted = false;

    // RingBuffer 的序号
    private final Sequence cursorSequence;

    // 生产者的序号控制器
    private final Sequencer sequencer;

    /**
     * @param sequencer          生产者序号控制器
     * @param waitStrategy       等待策略
     * @param cursorSequence     RingBuffer 的序号
     * @param dependentSequences 依赖的前置 Sequence(前置消费者的 Sequence)
     */
    public ProcessingSequenceBarrier(final Sequencer sequencer, final WaitStrategy waitStrategy,
                                     final Sequence cursorSequence, final Sequence[] dependentSequences) {
        this.sequencer = sequencer;
        this.waitStrategy = waitStrategy;
        this.cursorSequence = cursorSequence;
        if (0 == dependentSequences.length) {
            // 如果消费者不依赖于任何前置消费者，那么dependentSequence也指向生产者的序号
            dependentSequence = cursorSequence;
        } else {
            // 如果消费者依赖其它前置消费者，则记录这些前置消费者的序号
            // 这种情况下，当前消费者需要等待前置消费者处理完一个事件后，才能对该事件进行处理
            // 比如序号16的事件，对于的业务是 A 对 B 转账：
            // 1. 先减 A 的钱(消费者1) 2. 再加 B 的钱(消费者2)
            // 这种情况下可以设置为 B 的 dependentSequence 设置成 A 的序号
            dependentSequence = new FixedSequenceGroup(dependentSequences);
        }
    }

    /**
     * 该方法不保证总是返回未处理的序号；
     * 如果有更多的可处理序号时，返回的序号也可能是超过指定序号的。
     */
    @Override
    public long waitFor(final long sequence) throws AlertException, InterruptedException, TimeoutException {
        // 检查消费者是否被终止
        checkAlert();
        // 通过等待策略来获取可处理事件序号
        // 4个参数分别是：期望获取的序号，RingBuffer 的当前序号，依赖的前置序号，当前 SequenceBarrier
        long availableSequence = waitStrategy.waitFor(sequence, cursorSequence, dependentSequence, this);
        // 这个方法不保证总是返回可处理的序号
        if (availableSequence < sequence) {
            return availableSequence;
        }
        // 再通过生产者序号控制器返回最大的可处理序号
        return sequencer.getHighestPublishedSequence(sequence, availableSequence);
    }

    // 返回一个当前可以读取的序号值
    @Override
    public long getCursor() {
        return dependentSequence.get();
    }

    @Override
    public boolean isAlerted() {
        return alerted;
    }

    @Override
    public void alert() {
        alerted = true;
        waitStrategy.signalAllWhenBlocking();
    }

    @Override
    public void clearAlert() {
        alerted = false;
    }

    @Override
    public void checkAlert() throws AlertException {
        if (alerted) {
            throw AlertException.INSTANCE;
        }
    }
}
```

# WaitStrategy

WaitStrategy 放在这里讲可能不太合适，因为他的一个方法别生产者使用，一个方法被消费者使用

这种情况下，我觉得作者使用可以考虑拆分下接口，
比如 接口 ProducerWaitStrategy，里面有 signalAllWhenBlocking
接口 ConsumerWaitStrategy，里面有方法 waitFor
然后接口 WaitStrategy 实现接口 ProducerWaitStrategy 和 ConsumerWaitStrategy
这样子生产者和消费者使用的时候拿到的都是各自的接口，也比较好理解吧

下面分析这个接口

```java
/**
 * 等待策略
 * 消费者使用 waitFor() 方法获取 RingBuffer 中可用的序号
 * 生产者使用 signalAllWhenBlocking() 通知阻塞的消费者获取事件进行处理
 */
public interface WaitStrategy {
    /**
     * 消费者获取 RingBuffer 当前可用的序号值
     *
     * @param sequence          期望等待的序号值
     * @param cursor            RingBuffer 的 cursor，即 RingBuffer 的 Sequence
     * @param dependentSequence 依赖的的序号，即消费者可以处理的序号必须 小于等于 dependentSequence
     * @param barrier           栅栏，消费者通过栅栏获取序号
     * @return 返回的 availableSequence 可能大于请求的 sequence
     * @throws AlertException       if the status of the Disruptor has changed.
     * @throws InterruptedException if the thread is interrupted.
     * @throws TimeoutException
     */
    long waitFor(long sequence, Sequence cursor, Sequence dependentSequence, SequenceBarrier barrier)
            throws AlertException, InterruptedException, TimeoutException;

    /**
     * 生产者通知消费者有数据可用
     */
    void signalAllWhenBlocking();
}
```

WaitStrategy 接口有很多实现类，这里先分析一个典型的

# BlockingWaitStrategy

这个通过名字可以看出，当消费者获取不到数据时就会阻塞

BlockingWaitStrategy 内部通过 ReentrantLock 和条件队列 Condition 实现

waitFor 做了2件事情：
1. 等待 RingBuffer 事件槽可用(Producer 已经发布事件)
2. 等待前置依赖处理器处理完成(依赖别的消费者先处理完)


```java
public final class BlockingWaitStrategy implements WaitStrategy {
    private final Lock lock = new ReentrantLock();
    private final Condition processorNotifyCondition = lock.newCondition();

    @Override
    public long waitFor(long sequence,  // 消费者请求的序号值
                        Sequence cursorSequence,    // RingBuffer 的序号
                        Sequence dependentSequence, // 消费者依赖的序号
                        SequenceBarrier barrier)    // 栅栏
            throws AlertException, InterruptedException {
        long availableSequence;
        if (cursorSequence.get() < sequence) {
            lock.lock();
            try {
                // 无限等待直到 RingBuffer 有数据可用
                // 可以通过 barrier.checkAlert() 异常退出
                while (cursorSequence.get() < sequence) {
                    barrier.checkAlert();
                    processorNotifyCondition.await();
                }
            } finally {
                lock.unlock();
            }
        }
        // 这个 while 确保消费者依赖的序号已经在处理了，即实现先后消费
        // 同样支持 barrier.checkAlert() 异常退出
        // 这里 是 < 符号
        // 消费者只有将数据处理完了之后才会设置自身的 Sequence
        // 也就是说 通过消费者的 Sequence.get() 获取的表示消费者已经处理好的最大序号
        while ((availableSequence = dependentSequence.get()) < sequence) {
            barrier.checkAlert();
        }

        return availableSequence;
    }

    @Override
    public void signalAllWhenBlocking() {
        lock.lock();
        try {
            processorNotifyCondition.signalAll();
        } finally {
            lock.unlock();
        }
    }

    @Override
    public String toString() {
        return "BlockingWaitStrategy{" +
                "processorNotifyCondition=" + processorNotifyCondition +
                '}';
    }
}
```

# BusySpinWaitStrategy

从命名上看，以为是 CPU 忙转
看了源码，发现并不能确切的表达这里意思

```java
public final class BusySpinWaitStrategy implements WaitStrategy {
    @Override
    public long waitFor(
            final long sequence, Sequence cursor, final Sequence dependentSequence, final SequenceBarrier barrier)
            throws AlertException, InterruptedException {
        long availableSequence;
        // 等待前置处理器处理完成
        while ((availableSequence = dependentSequence.get()) < sequence) {
            barrier.checkAlert();
        }
        return availableSequence;
    }

    @Override
    public void signalAllWhenBlocking() {
    }
}
```

因为里面没有阻塞，所以 signalAllWhenBlocking 方法体是空的
waitFor 方法只做了一件事情，等待前置处理器处理完成

1. 比如 前置处理器 B 处理完序号19，而 RingBuffer 目前的序号(Curator)也是20
(可以推断这时候前置处理器应该可能在处理序号20，也可能在等待它的前置处理器)

当前处理器 A 去请求19，获取到19可处理，进行处理

2. 然后当前处理器 A 去请求20，
如果前置处理器 B 未完成对 20 的槽的事件处理完，当前处理器 A 通过waitFor方法，一直在while循环

3. 过了一段时间，当前置处理器 B 处理完了，A 走出 while

4. 然后 A 处理 20 的槽

5. 再此进入 waitFor，请求 21，这个时候 while 判断 true，一直在这里空转

因此 BusySpin 的含义得到了体现

分析: BusySpinWaitStrategy#waitFor 返回的序号肯定是未处理的？
(前提是消费者正确的请求，即不能处理完序号 n，然后继续请求序号n)

答案：不是！
在多生产者的情况下，通过 Sequencer.getHighestPublishedSequence 方法，
返回的是最小的可用序号，看案例：
多生产者中，当生产者1获取到序号 13，生产者2获取到14；生产者1没发布，生产者2发布，会导致获取的可用序号为12，而sequence为13！
消费者通过 WaitStrategy 获得的序号是 13，但是再通过 MultiProducerSequencer.getHighestPublishedSequence 返回的是12
这时候如果消费者处理完12，请求下一个13时，返回的可用序号仍然是12！
这就是为毛消费者获取到序号后，需要自行判断(BatchEventProcessor 中分析)
