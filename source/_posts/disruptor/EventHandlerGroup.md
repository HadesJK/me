---
title: Disruptor 源码分析四：EventHandlerGroup
date: 2017-04-12 09:49:38
categories:
- disruptor
---

在 Disruptor 类中，看到添加消费者的接口，返回的结果是一个 EventHandlerGroup

EventHandlerGroup 和 Disruptor 相似

Disruptor 类中创建 EventHandlerGroup 的地方，都传了一个参数：processorSequences
这个参数传给 EventHandlerGroup，用于记录前一次添加的消费者列表的 Sequences
如果后面再添加消费者，那么这些消费者的前置依赖消费者列表就是 processorSequences
再看 EventHandlerGroup 添加消费者的几个方法，都是调用 Disruptor 的方法

```java
public EventHandlerGroup<T> handleEventsWith(final EventHandler<? super T>... handlers) {
    return disruptor.createEventProcessors(sequences, handlers);
}

public EventHandlerGroup<T> handleEventsWith(final EventProcessorFactory<T>... eventProcessorFactories) {
    return disruptor.createEventProcessors(sequences, eventProcessorFactories);
}

public EventHandlerGroup<T> handleEventsWithWorkerPool(final WorkHandler<? super T>... handlers) {
    return disruptor.createWorkerPool(sequences, handlers);
}
```