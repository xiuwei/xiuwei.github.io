---
title: Java 9 Reactive Streams
description: Reactive Streams 是具有非阻塞背压的异步流处理标准
slug: java-9-reactive-streams
date: 2021-12-12 12:00:00+0000
author: baeldung
image: cover.jpg
categories:
    - Java
    - Reactive
tags:
    - Java
    - Reactive
    - 反应式流
weight: 1  
---

## 1.概述

在本文中，我们将研究 Java 9 反应式流。简而言之，我们将能够使用 Flow 类，它包含用于构建反应式流处理逻辑的主要构建块。

Reactive Streams 是具有非阻塞背压的异步流处理标准。该规范在 [Reactive Manifesto](http://www.reactive-streams.org/) 中定义，并且有多种实现，例如 RxJava 或 Akka-Streams。

## 2.Reactive API 概览

为了构建 *Flow* ，我们可以使用三个主要抽象并将它们组合成异步处理逻辑。

**每个 *Flow* 都需要处理 *Publisher* 实例向其发布的事件；** 发布者有一种方法——subscribe()。

**消息的接收者需要实现 *Subscriber* 接口。** 通常，这是每个流处理的结束，因为它的实例不会进一步发送消息。
我们可以将订阅者视为一个接收器。它有四个需要重写的方法 - *onSubscribe()*、*onNext()*、*onError()* 和 *onComplete()*。我们将在下一节中讨论这些内容。

**如果我们想转换传入的消息并将其进一步传递给下一个*Subscriber*，我们需要实现 *Processor* 接口。** 它既充当订阅者，因为它接收消息，又充当发布者，因为它处理这些消息并将其发送以进行进一步处理。

## 3.发布和消费消息

假设我们要创建一个简单的 *Flow*，其中一个发布消息的 *Publisher* 和一个简单的 *Subscriber* 在消息到达时消费消息 - 一次一个。

让我们创建一个 *EndSubscriber* 类。我们需要实现 *Subscriber* 接口。接下来，我们将重写所需方法。

*onSubscribe()* 方法在处理开始之前调用。*Subscription* (订阅)的实例作为参数传递。它是一个用于控制 *Subscriber* 和 *Publisher* 之间的消息流的类：

```java
public class MySubscriber<T> implements Flow.Subscriber<T> {
    private Flow.Subscription subscription;
    public List<T> consumedElements = new ArrayList<>();


    @Override
    public void onSubscribe(Flow.Subscription subscription) {
        this.subscription = subscription;
        subscription.request(1);
    }
}
```

我们还初始化了一个空的 ConsumerElements 列表，它将在测试中使用。

现在，我们需要实现 Subscriber 接口的其余方法。这里的主要方法是 onNext() – 每当Publisher发布新消息时都会调用该方法：

```java
@Override
public void onNext(T item) {
    System.out.println("接收到数据 ： "+item);
    consumedElements.add(item);
    subscription.request(1);
}
```

请注意，当我们在 onSubscribe() 方法中启动订阅并处理消息时，我们需要调用订阅上的 request() 方法来表明当前Subscriber已准备好消费更多消息。

最后，我们需要实现 onError() ——每当处理过程中抛出异常时就会调用它，以及 onComplete() ——当Publisher关闭时调用：

```java
@Override
public void onError(Throwable throwable) {
    throwable.printStackTrace();
}

@Override
public void onComplete() {
    System.out.println("MySubscriber Complete!");
}
```

让我们为处理流程编写一个测试。我们将使用 SubmissionPublisher 类——一个来自 java.util.concurrent 的构造——它实现了 Publisher 接口。

我们将向Publisher提交 N 个元素——我们的最终Subscriber将收到这些元素：

```java
@Test
public void whenSubscribeToIt_thenShouldConsumeAll()
        throws InterruptedException {

    SubmissionPublisher<String> publisher = new SubmissionPublisher<>();
    MySubscriber<String> subscriber = new MySubscriber<>();
    publisher.subscribe(subscriber);
    List<String> items = List.of("1", "x", "2", "x", "3", "x");

    assertEquals(1, publisher.getNumberOfSubscribers());
    items.forEach(publisher::submit);
    publisher.close();

    await().atMost(1000, TimeUnit.MILLISECONDS)
            .until(
                    () -> subscriber.consumedElements.size() == items.size()
            );
}
```

请注意，我们正在 EndSubscriber 实例上调用 close() 方法。它将在给定发布者的每个订阅者上调用下面的 onComplete() 回调。

运行该程序将产生以下输出：

```shell
接收到数据 ： 1
接收到数据 ： x
接收到数据 ： 2
接收到数据 ： x
接收到数据 ： 3
接收到数据 ： x
MySubscriber Complete!
```

## 4.消息的转换

假设我们想要在 Publisher 和 Subscriber 之间构建类似的逻辑，但也应用一些消息转换。

我们将创建实现 Processor 并扩展 SubmissionPublisher 的 TransformProcessor 类 - 这将同时是 Publisher 和 Subscriber。

我们将传入一个将输入转换为输出的 `Function`（函数）：

```java
public class TransformProcessor<T, R> extends SubmissionPublisher<R> implements Flow.Processor<T, R> {

    private final Function<T, R> function;
    private Flow.Subscription subscription;

    public TransformProcessor(Function<T, R> function) {
        super();
        this.function = function;
    }

    @Override
    public void onSubscribe(Flow.Subscription subscription) {
        this.subscription = subscription;
        subscription.request(1);
    }

    @Override
    public void onNext(T item) {
        submit(function.apply(item));
        subscription.request(1);
    }

    @Override
    public void onError(Throwable t) {
        t.printStackTrace();
    }

    @Override
    public void onComplete() {
        close();
    }
}
```

我们快速编写一个将 Publisher 发送的 String 元素转换为 Integer 元素的单元测试：

```java
@Test
public void whenSubscribeAndTransformElements_thenShouldConsumeAll()
        throws InterruptedException {

    SubmissionPublisher<String> publisher = new SubmissionPublisher<>();
    TransformProcessor<String, Integer> transformProcessor
            = new TransformProcessor<>(Integer::parseInt);
    MySubscriber<Integer> subscriber = new MySubscriber<>();
    List<String> items = List.of("1", "2", "3");
    List<Integer> expectedResult = List.of(1, 2, 3);

    publisher.subscribe(transformProcessor);
    transformProcessor.subscribe(subscriber);
    items.forEach(publisher::submit);
    publisher.close();

    await().atMost(1000, TimeUnit.MILLISECONDS)
            .until(() ->{
                        assertIterableEquals(expectedResult, subscriber.consumedElements);
                        return subscriber.consumedElements.size() == expectedResult.size();
                    }
            );

}
```
请注意，调用基础 Publisher 上的 close() 方法将导致调用 TransformProcessor 上的 onComplete() 方法。

请记住，处理链中的所有发布者都需要以这种方式关闭。

## 5.使用 Subscription 控制消息取值

假设我们只想使用 Subscription 中的第一个元素，应用一些逻辑并完成处理。我们可以使用 request() 方法来实现这一点。

让我们修改 MySubscriber 以仅消耗 N 条消息。我们将该数字作为 howMuchMessagesConsume 的构造函数参数进行传递：

```java
public class MySubscriber<T> implements Flow.Subscriber<T> {
    private Flow.Subscription subscription;
    private final AtomicInteger howMuchMessagesConsume;
    public List<T> consumedElements = new ArrayList<>();

    public MySubscriber(Integer howMuchMessagesConsume) {
        this.howMuchMessagesConsume = new AtomicInteger(howMuchMessagesConsume);
    }

    @Override
    public void onSubscribe(Flow.Subscription subscription) {
        this.subscription = subscription;
        subscription.request(1);
    }

    @Override
    public void onNext(T item) {
        howMuchMessagesConsume.decrementAndGet();
        System.out.println("接收到数据 ： "+item);
        if (howMuchMessagesConsume.get() == 0) {
            consumedElements.add(item);
            subscription.request(1);
        }
    }

    @Override
    public void onError(Throwable throwable) {
        throwable.printStackTrace();
    }

    @Override
    public void onComplete() {
        System.out.println("MySubscriber Complete!");
    }
}
```

编写单元测试，实现只使用 Subscription 中的第一个元素：

```java
@Test
public void whenRequestForOnlyOneElement_thenShouldConsumeOne()
        throws InterruptedException {

    SubmissionPublisher<String> publisher = new SubmissionPublisher<>();
    MySubscriber<String> subscriber = new MySubscriber<>(1);
    publisher.subscribe(subscriber);
    List<String> items = List.of("1", "x", "2", "x", "3", "x");
    List<String> expected = List.of("1");

    assertEquals(1, publisher.getNumberOfSubscribers());
    items.forEach(publisher::submit);
    publisher.close();

    await().atMost(1000, TimeUnit.MILLISECONDS)
            .until(() -> {
                        assertIterableEquals(expected, subscriber.consumedElements);
                        return expected.size() == subscriber.consumedElements.size();
                    }
            );
}
```

尽管 Publisher 正在发布六个元素，但我们的 MySubscriber 将仅消耗一个元素，因为它发出只处理该单个元素的信号。

通过在 Subscriber 上使用 request() 方法，我们可以实现更复杂的背压（back-pressure）机制来控制消息消费的速度。
