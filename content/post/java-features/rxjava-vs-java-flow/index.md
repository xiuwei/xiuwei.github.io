---
title: RxJava API 与 Java 9 Flow API 的区别
description: 认识 Reactive Stream，并了解它与 RxJava 和 Flow API的区别
slug: rxjava-vs-java-flow
date: 2021-12-12 18:00:00+0000
author: baeldung
image: cover.jpg
categories:
    - Java
    - Reactive
tags:
    - Java
    - Reactive Stream
    - RxJava
#weight: 1  
---

## 1.概述

Java Flow API 是在 Java 9 中作为 Reactive Stream 规范的实现引入的。

在这篇文章里，我们首先介绍响应式流，然后介绍它与RxJava和Flow API的关系。

## 2.什么是 Reactive Stream

[Reactive Manifesto](https://www.reactive-streams.org/) 引入了 Reactive Streams 来指定具有非阻塞背压的异步流处理的标准。

Reactive Stream 规范的范围是定义**一组最小的接口**来实现这些目标：

- **[org.reactivestreams.Publisher](http://www.reactive-streams.org/reactive-streams-1.0.0-javadoc/org/reactivestreams/Publisher.html)** 是一个数据提供者，根据订阅者的需求向订阅者发布数据
- **[org.reactivestreams.Subscriber](http://www.reactive-streams.org/reactive-streams-1.0.0-javadoc/org/reactivestreams/Subscriber.html)** 是数据的消费者——订阅发布者后可以接收数据
- **[org.reactivestreams.Subscription](http://www.reactive-streams.org/reactive-streams-1.0.0-javadoc/org/reactivestreams/Subscription.html)** 当发布者接受订阅者时创建
- **[org.reactivestreams.Processor](http://www.reactive-streams.org/reactive-streams-1.0.0-javadoc/org/reactivestreams/Processor.html)** 既是订阅者又是发布者 - 它订阅发布者，处理数据，然后将处理后的数据传递给订阅者

Flow API源自规范，RxJava早于它，但从2.0开始，RxJava也支持该规范。

我们将深入探讨两者，但首先，让我们看一个实际用例。

## 3.用例

在本教程中，我们使用直播视频服务作为我们的用例。与点播视频流相反，直播视频流不依赖于消费者。因此，服务器以自己的速度发布流，而适应是消费者的责任。

在最简单的形式中，我们的模型由一个视频流发布者和一个作为订阅者的视频播放器组成。

让我们实现VideoFrame作为我们的数据项：

```java
public class VideoFrame {

    /**
     * 编号
     */
    private long number;

    /**
     * 视频数据
     */
    private byte[] data;

    public VideoFrame(long number, byte[] data) {
        this.number = number;
        this.data = data;
    }

    public VideoFrame(long number) {
        this.number = number;
    }

    public VideoFrame() {
    }

    public long getNumber() {
        return number;
    }

    public void setNumber(long number) {
        this.number = number;
    }

    public byte[] getData() {
        return data;
    }

    public void setData(byte[] data) {
        this.data = data;
    }
}
```

## 4.Flow API 的实现

JDK 9 中的 Flow API 对应于 Reactive Streams 规范。使用 Flow API，如果应用程序最初请求 N 个项目，则发布者最多将 N 个项目推送给订阅者。

Flow API 接口均位于 java.util.concurrent.Flow 接口中。它们在语义上等同于各自的响应式流。

让我们将 VideoStreamServer 作为 VideoFrame 的发布者。

```java
public class VideoStreamServer extends SubmissionPublisher<VideoFrame> {

    public VideoStreamServer() {
        super(Executors.newSingleThreadExecutor(),5);
    }
}
```

我们从 SubmissionPublisher 扩展了 VideoStreamServer ，而不是直接实现 Flow::Publisher。 SubmissionPublisher 是 Fl​​ow::Publisher 的 JDK 实现，用于与订阅者进行异步通信，因此它让我们的 VideoStreamServer 按照自己的速度发送。

此外，它对于背压和缓冲区处理也很有帮助，因为当调用 SubmissionPublisher::subscribe 时，它​​会创建 BufferedSubscription 的实例，然后将新订阅添加到其订阅链中。 BufferedSubscription 可以缓冲已发布的项目，最高可达 SubmissionPublisher#maxBufferCapacity。

现在让我们定义 VideoPlayer，它消耗 VideoFrame 流。因此它必须实现 Flow::Subscriber。

```java
public class VideoPlayer implements Flow.Subscriber<VideoFrame>{

    Flow.Subscription subscription = null;

    @Override
    public void onSubscribe(Flow.Subscription subscription) {
        this.subscription = subscription;
        subscription.request(1);
    }

    @Override
    public void onNext(VideoFrame item) {
        System.out.println("播放 ： "+item.getNumber());
        subscription.request(1);
    }

    @Override
    public void onError(Throwable throwable) {
        throwable.printStackTrace();
    }

    @Override
    public void onComplete() {
        System.out.println("播放完了！");
    }
}
```

VideoPlayer 订阅 VideoStreamServer ，订阅成功后调用 VideoPlayer::onSubscribe方法，请求一帧。 VideoPlayer::onNext 接收帧并请求新帧。请求的帧的数量取决于用例和订阅者实现。

最后，我们把直播场景串起来：

```java
try (VideoStreamServer streamServer = new VideoStreamServer()) {
    streamServer.subscribe(new VideoPlayer());
    ScheduledExecutorService executorService = Executors.newScheduledThreadPool(1);
    AtomicLong frameNumber = new AtomicLong();
    executorService.scheduleAtFixedRate(() -> {
        streamServer.offer(new VideoFrame(frameNumber.incrementAndGet()),(subscriber, videoFrame) -> {
            subscriber.onError(new RuntimeException("Frame#： " + videoFrame.getNumber()
                    + " 因触发背压被丢弃！"));
            return true;
        });
    }, 0, 1, TimeUnit.MILLISECONDS);

    Thread.sleep(3000);
} catch (Exception e) {
    throw new RuntimeException(e);
}
```

## 5.使用 RxJava 实现

RxJava 是 ReactiveX 的 Java 实现。 ReactiveX（或 Reactive Extensions）项目旨在提供反应式编程概念。它是观察者模式、迭代器模式和函数式编程的组合。

RxJava 的最新版本是 3.x。 RxJava 从 2.x 版本开始就支持 Reactive Streams 及其 Flowable 基类，但它比 Reactive Streams 更重要，它有几个基类，如 Flowable、Observable、Single、Completable。

Flowable 从 Reactive Streams 扩展了 Publisher。因此，许多 RxJava 运算符直接接受 Publisher 并允许与其他 Reactive Streams 实现直接互操作。

首先，创建一个无限延迟流的视频流生成器：

```java
Stream<VideoFrame> videoStream = Stream.iterate(new VideoFrame(0), videoFrame -> {
    return new VideoFrame(videoFrame.getNumber() + 1);
});
```

然后我们定义一个 Flowable 实例来在单独的线程上生成帧：

```java
Flowable
  .fromStream(videoStream)
  .subscribeOn(Schedulers.from(Executors.newSingleThreadExecutor()))
```

需要注意的是，无限流对我们来说已经足够了，但是如果我们需要更灵活的方式来生成流，那么 Flowable.create 是一个不错的选择。

```java
Flowable
  .create(new FlowableOnSubscribe<VideoFrame>() {
      AtomicLong frame = new AtomicLong();
      @Override
      public void subscribe(@NonNull FlowableEmitter<VideoFrame> emitter) {
          while (true) {
              emitter.onNext(new VideoFrame(frame.incrementAndGet()));
              //sleep for 1 ms to simualte delay
          }
      }
  }, /* Set Backpressure Strategy Here */)
```

然后，在下一步中，VideoPlayer 订阅此 Flowable 并观察单独线程上的项目。

```java
videoFlowable
  .observeOn(Schedulers.from(Executors.newSingleThreadExecutor()))
  .subscribe(item -> {
      log.info("play #" + item.getNumber());
      // sleep for 30 ms to simualate frame display
  });
```

最后，我们将配置背压策略。如果我们想在帧丢失的情况下停止视频，因此我们必须在缓冲区已满时使用 BackPressureOverflowStrategy::ERROR。

```java
Flowable
  .fromStream(videoStream)
  .subscribeOn(Schedulers.from(Executors.newSingleThreadExecutor()))
  .onBackpressureBuffer(5, null, BackpressureOverflowStrategy.ERROR)
  .observeOn(Schedulers.from(Executors.newSingleThreadExecutor()))
  .subscribe(item -> {
       log.info("play #" + item.getNumber()); 
       // sleep for 30 ms to simualate frame display 
  });
```

## 6.RxJava 和 Flow API 的比较

即使在这两个简单的实现中，我们也可以看到 RxJava 的 API 是多么丰富，特别是在缓冲区管理、错误处理和反压策略方面。它以其流畅的 API 为我们提供了更多的选择和更少的代码行。现在让我们考虑更复杂的情况。

假设我们的播放器在没有编解码器的情况下无法显示视频帧。因此，对于 Flow API，我们需要实现一个处理器来模拟编解码器并位于服务器和播放器之间。使用 RxJava，我们可以使用 Flowable::flatMap 或 Flowable::map 来完成。

或者让我们想象一下，我们的播放器还将播放实时翻译音频，因此我们必须合并来自不同发布商的视频和音频流。有了RxJava，我们可以使用Flowable::combineLatest，但是有了Flow API，这并不是一件容易的事。

尽管如此，可以编写一个自定义处理器来订阅两个流并将组合数据发送到我们的视频播放器。然而，实施起来却很令人头疼。

## 7.为什么使用 Flow API？

说到这里，我们可能会有一个疑问，Flow API 背后的理念是什么？

如果我们在JDK中搜索Flow API的用法，我们可以在java.net.http和jdk.internal.net.http中找到一些东西。

此外，我们可以在reactor项目或 reactive stream 包中找到适配器。例如，org.reactivestreams.FlowAdapters 具有将 Flow API 接口转换为 Reactive Stream 接口的方法，反之亦然。因此，它有助于 Flow API 和具有反应式流支持的库之间的互操作性。

**所有这些事实都有助于我们理解 Flow API 的目的：它被创建为 JDK 中的一组反应式规范接口，无需依赖第三方。** 此外，Java 期望 Flow API 被接受为反应式规范的标准接口，并在 JDK 或其他为中间件和实用程序实现反应式规范的基于 Java 的库中使用。


## 参考资料

- [原文链接](https://www.baeldung.com/rxjava-vs-java-flow-api)
- [示例源码](https://github.com/eugenp/tutorials/tree/master/core-java-modules/core-java-9-new-features)