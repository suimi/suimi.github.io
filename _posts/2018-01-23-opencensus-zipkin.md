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
### 初始化过程
1. TraceComponent初始化,Tracing初始化时就会创建静态常量对象`traceComponent`,默认实例为`TraceComponentImpl`,不能存在时`io.opencensus.impllite.trace.TraceComponentImplLite`
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

2. TraceComponentImpl初始化
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
3. export组件
```
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
其中会初始化SpanExporter

4. **SpanExporter**
   SpanExporter是工作的核心，初始化时开启worker线程，检查span列表并以批次方式导出数据
```
private SpanExporterImpl(Worker worker) {
    this.workerThread =
        new DaemonThreadFactory("ExportComponent.ServiceExporterThread").newThread(worker);
    this.workerThread.start();
    this.worker = worker;
  }
```
```
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

5. start end 事件处理器
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

```
private static final class DisruptorEventHandler implements EventHandler<DisruptorEvent> {
    @Override
    // TODO(sebright): Fix the Checker Framework warning.
    @SuppressWarnings("nullness")
    public void onEvent(DisruptorEvent event, long sequence, boolean endOfBatch) {
      event.getEntry().process();
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
```
git clone git@github.com:census-instrumentation/opencensus-website.git
cd opencensus-website
```
2. 启动服务
```
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
- [PowerDesigner CDM LDM PDM OOM](http://blog.csdn.net/u010924834/article/details/48531669)
- [PowerDesigner 系列文章](http://www.cnblogs.com/sandea/p/4318540.html)
