---
layout: post
title:  "Disruptor编码总体流程"
date:   2018-10-01 8:50:9
categories: Java
comments: true
tags:
    - Disruptor
    - 源码阅读
   
---

* content
{:toc}

# Disruptor工作流程

总体概览Disruptor编程模型的情况

[![EHjgH0.md.png](https://s2.ax1x.com/2019/05/16/EHjgH0.md.png)](https://imgchr.com/i/EHjgH0)

调价消费者监听类
Disruptor 要和消费者关联起来disruptor.handleEventWith(**消费者处理事件** ),可以绑定多个消费者处理

可以把Disruptor看做一个线程池的ThreadPoolExecutors服务，一步一把这个东西给构建出来

## 一Disruptor 组件配置
- 定义event,考虑内存是否等方面的情况
- 定义eventFactory工程， 重写方法newinstance()方法
- 定义consumer消费者，消费者是直接放进入Disruptor构造方法当中的，
  - Disruptor方法，构造方法
  - eventfactory
  - ringbuffer大小
  - exector线程池
  - 生产者类型
  - 消费者从等待策略
 
 ## 生产者组件投递数据
 
 投递数据sendData数据之类，情况,
 
 如何投递数据这一块
 
 在这里面的情况
  
  ```java
  //获取得到下标
  long sequence  =ringBuffer=.next()
 // 我们知道Disruptor 采用内存预分配的策略，每一个Disruptor在使用之前都需要预先指定好大小的ringbuffer并且指定这个ringbuffer是存储什么类型的数据木 
 // 比如我们上面所讲解的orderEvent，所有在所有ringbuffe所有的sequence 当中都是一个orderEvent对象了，可能就有的orderEvent 为空而已
 // 所以下一步
  orderEvent envet = ringbuffer.get(sequence)
  event.setValue(data.getLong());
  //然后再有一个提交操作， 
  ringbuffer.publish(sequence) // 发布同一个
  ```
 
 
 ## 思考问题
 为何需要进行next? 这个点看起来有点奇怪
 >主要内存预分配的原则，避免频繁地进行gc.
 
 
 # 用法
 
 用法很简单，一共有三个角色，一生产者，而是消费者，三个Disruptor 对象，创建一个Ringbuffer对象,Disruptor 对象里面包括一个
 
 创建一个Disruptor对象有两个构造方法，其实这个对象的构造方法实质可以对照JDK 当中的线程池ThreadPoolExecutors, 因此后面也有类似于线程池的
 
 
 **Disruptor对象**
```java
public Disruptor(
        final EventFactory<T> eventFactory,    // 数据实体构造工厂
        final int ringBufferSize,              // 队列大小，必须是2的次方
        final ThreadFactory threadFactory,     // 线程工厂
        final ProducerType producerType,       // 生产者类型，单个生产者还是多个
        final WaitStrategy waitStrategy){      // 消费者等待策略
    ...
}

public Disruptor(
    final EventFactory<T> eventFactory,     
    final int ringBufferSize, 
    final ThreadFactory threadFactory){
    ...
}
```
 **生产者关键是实现Send方法**
 
 ```java
 public void send(String data){
    RingBuffer<MsgEvent> ringBuffer = this.disruptor.getRingBuffer();
    //获取下一个放置数据的位置
    long next = ringBuffer.next();
    try{
        MsgEvent event = ringBuffer.get(next);
        event.setValue(data);
    }finally {
        //发布事件
        ringBuffer.publish(next);
    }
}
 ```
 
 
 **消息批处理**
 
 ```java
public EventHandlerGroup<T> handleEventsWith(final EventHandler<? super T>... handlers){
    ...
}
public EventHandlerGroup<T> handleEventsWith(final EventProcessor... processors){
    ...
}
public EventHandlerGroup<T> handleEventsWith(final EventProcessorFactory<T>... eventProcessorFactories){
    ...
}
public EventHandlerGroup<T> handleEventsWithWorkerPool(final WorkHandler<T>... workHandlers){
    ...
}
 ```
 
 **简单用例**
 ```java
 //消费者
public class MsgConsumer implements EventHandler<MsgEvent>{
    private String name;
    public MsgConsumer(String name){
        this.name = name;
    }
    @Override
    public void onEvent(MsgEvent msgEvent, long l, boolean b) throws Exception {
        System.out.println(this.name+" -> 接收到信息： "+msgEvent.getValue());
    }
}

//生产者处理
public class MsgProducer {
    private Disruptor disruptor;
    public MsgProducer(Disruptor disruptor){
        this.disruptor = disruptor;
    }
    public void send(String data){
        RingBuffer<MsgEvent> ringBuffer = this.disruptor.getRingBuffer();
        long next = ringBuffer.next();
        try{
            MsgEvent event = ringBuffer.get(next);
            event.setValue(data);
        }finally {
            ringBuffer.publish(next);
        }
    }

    public void send(List<String> dataList){
        dataList.stream().forEach(data -> this.send(data));
    }
}


//触发测试
public class DisruptorDemo {
    @Test
    public void test(){
        Disruptor<MsgEvent> disruptor = new Disruptor<>(MsgEvent::new, 1024, Executors.defaultThreadFactory());

        //定义消费者
        MsgConsumer msg1 = new MsgConsumer("1");
        MsgConsumer msg2 = new MsgConsumer("2");
        MsgConsumer msg3 = new MsgConsumer("3");

        //绑定配置关系
        disruptor.handleEventsWith(msg1, msg2, msg3);
        disruptor.start();

        // 定义要发送的数据
        MsgProducer msgProducer = new MsgProducer(disruptor);
        msgProducer.send(Arrays.asList("nihao","hah"));
    }
}

 ```
 
 # Disruptor最新版源码
 
 ```java
 public class Disruptor<T>
{
  // 里面的Ringbuffer对象
    private final RingBuffer<T> ringBuffer;
    private final Executor executor;
    private final ConsumerRepository<T> consumerRepository = new ConsumerRepository<>();
    private final AtomicBoolean started = new AtomicBoolean(false);
    private ExceptionHandler<? super T> exceptionHandler = new ExceptionHandlerWrapper<>();

    /**
     * Create a new Disruptor. Will default to {@link com.lmax.disruptor.BlockingWaitStrategy} and
     * {@link ProducerType}.MULTI
     *
     * @deprecated Use a {@link ThreadFactory} instead of an {@link Executor} as a the ThreadFactory
     * is able to report errors when it is unable to construct a thread to run a producer.
     *
     * @param eventFactory   the factory to create events in the ring buffer.
     * @param ringBufferSize the size of the ring buffer.
     * @param executor       an {@link Executor} to execute event processors.
     */
    @Deprecated
    public Disruptor(final EventFactory<T> eventFactory, final int ringBufferSize, final Executor executor)
    {
        this(RingBuffer.createMultiProducer(eventFactory, ringBufferSize), executor);
    }
    /**
     * Create a new Disruptor.
     *
     * @deprecated Use a {@link ThreadFactory} instead of an {@link Executor} as a the ThreadFactory
     * is able to report errors when it is unable to construct a thread to run a producer.
     *
     * @param eventFactory   the factory to create events in the ring buffer.
     * @param ringBufferSize the size of the ring buffer, must be power of 2.
     * @param executor       an {@link Executor} to execute event processors.
     * @param producerType   the claim strategy to use for the ring buffer.
     * @param waitStrategy   the wait strategy to use for the ring buffer.
     */
    @Deprecated
    public Disruptor(
        final EventFactory<T> eventFactory,
        final int ringBufferSize,
        final Executor executor,
        final ProducerType producerType,
        final WaitStrategy waitStrategy)
    {
        this(RingBuffer.create(producerType, eventFactory, ringBufferSize, waitStrategy), executor);
    }

    /**
     * Create a new Disruptor. Will default to {@link com.lmax.disruptor.BlockingWaitStrategy} and
     * {@link ProducerType}.MULTI
     *
     * @param eventFactory   the factory to create events in the ring buffer.
     * @param ringBufferSize the size of the ring buffer.
     * @param threadFactory  a {@link ThreadFactory} to create threads to for processors.
     */
    public Disruptor(final EventFactory<T> eventFactory, final int ringBufferSize, final ThreadFactory threadFactory)
    {
        this(RingBuffer.createMultiProducer(eventFactory, ringBufferSize), new BasicExecutor(threadFactory));
    }
    /**
     * Create a new Disruptor.
     *
     * @param eventFactory   the factory to create events in the ring buffer.
     * @param ringBufferSize the size of the ring buffer, must be power of 2.
     * @param threadFactory  a {@link ThreadFactory} to create threads for processors.
     * @param producerType   the claim strategy to use for the ring buffer.
     * @param waitStrategy   the wait strategy to use for the ring buffer.
     */
    public Disruptor(
            final EventFactory<T> eventFactory,
            final int ringBufferSize,
            final ThreadFactory threadFactory,
            final ProducerType producerType,
            final WaitStrategy waitStrategy)
    {
        this(
            RingBuffer.create(producerType, eventFactory, ringBufferSize, waitStrategy),
            new BasicExecutor(threadFactory));
    }

    /**
     * Private constructor helper
     */
    private Disruptor(final RingBuffer<T> ringBuffer, final Executor executor)
    {
        this.ringBuffer = ringBuffer;
        this.executor = executor;
    }

    /**
     * <p>Set up event handlers to handle events from the ring buffer. These handlers will process events
     * as soon as they become available, in parallel.</p>
     *
     * <p>This method can be used as the start of a chain. For example if the handler <code>A</code> must
     * process events before handler <code>B</code>:</p>
     * <pre><code>dw.handleEventsWith(A).then(B);</code></pre>
     *
     * <p>This call is additive, but generally should only be called once when setting up the Disruptor instance</p>
     *
     * @param handlers the event handlers that will process events.
     * @return a {@link EventHandlerGroup} that can be used to chain dependencies.
     */
    @SuppressWarnings("varargs")
    @SafeVarargs
    public final EventHandlerGroup<T> handleEventsWith(final EventHandler<? super T>... handlers)
    {
        return createEventProcessors(new Sequence[0], handlers);
    }
    /**
     * <p>Set up custom event processors to handle events from the ring buffer. The Disruptor will
     * automatically start these processors when {@link #start()} is called.</p>
     *
     * <p>This method can be used as the start of a chain. For example if the handler <code>A</code> must
     * process events before handler <code>B</code>:</p>
     * <pre><code>dw.handleEventsWith(A).then(B);</code></pre>
     *
     * <p>Since this is the start of the chain, the processor factories will always be passed an empty <code>Sequence</code>
     * array, so the factory isn't necessary in this case. This method is provided for consistency with
     * {@link EventHandlerGroup#handleEventsWith(EventProcessorFactory...)} and {@link EventHandlerGroup#then(EventProcessorFactory...)}
     * which do have barrier sequences to provide.</p>
     *
     * <p>This call is additive, but generally should only be called once when setting up the Disruptor instance</p>
     *
     * @param eventProcessorFactories the event processor factories to use to create the event processors that will process events.
     * @return a {@link EventHandlerGroup} that can be used to chain dependencies.
     */
    @SafeVarargs
    public final EventHandlerGroup<T> handleEventsWith(final EventProcessorFactory<T>... eventProcessorFactories)
    {
        final Sequence[] barrierSequences = new Sequence[0];
        return createEventProcessors(barrierSequences, eventProcessorFactories);
    }
    /**
     * <p>Set up custom event processors to handle events from the ring buffer. The Disruptor will
     * automatically start this processors when {@link #start()} is called.</p>
     *
     * <p>This method can be used as the start of a chain. For example if the processor <code>A</code> must
     * process events before handler <code>B</code>:</p>
     * <pre><code>dw.handleEventsWith(A).then(B);</code></pre>
     *
     * @param processors the event processors that will process events.
     * @return a {@link EventHandlerGroup} that can be used to chain dependencies.
     */
    public EventHandlerGroup<T> handleEventsWith(final EventProcessor... processors)
    {
        for (final EventProcessor processor : processors)
        {
            consumerRepository.add(processor);
        }
        final Sequence[] sequences = new Sequence[processors.length];
        for (int i = 0; i < processors.length; i++)
        {
            sequences[i] = processors[i].getSequence();
        }
        ringBuffer.addGatingSequences(sequences);
        return new EventHandlerGroup<>(this, consumerRepository, Util.getSequencesFor(processors));
    }
    /**
     * Set up a to distribute an event to one of a pool of work handler threads.
     * Each event will only be processed by one of the work handlers.
     * The Disruptor will automatically start this processors when {@link #start()} is called.
     *
     * @param workHandlers the work handlers that will process events.
     * @return a  that can be used to chain dependencies.
     */
    @SafeVarargs
    @SuppressWarnings("varargs")
    public final EventHandlerGroup<T> handleEventsWithWorkerPool(final WorkHandler<T>... workHandlers)
    {
        return createWorkerPool(new Sequence[0], workHandlers);
    }
    /**
     * <p>Specify an exception handler to be used for any future event handlers.</p>
     *
     * <p>Note that only event handlers set up after calling this method will use the exception handler.</p>
     *
     * @param exceptionHandler the exception handler to use for any future {@link EventProcessor}.
     * @deprecated This method only applies to future event handlers. Use setDefaultExceptionHandler instead which applies to existing and new event handlers.
     */
    public void handleExceptionsWith(final ExceptionHandler<? super T> exceptionHandler)
    {
        this.exceptionHandler = exceptionHandler;
    }
    /**
     * <p>Specify an exception handler to be used for event handlers and worker pools created by this Disruptor.</p>
     *
     * <p>The exception handler will be used by existing and future event handlers and worker pools created by this Disruptor instance.</p>
     *
     * @param exceptionHandler the exception handler to use.
     */
    @SuppressWarnings("unchecked")
    public void setDefaultExceptionHandler(final ExceptionHandler<? super T> exceptionHandler)
    {
        checkNotStarted();
        if (!(this.exceptionHandler instanceof ExceptionHandlerWrapper))
        {
            throw new IllegalStateException("setDefaultExceptionHandler can not be used after handleExceptionsWith");
        }
        ((ExceptionHandlerWrapper<T>)this.exceptionHandler).switchTo(exceptionHandler);
    }
    /**
     * Override the default exception handler for a specific handler.
     * <pre>disruptorWizard.handleExceptionsIn(eventHandler).with(exceptionHandler);</pre>
     *
     * @param eventHandler the event handler to set a different exception handler for.
     * @return an ExceptionHandlerSetting dsl object - intended to be used by chaining the with method call.
     */
    public ExceptionHandlerSetting<T> handleExceptionsFor(final EventHandler<T> eventHandler)
    {
        return new ExceptionHandlerSetting<>(eventHandler, consumerRepository);
    }
    /**
     * <p>Create a group of event handlers to be used as a dependency.
     * For example if the handler <code>A</code> must process events before handler <code>B</code>:</p>
     *
     * <pre><code>dw.after(A).handleEventsWith(B);</code></pre>
     *
     * @param handlers the event handlers, previously set up with {@link #handleEventsWith(com.lmax.disruptor.EventHandler[])},
     *                 that will form the barrier for subsequent handlers or processors.
     * @return an {@link EventHandlerGroup} that can be used to setup a dependency barrier over the specified event handlers.
     */
    @SafeVarargs
    @SuppressWarnings("varargs")
    public final EventHandlerGroup<T> after(final EventHandler<T>... handlers)
    {
        final Sequence[] sequences = new Sequence[handlers.length];
        for (int i = 0, handlersLength = handlers.length; i < handlersLength; i++)
        {
            sequences[i] = consumerRepository.getSequenceFor(handlers[i]);
        }

        return new EventHandlerGroup<>(this, consumerRepository, sequences);
    }
    /**
     * Create a group of event processors to be used as a dependency.
     *
     * @param processors the event processors, previously set up with {@link #handleEventsWith(com.lmax.disruptor.EventProcessor...)},
     *                   that will form the barrier for subsequent handlers or processors.
     * @return an {@link EventHandlerGroup} that can be used to setup a {@link SequenceBarrier} over the specified event processors.
     * @see #after(com.lmax.disruptor.EventHandler[])
     */
    public EventHandlerGroup<T> after(final EventProcessor... processors)
    {
        return new EventHandlerGroup<>(this, consumerRepository, Util.getSequencesFor(processors));
    }
    /**
     * Publish an event to the ring buffer.
     *
     * @param eventTranslator the translator that will load data into the event.
     */
    public void publishEvent(final EventTranslator<T> eventTranslator)
    {
        ringBuffer.publishEvent(eventTranslator);
    }
    /**
     * Publish an event to the ring buffer.
     *
     * @param <A> Class of the user supplied argument.
     * @param eventTranslator the translator that will load data into the event.
     * @param arg             A single argument to load into the event
     */
    public <A> void publishEvent(final EventTranslatorOneArg<T, A> eventTranslator, final A arg)
    {
        ringBuffer.publishEvent(eventTranslator, arg);
    }
    /**
     * Publish a batch of events to the ring buffer.
     *
     * @param <A> Class of the user supplied argument.
     * @param eventTranslator the translator that will load data into the event.
     * @param arg             An array single arguments to load into the events. One Per event.
     */
    public <A> void publishEvents(final EventTranslatorOneArg<T, A> eventTranslator, final A[] arg)
    {
        ringBuffer.publishEvents(eventTranslator, arg);
    }
    /**
     * Publish an event to the ring buffer.
     *
     * @param <A> Class of the user supplied argument.
     * @param <B> Class of the user supplied argument.
     * @param eventTranslator the translator that will load data into the event.
     * @param arg0            The first argument to load into the event
     * @param arg1            The second argument to load into the event
     */
    public <A, B> void publishEvent(final EventTranslatorTwoArg<T, A, B> eventTranslator, final A arg0, final B arg1)
    {
        ringBuffer.publishEvent(eventTranslator, arg0, arg1);
    }
    /**
     * Publish an event to the ring buffer.
     *
     * @param eventTranslator the translator that will load data into the event.
     * @param <A> Class of the user supplied argument.
     * @param <B> Class of the user supplied argument.
     * @param <C> Class of the user supplied argument.
     * @param arg0            The first argument to load into the event
     * @param arg1            The second argument to load into the event
     * @param arg2            The third argument to load into the event
     */
    public <A, B, C> void publishEvent(final EventTranslatorThreeArg<T, A, B, C> eventTranslator, final A arg0, final B arg1, final C arg2)
    {
        ringBuffer.publishEvent(eventTranslator, arg0, arg1, arg2);
    }
    /**
     * <p>Starts the event processors and returns the fully configured ring buffer.</p>
     *
     * <p>The ring buffer is set up to prevent overwriting any entry that is yet to
     * be processed by the slowest event processor.</p>
     *
     * <p>This method must only be called once after all event processors have been added.</p>
     *
     * @return the configured ring buffer.
     */
    public RingBuffer<T> start()
    {
        checkOnlyStartedOnce();
        for (final ConsumerInfo consumerInfo : consumerRepository)
        {
            consumerInfo.start(executor);
        }
        return ringBuffer;
    }
    /**
     * Calls {@link com.lmax.disruptor.EventProcessor#halt()} on all of the event processors created via this disruptor.
     */
    public void halt()
    {
        for (final ConsumerInfo consumerInfo : consumerRepository)
        {
            consumerInfo.halt();
        }
    }
    /**
     * <p>Waits until all events currently in the disruptor have been processed by all event processors
     * and then halts the processors.  It is critical that publishing to the ring buffer has stopped
     * before calling this method, otherwise it may never return.</p>
     *
     * <p>This method will not shutdown the executor, nor will it await the final termination of the
     * processor threads.</p>
     */
    public void shutdown()
    {
        try
        {
            shutdown(-1, TimeUnit.MILLISECONDS);
        }
        catch (final TimeoutException e)
        {
            exceptionHandler.handleOnShutdownException(e);
        }
    }
    /**
     * <p>Waits until all events currently in the disruptor have been processed by all event processors
     * and then halts the processors.</p>
     *
     * <p>This method will not shutdown the executor, nor will it await the final termination of the
     * processor threads.</p>
     *
     * @param timeout  the amount of time to wait for all events to be processed. <code>-1</code> will give an infinite timeout
     * @param timeUnit the unit the timeOut is specified in
     * @throws TimeoutException if a timeout occurs before shutdown completes.
     */
    public void shutdown(final long timeout, final TimeUnit timeUnit) throws TimeoutException
    {
        final long timeOutAt = System.currentTimeMillis() + timeUnit.toMillis(timeout);
        while (hasBacklog())
        {
            if (timeout >= 0 && System.currentTimeMillis() > timeOutAt)
            {
                throw TimeoutException.INSTANCE;
            }
            // Busy spin
        }
        halt();
    }
    /**
     * The {@link RingBuffer} used by this Disruptor.  This is useful for creating custom
     * event processors if the behaviour of {@link BatchEventProcessor} is not suitable.
     *
     * @return the ring buffer used by this Disruptor.
     */
    public RingBuffer<T> getRingBuffer()
    {
        return ringBuffer;
    }
    /**
     * Get the value of the cursor indicating the published sequence.
     *
     * @return value of the cursor for events that have been published.
     */
    public long getCursor()
    {
        return ringBuffer.getCursor();
    }
    /**
     * The capacity of the data structure to hold entries.
     *
     * @return the size of the RingBuffer.
     * @see com.lmax.disruptor.Sequencer#getBufferSize()
     */
    public long getBufferSize()
    {
        return ringBuffer.getBufferSize();
    }
    /**
     * Get the event for a given sequence in the RingBuffer.
     *
     * @param sequence for the event.
     * @return event for the sequence.
     * @see RingBuffer#get(long)
     */
    public T get(final long sequence)
    {
        return ringBuffer.get(sequence);
    }
    /**
     * Get the {@link SequenceBarrier} used by a specific handler. Note that the {@link SequenceBarrier}
     * may be shared by multiple event handlers.
     *
     * @param handler the handler to get the barrier for.
     * @return the SequenceBarrier used by <i>handler</i>.
     */
    public SequenceBarrier getBarrierFor(final EventHandler<T> handler)
    {
        return consumerRepository.getBarrierFor(handler);
    }
    /**
     * Gets the sequence value for the specified event handlers.
     *
     * @param b1 eventHandler to get the sequence for.
     * @return eventHandler's sequence
     */
    public long getSequenceValueFor(final EventHandler<T> b1)
    {
        return consumerRepository.getSequenceFor(b1).get();
    }
   /**
     * Confirms if all messages have been consumed by all event processors
     */
    private boolean hasBacklog()
    {
        final long cursor = ringBuffer.getCursor();
        for (final Sequence consumer : consumerRepository.getLastSequenceInChain(false))
        {
            if (cursor > consumer.get())
            {
                return true;
            }
        }
        return false;
    }
    EventHandlerGroup<T> createEventProcessors(
        final Sequence[] barrierSequences,
        final EventHandler<? super T>[] eventHandlers)
    {
        checkNotStarted();
        final Sequence[] processorSequences = new Sequence[eventHandlers.length];
        final SequenceBarrier barrier = ringBuffer.newBarrier(barrierSequences);
        for (int i = 0, eventHandlersLength = eventHandlers.length; i < eventHandlersLength; i++)
        {
            final EventHandler<? super T> eventHandler = eventHandlers[i];

            final BatchEventProcessor<T> batchEventProcessor =
                new BatchEventProcessor<>(ringBuffer, barrier, eventHandler);
            if (exceptionHandler != null)
            {
                batchEventProcessor.setExceptionHandler(exceptionHandler);
            }
            consumerRepository.add(batchEventProcessor, eventHandler, barrier);
            processorSequences[i] = batchEventProcessor.getSequence();
        }
        updateGatingSequencesForNextInChain(barrierSequences, processorSequences);
        return new EventHandlerGroup<>(this, consumerRepository, processorSequences);
    }
    private void updateGatingSequencesForNextInChain(final Sequence[] barrierSequences, final Sequence[] processorSequences)
    {
        if (processorSequences.length > 0)
        {
            ringBuffer.addGatingSequences(processorSequences);
            for (final Sequence barrierSequence : barrierSequences)
            {
                ringBuffer.removeGatingSequence(barrierSequence);
            }
            consumerRepository.unMarkEventProcessorsAsEndOfChain(barrierSequences);
        }
    }
    EventHandlerGroup<T> createEventProcessors(
        final Sequence[] barrierSequences, final EventProcessorFactory<T>[] processorFactories)
    {
        final EventProcessor[] eventProcessors = new EventProcessor[processorFactories.length];
        for (int i = 0; i < processorFactories.length; i++)
        {
            eventProcessors[i] = processorFactories[i].createEventProcessor(ringBuffer, barrierSequences);
        }

        return handleEventsWith(eventProcessors);
    }
    EventHandlerGroup<T> createWorkerPool(
        final Sequence[] barrierSequences, final WorkHandler<? super T>[] workHandlers)
    {
        final SequenceBarrier sequenceBarrier = ringBuffer.newBarrier(barrierSequences);
        final WorkerPool<T> workerPool = new WorkerPool<>(ringBuffer, sequenceBarrier, exceptionHandler, workHandlers);
        consumerRepository.add(workerPool, sequenceBarrier);
        final Sequence[] workerSequences = workerPool.getWorkerSequences();
        updateGatingSequencesForNextInChain(barrierSequences, workerSequences);
        return new EventHandlerGroup<>(this, consumerRepository, workerSequences);
    }
 ```
 
  [![EHvRsA.md.png](https://s2.ax1x.com/2019/05/16/EHvRsA.md.png)](https://imgchr.com/i/EHvRsA)
