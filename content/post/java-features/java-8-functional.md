---
title: Java 8 函数式接口初识
description: 介绍 Java 8 java.util.function 包下的相关类的用途和基本用法
slug: java-8-functional
date: 2020-12-18 19:00:00+0000
#image: cover-java.jpg
categories:
    - Java
tags:
    - Java
    - Functional
weight: 1  
---

## 1. 概述

java.util.function 包是 Java 8 引入的一个包，其中包含了一组函数式接口，用于支持函数式编程和Lambda表达式。

这些函数式接口提供了一种方便的方式来定义和使用函数，可以在很多场景下简化代码的编写和处理。

## 2. @FunctionalInterface 是什么鬼？

@FunctionalInterface是一个Java注解，用于声明一个接口是函数式接口。函数式接口是指**只包含一个抽象方法的接口**，用于表示一个函数类型。

@FunctionalInterface注解的作用是强制编译器检查被注解的接口是否满足函数式接口的条件，即只包含一个抽象方法。如果接口不满足这个条件，编译器会报错。

函数式接口和@FunctionalInterface的使用场景是在函数式编程中，特别是在使用Lambda表达式时。Lambda表达式允许我们以更简洁的方式定义匿名函数，并将其作为参数传递给其他方法或函数式接口。函数式接口提供了一种标准化的方式来定义和使用这些可传递的函数。

下面是一个示例，演示了@FunctionalInterface的用途和示例代码：

```java
@FunctionalInterface
interface Calculator {
    int calculate(int a, int b);
}

public class FunctionalInterfaceExample {
    public static void main(String[] args) {
        Calculator addition = (a, b) -> a + b;
        int result = addition.calculate(5, 3);
        System.out.println(result); // 输出8

        Calculator subtraction = (a, b) -> a - b;
        result = subtraction.calculate(10, 4);
        System.out.println(result); // 输出6
    }
}
```

## 3. java.util.function 包接口清单介绍

java.util.function 包下的接口都有 @FunctionalInterface 注解。

以下表格是 java.util.function 包下所有接口的用途。

| 类                          | 描述                                                           |
| :-------------------------- | :------------------------------------------------------------- |
| Function <T,R>              | 表示接受一个参数并产生结果的函数。                             |
| Predicate &lt;T&gt;         | 表示一个参数的谓词（boolean函数）。                            |
| Consumer &lt;T&gt;          | 表示接受单个输入参数且不返回任何结果的操作。                   |
| Supplier &lt;T&gt;          | 代表结果的提供者。                                             |
| UnaryOperator &lt;T&gt;     | 表示对单个操作数的操作，该操作产生与其操作数类型相同的结果。   |
| BiConsumer <T,U>            | 表示接受两个输入参数且不返回任何结果的操作。                   |
| BiFunction <T,U,R>          | 表示接受两个参数并产生结果的函数。                             |
| BinaryOperator &lt;T&gt;    | 表示对两个相同类型的操作数的操作，产生与操作数相同类型的结果。 |
| BiPredicate <T,U>           | 表示两个参数的断言（boolean函数）。                            |
| BooleanSupplier             | 代表 boolean 值结果的供应商。                                  |
| DoubleBinaryOperator        | 表示对两个 double 值操作数并产生 double 值结果的操作。         |
| DoubleConsumer              | 表示接受单个 double 值参数且不返回任何结果的操作。             |
| DoubleFunction <R>          | 表示接受双值参数并产生结果的函数。                             |
| DoublePredicate             | 表示一个 double 值参数的断言（boolean函数）。                  |
| DoubleSupplier              | 代表 double 值结果的供应商。                                   |
| DoubleToIntFunction         | 表示接受双值参数并生成 int 值结果的函数。                      |
| DoubleToLongFunction        | 表示接受双值参数并产生 long 值结果的函数。                     |
| DoubleUnaryOperator         | 表示对产生 double 值结果的单个 double 值操作数的操作。         |
| IntBinaryOperator           | 表示对两个 int 值操作数并产生 int 值结果的操作。               |
| IntConsumer                 | 表示接受单个 int 值参数且不返回任何结果的操作。                |
| IntFunction <R>             | 表示接受 int 值参数并产生结果的函数。                          |
| IntPredicate                | 表示一个 int 值参数的谓词（boolean函数）。                     |
| IntSupplier                 | 代表 int 值结果的供应商。                                      |
| IntToDoubleFunction         | 表示接受 int 值参数并生成双值结果的函数。                      |
| IntToLongFunction           | 表示接受 int 值参数并产生 long 值结果的函数。                  |
| IntUnaryOperator            | 表示对产生 int 值结果的单个 int 值操作数的操作。               |
| LongBinaryOperator          | 表示对两个 long 值操作数并产生 long 值结果的操作。             |
| LongConsumer                | 表示接受单个 long 值参数且不返回任何结果的操作。               |
| LongFunction <R>            | 表示接受 long 值参数并产生结果的函数。                         |
| LongPredicate               | 表示一个 long 值参数的断言（boolean函数）。                    |
| LongSupplier                | 代表 long 值结果的供应商。                                     |
| LongToDoubleFunction        | 表示接受 long 值参数并产生双值结果的函数。                     |
| LongToIntFunction           | 表示接受 long 值参数并生成 int 值结果的函数。                  |
| LongUnaryOperator           | 表示对产生 long 值结果的单个 long 值操作数的操作。             |
| ObjDoubleConsumer &lt;T&gt; | 表示接受对象值和 double 值参数的操作，并且不返回任何结果。     |
| ObjIntConsumer &lt;T&gt;    | 表示接受对象值和 int 值参数并且不返回任何结果的操作。          |
| ObjLongConsumer &lt;T&gt;   | 表示接受对象值和 long 值参数并且不返回任何结果的操作。         |
| ToDoubleBiFunction <T,U>    | 表示接受两个参数并产生双值结果的函数。                         |
| ToDoubleFunction &lt;T&gt;  | 表示产生双值结果的函数。                                       |
| ToIntBiFunction <T,U>       | 表示接受两个参数并生成 int 值结果的函数。                      |
| ToIntFunction &lt;T&gt;     | 表示生成 int 值结果的函数。                                    |
| ToLongBiFunction <T,U>      | 表示接受两个参数并产生 long结果的函数。                        |
| ToLongFunction &lt;T&gt;    | 表示产生 long 值结果的函数。                                   |



## 4. 常用的函数式接口及其用途

1. Function<T, R>
   - 用途：接受一个输入参数并返回一个结果，用于实现将一个类型的值转换为另一个类型的值的操作。
   - 示例代码：
        ```java
        Function<Integer, String> intToString = (num) -> Integer.toString(num);
        String str = intToString.apply(42); // 结果为"42"
        ```
    - 除了apply方法，Function接口还提供了一些默认方法，用于支持函数的组合、转换和处理。下面是Function接口的所有方法的详细介绍和代码示例：
        1. default <V> Function&lt;T, V> andThen(Function<? super R, ? extends V> after)
            - 用途：返回一个组合函数，首先将当前函数的结果传递给参数after的apply方法，然后将该结果作为组合函数的输出结果。
            - 示例代码：
            ```java
            Function<Integer, Integer> square = (num) -> num * num;
            Function<Integer, String> squareAndToString = square.andThen(Object::toString);
            String result = squareAndToString.apply(5); // 结果为"25"
            ```
        2. default <V> Function<V, R> compose(Function<? super V, ? extends T> before)
            - 用途：返回一个组合函数，首先将参数before的结果传递给当前函数的apply方法，然后将该结果作为组合函数的输入参数。
            - 示例代码：
            ```java
            Function<Integer, Integer> multiplyBy2 = (num) -> num * 2;
            Function<Integer, Integer> subtractBy3 = (num) -> num - 3;
            Function<Integer, Integer> multiplyBy2AndSubtractBy3 = multiplyBy2.compose(subtractBy3);
            int result = multiplyBy2AndSubtractBy3.apply(5); // 结果为4
            ```
        3. default <T> Function<T, T> identity()
            - 用途：返回一个函数，该函数对输入参数执行恒等转换，即返回输入参数本身。
            - 示例代码：
            ```java
            Function<String, String> identityFunction = Function.identity();
            String result = identityFunction.apply("Hello"); // 结果为"Hello"
            ```

2. Predicate&lt;T&gt;
   - 用途：接受一个输入参数并返回一个布尔值，用于实现条件判断。
   - 示例代码：
        ```java
        Predicate<Integer> isEven = (num) -> num % 2 == 0;
        boolean result = isEven.test(5); // 结果为false
        ```
3. Consumer&lt;T&gt;
   - 用途：接受一个输入参数，但不返回任何结果，用于实现对输入参数的操作。
   - 示例代码：
        ```java
        Consumer<String> printMessage = (message) -> System.out.println(message);
        printMessage.accept("Hello, World!"); // 输出"Hello, World!"
        ```
4. Supplier&lt;T&gt;
   - 用途：不接受任何输入参数，但返回一个结果，用于实现对结果的生成。
   - 示例代码：
        ```java
        Supplier<Double> getRandomNumber = () -> Math.random();
        double number = getRandomNumber.get(); // 返回一个随机数
        ```
5. UnaryOperator&lt;T&gt;
   - 用途：接受一个输入参数，并返回与输入参数类型相同的结果，用于实现对输入参数的操作并返回结果。
   - 示例代码：
        ```java
        UnaryOperator<Integer> square = (num) -> num * num;
        int result = square.apply(5); // 结果为25
        ```
这只是java.util.function包中的一小部分函数式接口，还有其他一些接口如BiFunction、BiPredicate、BiConsumer等，用于处理两个输入参数的场景。

这些函数式接口在函数式编程和Lambda表达式中起到了重要的作用，简化了代码的编写，提高了代码的可读性和可维护性。