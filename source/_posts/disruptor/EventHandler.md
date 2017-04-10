---
title: Disruptor 源码分析四：EventHandler
date: 2017-04-11 21:39:04
categories:
- disruptor
---

# EventHandler

消费者远比生产者复杂

首先介绍消费者必须实现的一个接口： EventHandler
消费者对数据的处理是在 EventHandler#onEvent 中实现的

```java
public interface EventHandler<T> {
    /**
     * Called when a publisher has published an event to the {@link RingBuffer}
     *
     * @param event      published to the {@link RingBuffer}
     * @param sequence   of the event being processed
     * @param endOfBatch flag to indicate if this is the last event in a batch from the {@link RingBuffer}
     * @throws Exception if the EventHandler would like the exception handled further up the chain.
     */
    void onEvent(T event, long sequence, boolean endOfBatch) throws Exception;
}
```


这个接口的实现类会被分装成一个 BatchEventProcessor<T> 对象，而 BatchEventProcessor 又实现自接口 EventProcessor

![EventProcessor](/images/disruptor/EventProcessor.png)

# EventProcessor

```java
public interface EventProcessor extends Runnable {

    /**
     * 获取消费者自身的 Sequence
     * 消费者的 Sequence 用来记录消费者最大处理完的序号
     *
     * @return
     */
    Sequence getSequence();

    /**
     * 当调用这个方法后，消费者在处理完当前事件后，将会停止消费 RingBuffer 中的事件
     */
    void halt();

    /**
     * 指示消费者是否在运行状态
     *
     * @return
     */
    boolean isRunning();
}
```


# BatchEventProcessor

这是个批处理的对象，即消费者可以批量处理 RingBuffer 中的事件

因为消费者实现了 EventHandler 接口，但是这个接口的实现类是用户提供的，
而消费者是需要记录自身消费 RingBuffer 位置信息的数据 Sequence 
因此可知 BatchEventProcessor 中需要对这里进行处理

这个类只需要关注下几个字段和一个方法

```java
private final AtomicBoolean running = new AtomicBoolean(false);
// 异常处理器
private ExceptionHandler<? super T> exceptionHandler = new FatalExceptionHandler();
// 消费者的数据源，即 RingBuffer
private final DataProvider<T> dataProvider;
// 栅栏，消费者和 RingBuffer 交互的工具
private final SequenceBarrier sequenceBarrier;
// 消费者处理器
private final EventHandler<? super T> eventHandler;
// 消费者自己的序列(记录处理完成的最大序列，即消费到 RingBuffer 的位置)
private final Sequence sequence = new Sequence(Sequencer.INITIAL_CURSOR_VALUE);
private final TimeoutHandler timeoutHandler;
```

```java
@Override
public void run() {
    if (!running.compareAndSet(false, true)) {
        throw new IllegalStateException("Thread is already running");
    }
    // 清除前置序号关卡的通知状态
    sequenceBarrier.clearAlert();
    notifyStart();
    T event = null;
    // sequence.get()标示当前已经处理的序号
    // Sequence 初始值是 -1
    // 因此消费者处理的序号从 0 开始递增
    long nextSequence = sequence.get() + 1L;
    try {
        while (true) {
            try {
                // 从它的前置序号关卡获取下一个可处理的事件序号。
                // 如果这个事件处理器不依赖于其他的事件处理器，则前置关卡就是生产者序号；
                // 如果这个事件处理器依赖于1个或多个事件处理器，那么这个前置关卡就是这些前置事中最慢的一个。
                // 通过这样，可以确保事件处理器不会超前处理地事件。
                final long availableSequence = sequenceBarrier.waitFor(nextSequence);
                // 处理一批事件
                while (nextSequence <= availableSequence) {
                    event = dataProvider.get(nextSequence);
                    eventHandler.onEvent(event, nextSequence, nextSequenceavailableSequence);
                    nextSequence++;
                }
                // 处理完之后，设置自己最后处理完成的事件序号
                sequence.set(availableSequence);
            } catch (final TimeoutException e) {
                // 获取事件序号超时处理
                notifyTimeout(sequence.get());
            } catch (final AlertException ex) {
                // 处理通知事件；检测是否要停止，如果非则继续处理事件
                if (!running.get()) {
                    break;
                }
            } catch (final Throwable ex) {
                // 其他异常，用异常处理器处理；然后继续处理下一个事件
                exceptionHandler.handleEventException(ex, nextSequence, event);
                sequence.set(nextSequence);
                nextSequence++;
            }
        }
    } finally {
        notifyShutdown();
        running.set(false);
    }
}
```



