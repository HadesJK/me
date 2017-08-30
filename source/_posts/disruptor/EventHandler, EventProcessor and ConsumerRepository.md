---
title: Disruptor 源码分析四：EventHandler、EventProcessor 和 ConsumerRepository
date: 2017-04-11 21:39:04
categories:
- disruptor
---

# EventHandler

消费者远比生产者复杂

之前讲的 SequenceBarrier 和 WaitStrategy 都是为消费者服务的

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
                // 如果这个事件处理器依赖于1个或多个事件处理器，那么这个前置关卡就是这些前置事件处理器中最慢的一个。
                // 通过这样，可以确保事件处理器不会超前处理地事件。
                final long availableSequence = sequenceBarrier.waitFor(nextSequence);
                // 处理一批事件==>这里需要判断返回到结果，因为 sequenceBarrier.waitFor 返回的结果不一定可用(可能是旧的序号值)
                while (nextSequence <= availableSequence) {
                    event = dataProvider.get(nextSequence);
                    eventHandler.onEvent(event, nextSequence, nextSequence == availableSequence);
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

# ConsumerInfo

![ConsumerInfo](/images/disruptor/ConsumerInfo.png)

从类的命名上来看，这个类保存了消费者的一些信息

它的实现类是 EventProcessorInfo

EventProcessorInfo 的实现比较简单，只需要了解它的几个字段就可以了

```java
// EventHandler 会被封装成 EventProcessor 的实现类
private final EventProcessor eventprocessor;
// EventHandler 是消费者需要实现的接口
private final EventHandler<? super T> handler;
// 消费者的栅栏
private final SequenceBarrier barrier;
// 是否是 生产者关注的 链 中的最有一链
private boolean endOfChain = true;
```


# ConsumerRepository

ConsumerRepository 是消费者的集合
消费者实现 EventHandler 接口，然后被封装成一个 BatchEventProcessor
而 BatchEventProcessor 又被放入消费者集合 ConsumerRepository 中
通时 BatchEventProcessor 又被构造成一个 EventProcessorInfo，也放入 ConsumerRepository

这里看到 ConsumerRepository, ConsumerInfo 等这些封装有点过分了，太重了，导致代码阅读体验差

其实消费者真正的信息有以下几个：
1. 消费者实现 EventHandler 接口
2. 消费者栅栏 SequenceBarrier
消费者依赖的前置消费者列表 Sequence[] 存在栅栏中
1 和 2 又构成 BatchEventProcessor

上面提到的所有接口都是通过 1 和 2 的组合构造起来

# Disruptor 添加消费者的方法

现在可以来看 Disruptor 的几个添加消费者的方法了

最简单的添加消费者方法：

```java
// 消费者类
public class LongEventHandler implements EventHandler<LongEvent> {

    private final int id;

    public LongEventHandler(int id) {
        this.id = id;
    }

    public void onEvent(LongEvent event, long sequence, boolean endOfBatch) {
        System.out.println("Event[" + id + "]: " + event + "， sequence: " + sequence + ", endOfBatch: " + endOfBatch);
    }
}
```

```
// Main.main 中添加消费者
int id = 9527;
disruptor.handleEventsWith(new LongEventHandler(id));
```

上面的这个方法调用

```java
// Disruptor 类
// 这种添加方式，消费者不依赖其它前置消费者，因此只要等到 RingBuffer 中可用的序号比自身大时就可以处理这些槽
public EventHandlerGroup<T> handleEventsWith(final EventHandler<? super T>... handlers) {
    return createEventProcessors(new Sequence[0], handlers);
}

// barrierSequences 其实是 DependencySequences，
// 即依赖的前置消费者列表，如果没有前置消费者，则传一个长度为0的数组
EventHandlerGroup<T> createEventProcessors(final Sequence[] barrierSequences, final EventHandler<? super T>[] eventHandlers) {
    // 只能在 Disruptor 未启动之前添加消费者
    checkNotStarted();
    // 为每个消费这创建一个消费序列，消费者通过 Sequence 来记录自己的消费位置，
    // 同时生产者会跟踪所有消费者的消费位置
    final Sequence[] processorSequences = new Sequence[eventHandlers.length];
    // eventHandlers 都使用同一个 SequenceBarrier
    // 创建一个 SequenceBarrier【目前只有 ProcessingSequenceBarrier 】
    final SequenceBarrier barrier = ringBuffer.newBarrier(barrierSequences);

    for (int i = 0, eventHandlersLength = eventHandlers.length; i < eventHandlersLength; i++) {
        // 将 EventHandler 封装成 EventProcessor
        final EventHandler<? super T> eventHandler = eventHandlers[i];
        final BatchEventProcessor<T> batchEventProcessor = new BatchEventProcessor<>(ringBuffer, barrier, eventHandler);
        if (exceptionHandler != null) {
            batchEventProcessor.setExceptionHandler(exceptionHandler);
        }
        // 存入消费者集合中
        consumerRepository.add(batchEventProcessor, eventHandler, barrier);
        processorSequences[i] = batchEventProcessor.getSequence();
    }
    // 因为 Disruptor 只维护头节点，对于尾节点是由生产者跟踪消费者的 Sequence 处理的
    updateGatingSequencesForNextInChain(barrierSequences, processorSequences);
    return new EventHandlerGroup<>(this, consumerRepository, processorSequences);
}

/**
* 因为 一个消费者 依赖 前置消费者，那么只有前置消费者处理完成后，当前消费者才能进行处理
* 这种情况下，生产者只需要关注 当前消费者的消费序号即可
* 因为当前消费者的消费速度必须小于等于其前置消费者
* 所以这里把当前消费者的前置消费者的序号从跟踪链中移除，同时将当前消费者的序号添加到跟踪链中
*
* @param barrierSequences   processorSequences 中依赖的前置消费者的 Sequence
* @param processorSequences 本次添加的消费者的 Sequence
*/
private void updateGatingSequencesForNextInChain(Sequence[] barrierSequences, Sequence[] processorSequences) {
    if (processorSequences.length > 0) {
        // 给生产者使用，生产者需要知道消费的位置，防止数据覆盖
        ringBuffer.addGatingSequences(processorSequences);
        for (final Sequence barrierSequence : barrierSequences) {
            // processorSequences 中有依赖 barrierSequences 的
            // 那么生产者只需要关注 processorSequences 的最小序号即可保证这个值是最小的序号
            // 因为需要 barrierSequences 中的先处理，再处理 processorSequences
            ringBuffer.removeGatingSequence(barrierSequence);
        }
        // 如果有前置消费者，则取消之前的前置消费者对应的 EventProcessor 的 链的终点标记。
        consumerRepository.unMarkEventProcessorsAsEndOfChain(barrierSequences);
    }
}
```

**根据这个分析，看一下代码段的结果**

```java
public class LongEventMain {
    public static void main(String[] args) throws Exception {
        LongEventFactory factory = new LongEventFactory();
        int bufferSize = 1 << 3;
        Disruptor<LongEvent> disruptor = new Disruptor<>(factory, bufferSize, Executors.defaultThreadFactory(),
                ProducerType.MULTI, new BlockingWaitStrategy());
        // 有2个消费，并且这两个消费者都需要消费 RingBuffer，没有先后顺序
        // 即一个 RingBuffer 的槽中一旦有时间发布
        // id = 1 和 id = 2 的消费者都会处理 事件
        disruptor.handleEventsWith(new LongEventHandler(1), new LongEventHandler(2));
        disruptor.start();
        RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();
        LongEventProducer producer = new LongEventProducer(ringBuffer);
        ByteBuffer bb = ByteBuffer.allocate(8);
        for (long l = 0; true; l++) {
            bb.putLong(0, l);
            producer.onData(bb);
            Thread.sleep(1000);
        }
    }
}
```

**结果**
红色的部分可以看出，消费没有先后顺序

![result](/images/disruptor/result1.png)

然后代码段改成

```java
disruptor.handleEventsWith(new LongEventHandler(1), new LongEventHandler(2)).handleEventsWith(new LongEventHandler(3));
```

**结果**
可以看出 id = 1 和 id = 2 没有先后顺序
id = 3 必须在 1 和 2 处理结束后处理

![result](/images/disruptor/result2.png)
























