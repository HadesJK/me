---
title: 生产者生产数据
date: 2017-04-07 11:26:42
categories:
- disruptor
---

分析了 Sequence 和 Sequencer，就可以了解生产者如何向 RingBuffer 写数据了

1. 获取到可用的 RingBuffer 序号
-> RingBuffer.next()[or RingBuffer.tryNext()]
-> Sequencer.next()[or Sequencer.tryNext()]

2. 然后通过这个序号[或者一系列序号]获取 RingBuffer 的 Event对象
-> RingBuffer.get()

3. 然后修改这个 Event 对象的数据

4. 发布数据
-> RingBuffer.publish()
-> Sequencer.publish()
