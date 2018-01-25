---
layout: post
title: OpenCensus Zipkin 源码解析
tags: OpenCensus
categories: 监控
---
* TOC
{:toc}

# OpenCensus 介绍
谷歌开源了OpenCensus，是一个厂商中立的开放源码库，用于度量收集和跟踪。OpenCensus的构建是为了增加最小的开销，并部署在整个团队中，特别是基于微服务的架构。

# OpenCensus Zipkin Trace Exporter
## 启动spring boot zipkin(2.4.5)
```xml
        <dependency>
            <groupId>io.zipkin.java</groupId>
            <artifactId>zipkin-server</artifactId>
        </dependency>
        <dependency>
            <groupId>io.zipkin.java</groupId>
            <artifactId>zipkin-autoconfigure-ui</artifactId>
        </dependency>
```
## 集成OpenCensus Zipkin
 - pom
    ``` xml
            <dependency>
                <groupId>io.opencensus</groupId>
                <artifactId>opencensus-api</artifactId>
                <version>0.11.1</version>
            </dependency>
            <dependency>
                <groupId>io.opencensus</groupId>
                <artifactId>opencensus-exporter-trace-logging</artifactId>
                <version>0.11.1</version>
            </dependency>
            <dependency>
                <groupId>io.opencensus</groupId>
                <artifactId>opencensus-exporter-trace-zipkin</artifactId>
                <version>0.11.1</version>
            </dependency>
            <dependency>
                <groupId>io.opencensus</groupId>
                <artifactId>opencensus-impl</artifactId>
                <version>0.11.1</version>
                <scope>runtime</scope>
            </dependency>
    ```

  - register exporter
  ``` java
    @SpringBootApplication
    public class OpencensusDemoApplication {

    public static void main(String[] args) {
        LoggingExporter.register();
        ZipkinExporter.createAndRegister("http://127.0.0.1:8883/api/v2/spans", "Test");
        SpringApplication.run(OpencensusDemoApplication.class, args);
    }
}
  ```

 - 开启span
 ``` java
     @GetMapping("test")
     public String test(String value) {

         LoggingExporter.register();

         Span span = tracer.spanBuilder("test").setRecordEvents(true).setSampler(Samplers.alwaysSample()).startSpan();
         span.addAnnotation("Annotation to the root Span before child is created.");
         log.info("test value:{}", value);
         test();
         span.addAnnotation("Annotation to the root Span after child is ended.");
         span.end();
         return value;
     }
 ```
 注意事项：如果需要发送到zipkin，需要`setRecordEvents(true)`同时`setSampler(Samplers.alwaysSample())`

## 源码解析
整体而言采用开源的并发框架 Disruptor,在性能上优于zipkin自身提供的 reporter

### 初始化过程
1. TraceComponent初始化,Tracing初始化时就会创建静态常量对象`traceComponent`,默认实例为`TraceComponentImpl`,不能存在时`TraceComponentImplLite`

    ``` java
    public final class Tracing {
      private static final Logger logger = Logger.getLogger(Tracing.class.getName());
      private static final TraceComponent traceComponent =
          loadTraceComponent(TraceComponent.class.getClassLoader());

      @VisibleForTesting
        static TraceComponent loadTraceComponent(@Nullable ClassLoader classLoader) {
          try {
            // Call Class.forName with literal string name of the class to help shading tools.
            return Provider.createInstance(
                Class.forName("io.opencensus.impl.trace.TraceComponentImpl", true, classLoader),
                TraceComponent.class);
          } catch (ClassNotFoundException e) {
            logger.log(
                Level.FINE,
                "Couldn't load full implementation for TraceComponent, now trying to load lite "
                    + "implementation.",
                e);
          }
          try {
            // Call Class.forName with literal string name of the class to help shading tools.
            return Provider.createInstance(
                Class.forName("io.opencensus.impllite.trace.TraceComponentImplLite", true, classLoader),
                TraceComponent.class);
          } catch (ClassNotFoundException e) {
            logger.log(
                Level.FINE,
                "Couldn't load lite implementation for TraceComponent, now using "
                    + "default implementation for TraceComponent.",
                e);
          }
          return TraceComponent.newNoopTraceComponent();
        }
    }
    ```

2. TraceComponentImpl初始化, 采用disruptor queue来临时存储span, 同时会初始化 `ExportComponent` 和 `StartEndHandler`

    ``` java
    public TraceComponentImpl() {
        traceComponentImplBase =
            // MillisClock.getInstance()时钟
            // RandomHandler trace,span id 随机生成器
            // eventQueue 事件队列，这里采用的是DisruptorEventQueue，是采用并发框架Disruptor实现
            new TraceComponentImplBase(MillisClock.getInstance(),new ThreadLocalRandomHandler(),DisruptorEventQueue.getInstance());
      }
    ```
    ``` java
    public TraceComponentImplBase(Clock clock, RandomHandler randomHandler, EventQueue eventQueue) {
        this.clock = clock;
        // TODO(bdrutu): Add a config/argument for supportInProcessStores.
        //export 组件初始化
        if (eventQueue instanceof SimpleEventQueue) {
          exportComponent = ExportComponentImpl.createWithoutInProcessStores(eventQueue);
        } else {
          exportComponent = ExportComponentImpl.createWithInProcessStores(eventQueue);
        }
        StartEndHandler startEndHandler =
            new StartEndHandlerImpl(exportComponent.getSpanExporter(),exportComponent.getRunningSpanStore(),
                exportComponent.getSampledSpanStore(),eventQueue);
        tracer = new TracerImpl(randomHandler, startEndHandler, clock, traceConfig);
      }
    ```

3. `ExportComponent` 初始化过程创建 `SpanExporter`

    ```java
    public final class ExportComponentImpl extends ExportComponent {

      public static ExportComponentImpl createWithInProcessStores(EventQueue eventQueue) {
        return new ExportComponentImpl(true, eventQueue);
      }

      /**
       * Returns a new {@code ExportComponentImpl} that has {@code null} instances for {@link
       * RunningSpanStore} and {@link SampledSpanStore}.
       *
       * @return a new {@code ExportComponentImpl}.
       */
      public static ExportComponentImpl createWithoutInProcessStores(EventQueue eventQueue) {
        return new ExportComponentImpl(false, eventQueue);
      }

     private ExportComponentImpl(boolean supportInProcessStores, EventQueue eventQueue) {
        this.spanExporter = SpanExporterImpl.create(EXPORTER_BUFFER_SIZE, EXPORTER_SCHEDULE_DELAY);
        this.runningSpanStore = supportInProcessStores ? new RunningSpanStoreImpl() : null;
        this.sampledSpanStore = supportInProcessStores ? new SampledSpanStoreImpl(eventQueue) : null;
      }
    }

    ```

4. **SpanExporter**
   SpanExporter是工作的核心，初始化时开启worker线程，检查span列表并以批次方式导出数据
    ```java
    private SpanExporterImpl(Worker worker) {
        this.workerThread =
            new DaemonThreadFactory("ExportComponent.ServiceExporterThread").newThread(worker);
        this.workerThread.start();
        this.worker = worker;
      }
    ```
    ```java
    private static final class Worker implements Runnable {
        private final Object monitor = new Object();

        @GuardedBy("monitor")
        private final List<SpanImpl> spans;
        // See SpanExporterImpl#addSpan.
        private void addSpan(SpanImpl span) {
          synchronized (monitor) {
            this.spans.add(span);
            if (spans.size() > bufferSize) {
              monitor.notifyAll();
            }
          }
        }

        @Override
        public void run() {
          while (true) {
            // Copy all the batched spans in a separate list to release the monitor lock asap to
            // avoid blocking the producer thread.
            List<SpanImpl> spansCopy;
            synchronized (monitor) {
              if (spans.size() < bufferSize) {
                do {
                  // In the case of a spurious wakeup we export only if we have at least one span in
                  // the batch. It is acceptable because batching is a best effort mechanism here.
                  try {
                    monitor.wait(scheduleDelayMillis);
                  } catch (InterruptedException ie) {
                    // Preserve the interruption status as per guidance and stop doing any work.
                    Thread.currentThread().interrupt();
                    return;
                  }
                } while (spans.isEmpty());
              }
              spansCopy = new ArrayList<SpanImpl>(spans);
              spans.clear();
            }
            // Execute the batch export outside the synchronized to not block all producers.
            final List<SpanData> spanDataList = fromSpanImplToSpanData(spansCopy);
            if (!spanDataList.isEmpty()) {
              onBatchExport(spanDataList);
            }
          }
        }
      }
    ```

5. `StartEndHandler`主要用于在事件发生时，调用Queue发布事件到`Disruptor`组件

    ``` java
    public StartEndHandlerImpl(
          SpanExporterImpl spanExporter,
          @Nullable RunningSpanStoreImpl runningSpanStore,
          @Nullable SampledSpanStoreImpl sampledSpanStore,
          EventQueue eventQueue) {
        this.spanExporter = spanExporter;
        this.runningSpanStore = runningSpanStore;
        this.sampledSpanStore = sampledSpanStore;
        this.enqueueEventForNonSampledSpans = runningSpanStore != null || sampledSpanStore != null;
        this.eventQueue = eventQueue;
      }

        // 调用Queue发布start事件
      @Override
      public void onStart(SpanImpl span) {
        if (span.getOptions().contains(Options.RECORD_EVENTS) && enqueueEventForNonSampledSpans) {
          eventQueue.enqueue(new SpanStartEvent(span, runningSpanStore));
        }
      }

        // 调用Queue发布end事件
      @Override
      public void onEnd(SpanImpl span) {
        if ((span.getOptions().contains(Options.RECORD_EVENTS) && enqueueEventForNonSampledSpans)
            || span.getContext().getTraceOptions().isSampled()) {
          eventQueue.enqueue(new SpanEndEvent(span, spanExporter, runningSpanStore, sampledSpanStore));
        }
      }
    ```
6. DisruptorEventQueue, 初始化时配置事件处理器`disruptor.handleEventsWith(new DisruptorEventHandler())`

    ``` java
    @ThreadSafe
    public final class DisruptorEventQueue implements EventQueue {
      // Number of events that can be enqueued at any one time. If more than this are enqueued,
      // then subsequent attempts to enqueue new entries will block.
      // TODO(aveitch): consider making this a parameter to the constructor, so the queue can be
      // configured to a size appropriate to the system (smaller/less busy systems will not need as
      // large a queue.
      private static final int DISRUPTOR_BUFFER_SIZE = 8192;
      // The single instance of the class.
      private static final DisruptorEventQueue eventQueue = new DisruptorEventQueue();

      // The event queue is built on this {@link Disruptor}.
      private final Disruptor<DisruptorEvent> disruptor;
      // Ring Buffer for the {@link Disruptor} that underlies the queue.
      private final RingBuffer<DisruptorEvent> ringBuffer;

      // Creates a new EventQueue. Private to prevent creation of non-singleton instance.
      // Suppress warnings for disruptor.handleEventsWith and Disruptor constructor
      @SuppressWarnings({"deprecation", "unchecked", "varargs"})
      private DisruptorEventQueue() {
        // Create new Disruptor for processing. Note that this uses a single thread for processing; this
        // ensures that the event handler can take unsynchronized actions whenever possible.
        disruptor =
            new Disruptor<DisruptorEvent>(
                new DisruptorEventFactory(),
                DISRUPTOR_BUFFER_SIZE,
                Executors.newSingleThreadExecutor(new DaemonThreadFactory("OpenCensus.Disruptor")),
                ProducerType.MULTI,
                new SleepingWaitStrategy());
        disruptor.handleEventsWith(new DisruptorEventHandler());
        disruptor.start();
        ringBuffer = disruptor.getRingBuffer();
      }

       // 发布事件
      @Override
      public void enqueue(Entry entry) {
        long sequence = ringBuffer.next();
        try {
          DisruptorEvent event = ringBuffer.get(sequence);
          event.setEntry(entry);
        } finally {
          ringBuffer.publish(sequence);
        }
      }
    ```

    `DisruptorEventHandler` 调用event的process方法处理，例如`SpanEndEvent.process()`，end event process将span加入到`SpanExporter`的处理列表

    ```java
    private static final class DisruptorEventHandler implements EventHandler<DisruptorEvent> {
        @Override
        // TODO(sebright): Fix the Checker Framework warning.
        @SuppressWarnings("nullness")
        public void onEvent(DisruptorEvent event, long sequence, boolean endOfBatch) {
          event.getEntry().process();
        }
      }
    ```

## 与zipkin原生reporter比较
zipkin 原生reporter同样是采用缓存分组发送机制，存储同样采用自旋队列的方式，但会有对象创建内存开销的差异
1. `ZipkinAutoConfiguration` Auto Config
    ```java
    @Bean
        @ConditionalOnMissingBean
        public ZipkinSpanReporter reporter(SpanMetricReporter spanMetricReporter, ZipkinProperties zipkin,
                ZipkinRestTemplateCustomizer zipkinRestTemplateCustomizer) {
            RestTemplate restTemplate = new RestTemplate();
            zipkinRestTemplateCustomizer.customize(restTemplate);
            return new HttpZipkinSpanReporter(restTemplate, zipkin.getBaseUrl(), zipkin.getFlushInterval(),
                    spanMetricReporter);
        }
    ```
    ```
    public HttpZipkinSpanReporter(RestTemplate restTemplate, String baseUrl, int flushInterval,
                SpanMetricReporter spanMetricReporter) {
            this.sender = new RestTemplateSender(restTemplate, baseUrl);
            this.delegate = AsyncReporter.builder(this.sender)
                    .queuedMaxSpans(1000) // historical constraint. Note: AsyncReporter supports memory bounds
                    .messageTimeout(flushInterval, TimeUnit.SECONDS)
                    .metrics(new ReporterMetricsAdapter(spanMetricReporter))
                    .build();
        }
    ```

2. 异步Reporter `AsyncReporter`

    ```java
    public abstract class AsyncReporter<S> implements Reporter<S>, Flushable, Component {
        /** Builds an async reporter that encodes zipkin spans as they are reported. */
        public AsyncReporter<Span> build() {
          switch (sender.encoding()) {
            case JSON:
              return build(Encoder.JSON);
            case THRIFT:
              return build(Encoder.THRIFT);
            default:
              throw new UnsupportedOperationException(sender.encoding().name());
          }
        }

        /** Builds an async reporter that encodes arbitrary spans as they are reported. */
        public <S> AsyncReporter<S> build(Encoder<S> encoder) {
          checkNotNull(encoder, "encoder");
          checkArgument(encoder.encoding() == sender.encoding(),
              "Encoder.encoding() %s != Sender.encoding() %s",
              encoder.encoding(), sender.encoding());

          // 异步Reporter
          final BoundedAsyncReporter<S> result = new BoundedAsyncReporter<>(this, encoder);

          if (messageTimeoutNanos > 0) { // Start a thread that flushes the queue in a loop.
            final BufferNextMessage consumer =
                new BufferNextMessage(sender, messageMaxBytes, messageTimeoutNanos);
            final Thread flushThread = new Thread(() -> {
              try {
                while (!result.closed.get()) {
                  result.flush(consumer);
                }
              } finally {
                for (byte[] next : consumer.drain()) result.pending.offer(next);
                result.close.countDown();
              }
            }, "AsyncReporter(" + sender + ")");
            flushThread.setDaemon(true);
            //启动线程，异步发送span
            flushThread.start();
          }
          return result;
        }

        static final class BoundedAsyncReporter<S> extends AsyncReporter<S> {
            static final Logger logger = Logger.getLogger(BoundedAsyncReporter.class.getName());
            final AtomicBoolean closed = new AtomicBoolean(false);
            final Encoder<S> encoder;
            final ByteBoundedQueue pending;
            final Sender sender;
            final int messageMaxBytes;
            final long messageTimeoutNanos;
            final CountDownLatch close;
            final ReporterMetrics metrics;

            BoundedAsyncReporter(Builder builder, Encoder<S> encoder) {

              //有界队列，是一个多生产者，多消费者队列
              this.pending = new ByteBoundedQueue(builder.queuedMaxSpans, builder.queuedMaxBytes);
              this.sender = builder.sender;
              this.messageMaxBytes = builder.messageMaxBytes;
              this.messageTimeoutNanos = builder.messageTimeoutNanos;
              this.close = new CountDownLatch(builder.messageTimeoutNanos > 0 ? 1 : 0);
              this.metrics = builder.metrics;
              this.encoder = encoder;
            }

            /** Returns true if the was encoded and accepted onto the queue. */
            //接收span,编码并放入队列
            @Override
            public void report(S span) {
              checkNotNull(span, "span");
              metrics.incrementSpans(1);
              byte[] next = encoder.encode(span);
              int messageSizeOfNextSpan = sender.messageSizeInBytes(Collections.singletonList(next));
              metrics.incrementSpanBytes(next.length);
              if (closed.get() ||
                  // don't enqueue something larger than we can drain
                  messageSizeOfNextSpan > messageMaxBytes ||
                  !pending.offer(next)) {
                metrics.incrementSpansDropped(1);
              }
            }

            @Override
            public final void flush() {
              flush(new BufferNextMessage(sender, messageMaxBytes, 0));
            }
            //刷新队列，读取队列所有span放入 message buffer,并发送
            void flush(BufferNextMessage bundler) {
              if (closed.get()) throw new IllegalStateException("closed");
             // 这里读取队列消息到bundler做了一次buffer.add()操作
              pending.drainTo(bundler, bundler.remainingNanos());

              // record after flushing reduces the amount of gauge events vs on doing this on report
              metrics.updateQueuedSpans(pending.count);
              metrics.updateQueuedBytes(pending.sizeInBytes);

              if (!bundler.isReady()) return; // try to fill up the bundle

              // Signal that we are about to send a message of a known size in bytes
              metrics.incrementMessages();
              metrics.incrementMessageBytes(bundler.sizeInBytes());
              //获取下一组数据，并清空buffer，这里有一定开销
              List<byte[]> nextMessage = bundler.drain();

              // In failure case, we increment messages and spans dropped.
              Callback failureCallback = sendSpansCallback(nextMessage.size());
              try {
                sender.sendSpans(nextMessage, failureCallback);
              } catch (RuntimeException e) {
                failureCallback.onError(e);
                // Raise in case the sender was closed out-of-band.
                if (e instanceof IllegalStateException) throw e;
              }
            }
      }
    }
    ```


# opencensus-website
## Hugo
  - Hugo是由Go语言实现的静态网站生成器。简单、易用、高效、易扩展、快速部署。
  - 官网：http://www.gohugo.org/
  - 下载地址: https://github.com/gohugoio/hugo/releases
  - 下载后配置环境变量

## 启动opencensus-website
1. 获取代码
```shell
git clone git@github.com:census-instrumentation/opencensus-website.git
cd opencensus-website
```
2. 启动服务

```shell
$ hugo serve

[K25lBuilding sites … [?25h
                   | EN
+------------------+----+
  Pages            |  8
  Paginator pages  |  0
  Non-page files   |  0
  Static files     | 10
  Processed images |  0
  Aliases          |  0
  Sitemaps         |  1
  Cleaned          |  0

Total in 87 ms
Watching for changes in E:\workspace\opencensus-website\{content,static,themes}
Serving pages from memory
Running in Fast Render Mode. For full rebuilds on change: hugo server --disableFastRender
Web Server is available at http://localhost:1313/ (bind address 127.0.0.1)
Press Ctrl+C to stop
```

参考:
- [谷歌开源发布opencensus](http://www.atyun.com/15317_%E8%B0%B7%E6%AD%8C%E5%BC%80%E6%BA%90%E5%8F%91%E5%B8%83opencensus%E4%B8%80%E4%B8%AA%E7%BB%9F%E8%AE%A1%E6%95%B0%E6%8D%AE%E6%94%B6%E9%9B%86%E5%92%8C%E5%88%86%E5%B8%83%E5%BC%8F%E8%B7%9F%E8%B8%AA%E6%A1%86.html)
- [Github](https://github.com/census-instrumentation/opencensus-java/blob/master/exporters/trace/zipkin/README.md)
