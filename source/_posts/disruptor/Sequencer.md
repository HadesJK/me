---
title: Disruptor 源码分析二：Sequencer
date: 2017-04-01 15:45:04
categories:
- disruptor
---

讲完了 Sequence 类，接下来分析 Sequencer 接口

Sequencer 持有 Sequence，并提供了一系列方法来操作 Sequence

Sequencer 是给事件生产者使用的

RingBuffer 的数据结构中，只记录了当前写的位置，即 head，但没有维护 tail

因此生产者为了防止覆盖写，需要记录消费者消费的位置

抽象类 AbstractSequencer 内部有一个 Sequence 数组 gatingSequences，记录消费者的消费位置

同时生产者需要知道 RingBuffer 的当前位置，AbstractSequencer 还保存一个 Sequence

就是通过这个 Sequence 来维护 RingBuffer 的位置


# Sequencer 类图

![Sequencer](/images/disruptor/Sequencer.png)

# Cursored 

```java
public interface Cursored {
    /**
     * 获取 RingBuffer 的当前位置
     */
    long getCursor();
}
```

# Sequenced

```java
public interface Sequenced {
    /**
     * 获取 RingBuffer 的大小
     */
    int getBufferSize();

    /**
     * 获取 RingBuffer 当前的可用容量，
     * 这个方法返回的并不精确，可能在计算中 RingBuffer 发生改变了
     */
    boolean hasAvailableCapacity(final int requiredCapacity);

    /**
     * Get the remaining capacity for this sequencer.
     *
     * @return The number of slots remaining.
     */
    long remainingCapacity();

    /**
     * 获取下一个可用的写空间位置
     */
    long next();

    /**
     * 获取下n个可用的写空间
     */
    long next(int n);

    /**
     * 尝试获取下一个可用的写空间
     * 如果没有，则抛出异常
     *
     * @throws InsufficientCapacityException
     */
    long tryNext() throws InsufficientCapacityException;

    /**
     * 尝试获取下n个可用的写空间
     * 如果没有，则抛出异常
     *
     * @throws InsufficientCapacityException
     */
    long tryNext(int n) throws InsufficientCapacityException;

    /**
     * 当生产者将消息填充到 RingBuffer 的槽后，
     * 调用这个方法将 RingBuffer 的序号设置为当前的 sequence 值
     */
    void publish(long sequence);

    /**
     * 当批量填充 RingBuffer 后，调用这个方法设置 RingBuffer 的序号
     *
     * @param lo first sequence number to publish
     * @param hi last sequence number to publish
     */
    void publish(long lo, long hi);
}
```

# Sequencer

Sequencer 继承了 Cursored, Sequenced 接口的基础上，又定义了一些方法

```java
public interface Sequencer extends Cursored, Sequenced {
    /**
     * Sequence 的初始值
     */
    long INITIAL_CURSOR_VALUE = -1L;

    /**
     * Claim a specific sequence.  Only used if initialising the ring buffer to
     * a specific value.
     *
     * @param sequence The sequence to initialise too.
     */
    void claim(long sequence);

    /**
     * 判断 RingBuffer 的 sequence 位置的槽是否可消费
     * 等价于 RingBuffer 的 sequence 位置的槽 事件是否已发布
     */
    boolean isAvailable(long sequence);

    /**
     * 添加一组消费者维护的序号 到控制序号组
     */
    void addGatingSequences(Sequence... gatingSequences);

    /**
     * 从控制序号组中 移除一个指定的消费者序号
     */
    boolean removeGatingSequence(Sequence sequence);

    /**
     * 创建一个序号栅栏
     * 【后面分析】
     */
    SequenceBarrier newBarrier(Sequence... sequencesToTrack);

    /**
     * 返回控制序号组中的最小序号
     * 当控制序号组的数组长度为0时，返回 RingBuffer 的位置
     */
    long getMinimumSequence();

    /**
     * Get the highest sequence number that can be safely read from the ring buffer.  Depending
     * on the implementation of the Sequencer this call may need to scan a number of values
     * in the Sequencer.  The scan will range from nextSequence to availableSequence.  If
     * there are no available values <code>&gt;= nextSequence</code> the return value will be
     * <code>nextSequence - 1</code>.  To work correctly a consumer should pass a value that
     * is 1 higher than the last sequence that was successfully processed.
     *
     * @param nextSequence      The sequence to start scanning from.
     * @param availableSequence The sequence to scan to.
     * @return The highest value that can be safely read, will be at least <code>nextSequence - 1</code>.
     */
    long getHighestPublishedSequence(long nextSequence, long availableSequence);

    <T> EventPoller<T> newPoller(DataProvider<T> provider, Sequence... gatingSequences);
}
```

# AbstractSequencer 

前面说过 Sequencer 是给生产者用的，AbstractSequencer 提供了基本的实现

```java
/**
 * 注意：这个对象是给【生产者(Producer)】用的
 */
public abstract class AbstractSequencer implements Sequencer {

    /**
     * 用来更新 控制序号组 这个字段
     */
    private static final AtomicReferenceFieldUpdater<AbstractSequencer, Sequence[]> SEQUENCE_UPDATER =
            AtomicReferenceFieldUpdater.newUpdater(AbstractSequencer.class, Sequence[].class, "gatingSequences");

    /**
     * RingBuffer 的大小，2的指数次方
     */
    protected final int bufferSize;

    /**
     * 消费者的等待策略
     */
    protected final WaitStrategy waitStrategy;

    /**
     * 维护 RingBuffer 的头部，
     * 生产者会竞争这个对象
     * 一个生产者，在生产 Event 时，
     * 1. 先获取下一可写的位置
     * 2. 填充 Event，
     * 3. 填充好后再 publish
     * 这之后，这个 Event 就可以被消费处理了
     * 注意：一旦获取到序号后，如果没有填充 Event，将导致旧数据被重复消费
     */
    protected final Sequence cursor = new Sequence(Sequencer.INITIAL_CURSOR_VALUE);

    /**
     * RingBuffer 的头由一个名字为 Cursor 的 Sequence对象维护，⬆️
     * 用来协调生产者向 RingBuffer 中填充数据。
     * 表示队列尾的 Sequence 并没有在 RingBuffer 中，
     * 而是由消费者维护，生产者为了防止数据被覆盖，需要跟踪消费者的消费信息
     * 控制序号组 gatingSequences 来维护消费者的消费位置信息
     * 可想而知，当消费者添加到 Disruptor 时，这个数组就需要维护了
     */
    protected volatile Sequence[] gatingSequences = new Sequence[0];

    /**
     * @param bufferSize   RingBuffer 的大小，2 的指数次
     * @param waitStrategy 等待策略
     */
    public AbstractSequencer(int bufferSize, WaitStrategy waitStrategy) {
        if (bufferSize < 1) {
            throw new IllegalArgumentException("bufferSize must not be less than 1");
        }
        if (Integer.bitCount(bufferSize) != 1) {
            throw new IllegalArgumentException("bufferSize must be a power of 2");
        }
        this.bufferSize = bufferSize;
        this.waitStrategy = waitStrategy;
    }

    /**
     * 返回 RingBuffer 的当前位置
     */
    @Override
    public final long getCursor() {
        return cursor.get();
    }

    /**
     * 获取 RingBuffer 的大小
     */
    @Override
    public final int getBufferSize() {
        return bufferSize;
    }

    /**
     * 添加一组消费者维护的序号 到控制序号组
     * 因为消费者需要在 Disruptor 启动之前添加的，
     * 因此这个方法也只能在启动之前调用
     */
    @Override
    public final void addGatingSequences(Sequence... gatingSequences) {
        SequenceGroups.addSequences(this, SEQUENCE_UPDATER, this, gatingSequences);
    }

    /**
     * 从控制序号组中 移除一个指定的消费者序号
     */
    @Override
    public boolean removeGatingSequence(Sequence sequence) {
        return SequenceGroups.removeSequence(this, SEQUENCE_UPDATER, sequence);
    }

    /**
     * 返回控制序号组中的最小序号
     * 当控制序号组的数组长度为0时，返回 RingBuffer 的位置
     */
    @Override
    public long getMinimumSequence() {
        return Util.getMinimumSequence(gatingSequences, cursor.get());
    }

    /**
     * todo: 后续分析
     */
    @Override
    public SequenceBarrier newBarrier(Sequence... sequencesToTrack) {
        return new ProcessingSequenceBarrier(this, waitStrategy, cursor, sequencesToTrack);
    }

    /**
     * Creates an event poller for this sequence that will use the supplied data provider and
     * gating sequences.
     *
     * @param dataProvider    The data source for users of this event poller
     * @param gatingSequences Sequence to be gated on.
     * @return A poller that will gate on this ring buffer and the supplied sequences.
     */
    @Override
    public <T> EventPoller<T> newPoller(DataProvider<T> dataProvider, Sequence... gatingSequences) {
        return EventPoller.newInstance(dataProvider, this, new Sequence(), cursor, gatingSequences);
    }

    @Override
    public String toString() {
        return "AbstractSequencer{" +
                "waitStrategy=" + waitStrategy +
                ", cursor=" + cursor +
                ", gatingSequences=" + Arrays.toString(gatingSequences) +
                '}';
    }
}
```

# SingleProducerSequencer

SingleProducerSequencer 作为单个生产者用的，实际情况一般都是多生产者多消费者的情况

单个生产者不会竞争 RingBuffer 的写操作


SingleProducerSequencer 内部缓存了 RingBuffer 的下一个可写的位置 nextValue
还缓存了消费者消费的最小位置 cachedValue

同时使用缓存行填充这两个字段来避免伪共享的问题

```java
public final class SingleProducerSequencer extends RightPad {
    public SingleProducerSequencer(int bufferSize, final WaitStrategy waitStrategy) {
        super(bufferSize, waitStrategy);
    }
    
    @Override
    public boolean hasAvailableCapacity(final int requiredCapacity) {
        long nextValue = this.nextValue;

        /**
         * RingBuffer 的序号值是递增的，不像一般的 环形队列，当超过 size 后，值会取模
         *
         * nextValue + requiredCapacity 表示 RingBuffer 将要达到的位置
         * 这里又减去了一个 bufferSize：
         * wrapPoint = (nextValue + requiredCapacity) - bufferSize;
         * 因此，当 wrapPoint <= 0 时，表示一圈还未走完，那么空间肯定是有的
         *
         * cachedValue 是一个旧值，只在必要的时候取维护更新
         * cachedValue 这个值偏小的，也就是说 消费者往前消费时，这个 cachedValue 值并没有更新【生产者维护的】
         *
         * 讲道理 没有出现 cachedGatingSequence > nextValue 的情况把？
         *
         * 如果 wrapPoint <= cachedGatingSequence，那肯定有空间
         * 如果 wrapPoint > cachedGatingSequence，这时候需要获取消费者的最新消费位置，即更新 cachedValue
         * 举个栗子：
         * RingBuffer 的 size = 8
         * nextValue = 18
         * cachedValue = 12
         * 然而消费者接下来一直在消费，当最慢的消费者消费到 15 时
         * 这时候需要申请 1 个空间，那么 wrapPoint = 18 + 1 - 8 = 11 < cachedValue
         * 那么直接返回有空间 true
         *
         * 此时 nextValue = 19
         * cachedValue = 12
         * 这时候又申请 4 个空间
         *
         * 那么 wrapPoint = 19 + 4 - 8 = 15 > cachedValue
         * 这时候更新 cachedValue
         * 更新之后 cachedValue = 15
         * 此时判断发现 nextValue >= cachedValue
         * 又返回true
         *
         */
        long wrapPoint = (nextValue + requiredCapacity) - bufferSize;
        long cachedGatingSequence = this.cachedValue;
        if (wrapPoint > cachedGatingSequence || cachedGatingSequence > nextValue) {
            cursor.setVolatile(nextValue);  // StoreLoad fence
            long minSequence = Util.getMinimumSequence(gatingSequences, nextValue);
            // 重新加载 当前消费者 最小的消费序号, 【这个值生产者不需要每次都维护（只在必要的时候维护）】
            this.cachedValue = minSequence;
            if (wrapPoint > minSequence) {
                return false;
            }
        }
        return true;
    }

    @Override
    public long next() {
        return next(1);
    }

    @Override
    public long next(int n) {
        if (n < 1) {
            throw new IllegalArgumentException("n must be > 0");
        }

        long nextValue = this.nextValue;

        long nextSequence = nextValue + n;
        long wrapPoint = nextSequence - bufferSize;
        long cachedGatingSequence = this.cachedValue;

        if (wrapPoint > cachedGatingSequence || cachedGatingSequence > nextValue) {
            cursor.setVolatile(nextValue);  // StoreLoad fence

            long minSequence;
            while (wrapPoint > (minSequence = Util.getMinimumSequence(gatingSequences, nextValue))) {
                // 通知消费者有数据可以消费【消费者消费后，生产者又可以发布数据了】
                waitStrategy.signalAllWhenBlocking();
                LockSupport.parkNanos(1L); // TODO: Use waitStrategy to spin?
            }
            // 从新加载 当前消费者 最小的消费序号, 【这个值生产者不需要每次都维护（只在必要的时候维护）】
            this.cachedValue = minSequence;
        }
        this.nextValue = nextSequence;

        return nextSequence;
    }

    @Override
    public long tryNext() throws InsufficientCapacityException {
        return tryNext(1);
    }

    @Override
    public long tryNext(int n) throws InsufficientCapacityException {
        if (n < 1) {
            throw new IllegalArgumentException("n must be > 0");
        }

        if (!hasAvailableCapacity(n)) {
            throw InsufficientCapacityException.INSTANCE;
        }

        long nextSequence = this.nextValue += n;

        return nextSequence;
    }

    @Override
    public long remainingCapacity() {
        long nextValue = this.nextValue;

        long consumed = Util.getMinimumSequence(gatingSequences, nextValue);
        long produced = nextValue;
        return getBufferSize() - (produced - consumed);
    }

    @Override
    public void claim(long sequence) {
        this.nextValue = sequence;
    }

    @Override
    public void publish(long sequence) {
        // 发布之后，设置当前的序号
        cursor.set(sequence);
        // 发布之后，通知所有需要被通知的消费者
        waitStrategy.signalAllWhenBlocking();
    }

    /**
     * 当批量填充 RingBuffer 后，调用这个方法设置 RingBuffer 的序号
     * 单生产者的情况下，没有批量发布
     *
     * @param lo first sequence number to publish
     * @param hi last sequence number to publish
     */
    @Override
    public void publish(long lo, long hi) {
        publish(hi);
    }

    @Override
    public boolean isAvailable(long sequence) {
        return sequence <= cursor.get();
    }

    @Override
    public long getHighestPublishedSequence(long lowerBound, long availableSequence) {
        return availableSequence;
    }
}
```

# MultiProducerSequencer

MultiProducerSequencer 是多事件生产者的情况，
相比 SingleProducerSequencer，MultiProducerSequencer应该会更常用，也是默认的方式

```java
public final class MultiProducerSequencer extends AbstractSequencer {
    private static final Unsafe UNSAFE = Util.getUnsafe();
    // 获取 int[] 数组在类中的偏移地址
    private static final long BASE = UNSAFE.arrayBaseOffset(int[].class);
    // 获取 int[] 数组中每个元素的 size，java 中int类型是 4 字节的
    private static final long SCALE = UNSAFE.arrayIndexScale(int[].class);
    // 缓存了消费者的最小消费序号，这个值只在必要的时候更新
    private final Sequence gatingSequenceCache = new Sequence(Sequencer.INITIAL_CURSOR_VALUE);

    // availableBuffer tracks the state of each ringbuffer slot
    // see below for more details on the approach
    // 记录 RingBuffer 每一个槽的状态
    // 圈数, 初始值 -1，第 1 圈用 0 表示，第 2 圈用 1 表示
    private final int[] availableBuffer;
    // indexMask 和 indexShift 和 RingBuffer 中的字段类似
    // indexMask =  bufferSize - 1， 假设 buffersize = 1024， 那么 indexMask = 1023
    private final int indexMask;
    // indexShift = log2(bufferSize)， 假设 buffersize = 1024， 那么 indexShift = 10
    private final int indexShift;

    /**
     * Construct a Sequencer with the selected wait strategy and buffer size.
     *
     * @param bufferSize   the size of the buffer that this will sequence over.
     * @param waitStrategy for those waiting on sequences.
     */
    public MultiProducerSequencer(int bufferSize, final WaitStrategy waitStrategy) {
        super(bufferSize, waitStrategy);
        availableBuffer = new int[bufferSize];
        indexMask = bufferSize - 1;
        indexShift = Util.log2(bufferSize);
        initialiseAvailableBuffer();
    }

    @Override
    public boolean hasAvailableCapacity(final int requiredCapacity) {
        return hasAvailableCapacity(gatingSequences, requiredCapacity, cursor.get());
    }

    private boolean hasAvailableCapacity(Sequence[] gatingSequences, final int requiredCapacity, long cursorValue) {
        long wrapPoint = (cursorValue + requiredCapacity) - bufferSize;
        long cachedGatingSequence = gatingSequenceCache.get();
        if (wrapPoint > cachedGatingSequence || cachedGatingSequence > cursorValue) {
            // 获取消费者的最小消费的序号， 这个值可能不等于 gatingSequenceCache
            long minSequence = Util.getMinimumSequence(gatingSequences, cursorValue);
            // 维护 gatingSequenceCache
            gatingSequenceCache.set(minSequence);
            if (wrapPoint > minSequence) {
                return false;
            }
        }
        return true;
    }

    @Override
    public void claim(long sequence) {
        cursor.set(sequence);
    }

    @Override
    public long next() {
        return next(1);
    }

    @Override
    public long next(int n) {
        if (n < 1) {
            throw new IllegalArgumentException("n must be > 0");
        }

        long current;
        long next;

        do {
            // RingBuffer 的 head 位置
            current = cursor.get();
            next = current + n;
            long wrapPoint = next - bufferSize;
            long cachedGatingSequence = gatingSequenceCache.get();
            if (wrapPoint > cachedGatingSequence || cachedGatingSequence > current) {
                long gatingSequence = Util.getMinimumSequence(gatingSequences, current);
                if (wrapPoint > gatingSequence) {
                    // 通知等待的消费者，现在有数据可用
                    waitStrategy.signalAllWhenBlocking();
                    LockSupport.parkNanos(1); // TODO, should we spin based on the wait strategy?
                    continue;
                }
                gatingSequenceCache.set(gatingSequence);
            } else if (cursor.compareAndSet(current, next)) { // 和 SingleProducerSequencer 不同，这里需要 cas 进行写操作
                break;
            }
        } while (true);
        return next;
    }

    @Override
    public long tryNext() throws InsufficientCapacityException {
        return tryNext(1);
    }

    /**
     * @see Sequencer#tryNext(int)
     */
    @Override
    public long tryNext(int n) throws InsufficientCapacityException {
        if (n < 1) {
            throw new IllegalArgumentException("n must be > 0");
        }
        long current;
        long next;
        do {
            current = cursor.get();
            next = current + n;

            if (!hasAvailableCapacity(gatingSequences, n, current)) {
                throw InsufficientCapacityException.INSTANCE;
            }
        }
        while (!cursor.compareAndSet(current, next));
        return next;
    }

    @Override
    public long remainingCapacity() {
        long consumed = Util.getMinimumSequence(gatingSequences, cursor.get());
        long produced = cursor.get();
        return getBufferSize() - (produced - consumed);
    }

    private void initialiseAvailableBuffer() {
        for (int i = availableBuffer.length - 1; i != 0; i--) {
            setAvailableBufferValue(i, -1);
        }
        setAvailableBufferValue(0, -1);
    }

    @Override
    public void publish(final long sequence) {
        setAvailable(sequence);
        waitStrategy.signalAllWhenBlocking();
    }

    /**
     * 维护 RingBuffer 每一个槽的数组的 [lo, hi] 位置更新
     */
    @Override
    public void publish(long lo, long hi) {
        for (long l = lo; l <= hi; l++) {
            setAvailable(l);
        }
        waitStrategy.signalAllWhenBlocking();
    }

    /**
     * The below methods work on the availableBuffer flag.
     * <p>
     * The prime reason is to avoid a shared sequence object between publisher threads.
     * (Keeping single pointers tracking start and end would require coordination
     * between the threads).
     * <p>
     * --  Firstly we have the constraint that the delta between the cursor and minimum
     * gating sequence will never be larger than the buffer size (the code in
     * next/tryNext in the Sequence takes care of that).
     * -- Given that; take the sequence value and mask off the lower portion of the
     * sequence as the index into the buffer (indexMask). (aka modulo operator)
     * -- The upper portion of the sequence becomes the value to check for availability.
     * ie: it tells us how many times around the ring buffer we've been (aka division)
     * -- Because we can't wrap without the gating sequences moving forward (i.e. the
     * minimum gating sequence is effectively our last available position in the
     * buffer), when we have new data and successfully claimed a slot we can simply
     * write over the top.
     */
    private void setAvailable(final long sequence) {
        setAvailableBufferValue(calculateIndex(sequence), calculateAvailabilityFlag(sequence));
    }

    private void setAvailableBufferValue(int index, int flag) {
        long bufferAddress = (index * SCALE) + BASE;
        UNSAFE.putOrderedInt(availableBuffer, bufferAddress, flag);
    }

    @Override
    public boolean isAvailable(long sequence) {
        int index = calculateIndex(sequence);
        int flag = calculateAvailabilityFlag(sequence);
        long bufferAddress = (index * SCALE) + BASE;
        // 圈数相同代表可用
        return UNSAFE.getIntVolatile(availableBuffer, bufferAddress) == flag;
    }

    @Override
    public long getHighestPublishedSequence(long lowerBound, long availableSequence) {
        for (long sequence = lowerBound; sequence <= availableSequence; sequence++) {
            if (!isAvailable(sequence)) {
                return sequence - 1;
            }
        }
        return availableSequence;
    }

    /**
     * >>> 无符号右移，忽略符号位，空位都以 0 补齐
     * 相当于计算 圈数, 第1圈用 0 表示，第2圈用 1 表示
     */
    private int calculateAvailabilityFlag(final long sequence) {
        return (int) (sequence >>> indexShift);
    }

    private int calculateIndex(final long sequence) {
        return ((int) sequence) & indexMask;
    }
}
```















