---
title: RxJava API 与 Java 9 Flow API 的区别
description: 认识 Reactive Stream，并了解它与 RxJava 和 Flow API的区别
slug: rxjava-vs-java-flow
date: 2021-12-12 18:00:00+0000
author: baeldung
#image: cover-java.jpg
categories:
    - Java
    - Reactive
tags:
    - Java
    - Reactive Stream
    - RxJava
weight: 1  
---

## 1.概述

Java Flow API 是在 Java 9 中作为 Reactive Stream 规范的实现引入的。

在这篇文章里，我们首先研究 Reactive Stream，然后再了解它与 RxJava 和 Flow API的区别。

## 2.Reactive Stream 是什么鬼?

[Reactive Manifesto](https://www.reactive-streams.org/) 引入了 Reactive Streams 来指定具有非阻塞背压的异步流处理的标准。

Reactive Stream 规范的范围是定义**一组最小的接口**来实现这些目标：

- **[org.reactivestreams.Publisher](http://www.reactive-streams.org/reactive-streams-1.0.0-javadoc/org/reactivestreams/Publisher.html)** 是一个数据提供者，根据订阅者的需求向订阅者发布数据
- **[org.reactivestreams.Subscriber](http://www.reactive-streams.org/reactive-streams-1.0.0-javadoc/org/reactivestreams/Subscriber.html)** 是数据的消费者——订阅发布者后可以接收数据
- **[org.reactivestreams.Subscription](http://www.reactive-streams.org/reactive-streams-1.0.0-javadoc/org/reactivestreams/Subscription.html)** 当发布者接受订阅者时创建
- **[org.reactivestreams.Processor](http://www.reactive-streams.org/reactive-streams-1.0.0-javadoc/org/reactivestreams/Processor.html)** 既是订阅者又是发布者 - 它订阅发布者，处理数据，然后将处理后的数据传递给订阅者

Flow API源自规范。 RxJava 领先于它，但从 2.0 开始，RxJava 也支持该规范。

我们将深入探讨两者，但首先让我们看一个实际用例。

## 3.用例

我们用抖音的网红直播作为用例。

与抖音点播不同，视频直播不依赖于观众，因此，网红按自己的节奏发布视频流，而观众需要有适合的网络带宽来适应网红的直播流推送。

我们将用最简洁的代码，演示网红、观众、直播流之间的协作，并分别演示 Flow API 和 RxJava 是如何实现的。

让我们先定义“直播流“ DouyinStream 的代码，作为我们的数据项。

```java
public class DouyinStream {

    /**
     * 抖音编号
     */
    private long number;

    /**
     * 视频数据
     */
    private byte[] data;

    public DouyinStream(long number, byte[] data) {
        this.number = number;
        this.data = data;
    }

    public DouyinStream(long number) {
        this.number = number;
    }

    public DouyinStream() {
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

让我们实现“网红” DouyinInfluencer 作为 DouyinStream 的发布者。

```java
public class DouyinInfluencer extends SubmissionPublisher<DouyinStream> {

    public DouyinInfluencer() {
        super(Executors.newSingleThreadExecutor(),5);
    }
}
```

我们从 SubmissionPublisher 扩展了 DouyinInfluencer，而不是直接实现 Flow::Publisher。 SubmissionPublisher 是 Fl​​ow::Publisher 的 JDK 实现，用于与订阅者进行异步通信，因此它让我们的 DouyinInfluencer 按照自己的节奏发送视频流。

此外，它对于背压和缓冲区处理也很有帮助，因为当调用 SubmissionPublisher::subscribe 时，它​​会创建 BufferedSubscription 的实例，然后将新订阅添加到其订阅链中。 BufferedSubscription 可以缓冲已发布的项目，最高可达 SubmissionPublisher#maxBufferCapacity。

现在让我们定义“观众” VideoPlayer，它消耗 VideoFrame 流。因此它必须实现 Flow::Subscriber。

```java
public class DouyinAudience implements Flow.Subscriber<DouyinStream>{

    Flow.Subscription subscription = null;

    @Override
    public void onSubscribe(Flow.Subscription subscription) {
        this.subscription = subscription;
        subscription.request(1);
    }

    @Override
    public void onNext(DouyinStream item) {
        System.out.println("接收 直播流（id)）："+item.getNumber());
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

DouyinAudience订阅DouyinInfluencer，订阅成功后调用DouyinAudience::onSubscribe方法，请求一帧。 DouyinAudience::onNext 接收帧并请求新帧。请求的帧的数量取决于用例和订阅者实现。

最后，我们把直播场景串起来：

```java
try (DouyinInfluencer influencer = new DouyinInfluencer()) {
    influencer.subscribe(new DouyinAudience());
    ScheduledExecutorService executorService = Executors.newScheduledThreadPool(1);
    AtomicLong douyinNumber = new AtomicLong();
    executorService.scheduleAtFixedRate(() -> {
        influencer.offer(new DouyinStream(douyinNumber.incrementAndGet()),(subscriber, douyinStream) -> {
            subscriber.onError(new RuntimeException("直播流 id： " + douyinStream.getNumber()
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

RxJava 的最新版本是 3.x。 RxJava 从 2.x 版本开始就支持 Reactive Streams 及其 Flowable 基类，但它比 Reactive Streams 更重要，它有几个基类，如 Flowable、Observable、Single、Completable。

Flowable 作为反应流合规组件是具有反压处理的 0 到 N 项的流。 Flowable 从 Reactive Streams 扩展了 Publisher。因此，许多 RxJava 运算符直接接受 Publisher 并允许与其他 Reactive Streams 实现直接互操作。

```java
//step1:制作一个无限延迟流的视频流生成器
Stream<DouyinStream> videoStream = Stream.iterate(new DouyinStream(0), douyinStream -> {
    return new DouyinStream(douyinStream.getNumber() + 1);
});

//step2:创建一个抖音网红实例，并设置其被订阅后的视频流数据推送逻辑
Flowable<DouyinStream> douyinInfluencer = Flowable.create(new FlowableOnSubscribe<DouyinStream>() {

    @Override
    public void subscribe(@NonNull FlowableEmitter<DouyinStream> emitter) throws InterruptedException {

        Iterator<DouyinStream> iterator = videoStream.iterator();
        while (iterator.hasNext()) {
            emitter.onNext(iterator.next());
            Thread.sleep(500);
        }
    }
}, BackpressureStrategy.ERROR);

//setp3:编写观众点播网红直播，并设置观众对视频流数据的处理逻辑
douyinInfluencer
        .observeOn(Schedulers.from(Executors.newSingleThreadExecutor()))
        .onBackpressureDrop(douyinStream -> {
            System.out.println("丢弃第" + douyinStream.getNumber() + "帧视频流");
        })
        .subscribe(item -> {
            System.out.println("播放 第" + item.getNumber() + "帧视频流");
            Thread.sleep(500);
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
