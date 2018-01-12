---
layout: post
title:  Hello World To Java8
date:   2018-01-12 11:57:44 +0800
categories: java8
---

## 为什么用java8
在了解和学习了java8之后，我发现java8带来的优势和所提升的效率，比学习它的成本要多得多，
所以在此向大家推荐使用java8的新特性：
1. 通过方法参数传递代码块的能力以及使用lambda匿名函数替代实现匿名类。
2. 新的流（Stream）API，提供了一种高层次集合操作方法，并带来了随手可得的并行能力。
3. 为接口提供默认方法的能力，为扩展接口带来了福音。

## 传递代码的能力
java8之前的方法参数只能传递对象，而java8允许我们将方法的引用作为参数传递给方法。
如`findApples(apples, Apple::isRedColor)`，我们传递了`Apple`类的方法，让
`findApples`方法根据`isRedColor`方法筛选需要的苹果。下面介绍一个完整的演变过程。
如果从一堆苹果中找出大的苹果，你之前可能会写如下代码。
```
public List<Apple> findBigApples(List<Apple> apples) {
    List<Apple> results = new ArrayList<>();
    for (Apple apple : apples) {
        if (apple.getWeight() > 200) {
            results.add(apple);
        }
    }
    return results;
}
```
后来可能又有需求是需要找出红苹果，于是可能会新增一个找红苹果的方法。
```
public List<Apple> findRedApples(List<Apple> apples) {
    List<Apple> results = new ArrayList<>();
    for (Apple apple : apples) {
        if (apple.getColor() == Color.RED) {
            results.add(apple);
        }
    }
    return results;
}
```
而`findBigApples`和`findRedApples`的唯一区别仅仅是判断苹果的条件，所以你可能会通过一个
接口，将筛选方法作为策略传入。
```
// 判断策略
public interface Predicate<T> {
    boolean test(T t);
}

// 通过指定的策略查找苹果
public List<Apple> findApples(List<Apple> apples, Predicate<Apple> p) {
    List<Apple> results = new ArrayList<>();
    for (Apple apple : apples) {
        if (p.test(apple)) {  // 如果苹果满足p所代表的筛选方法
            results.add(apple);
        }
    }
    return results;
}
```
而如果使用了传递代码的能力，你可以将筛选大苹果的使用方法优化为如下：
```
findApples(apples, Apple::isBigEnough);

// 其中Apple::isBigEnough为Apple对象的方法引用，isBigEnough方法为判断是否是大苹果的方法
public boolean isBigEnough() {
    return this.getWeight() > 200;
}
```
而找出红苹果的方法可以优化为
```
findApples(apples, Apple::isRedColor);
```
而对于既要大苹果又要红苹果的情形，你可能会写一个仅用一次的方法，针对这种情况，可以通过
lambda表达式进行优化。
### 使用lambda匿名函数
方法`public int add(int x, int y) {return x + y;}`可等效于lambda匿名函数
`(int x, int y) -> {return x + y;}`，当代码块只有一行时，可简写为`(x, y) -> x + y`。
所以找出又大又红的苹果可以通过传入lambda表达式实现
```
findApples(apples, (Apple apple) -> {
    return apple.isBigEnough() && apple.isRedColor()
});
```
如果lambda仅有一行表达式，可以省略return语句和两端花括号，同时Apple参数类型也可以省略，
jvm会根据上下文分析正确的参数类型。所以简化版的表达式如下：
```
findApples(apples, (apple) -> apple.isBigEnough() && apple.isRedColor());
```
有高手会表示之前的java也能通过策略模式向方法中传递代码，比如通过匿名类对接口的实现来传递
不同的代码块。但是相比冗长地定义接口实现，通过直接传递方法引用和lambda表达式能更加
简化代码并且提供清晰的可读性。对比一下我们以前通过匿名类实现的找出大苹果的方式，体会一下
可读性。
```
findApples(apples, new Predicate<Apple> {
    @Override
    public boolean test(Apple apple) {
        return apple.isBigEnough();
    }
});
```
引入lambda，并不只是一个语法糖，而是通过java7中新的[invokedynamic][url-invokedynamic]字节码，在运行时
确定执行的方法，这样可以将Groovy、Scala这类脚本语言进行粘合。

## 流API
流提供了一种抽象的集合操作，通过类比SQL提供的数据操作指令，流提供了在java环境中的
一套编程模式，继续找苹果的例子，通过流，我们可以免去编写显示的循环操作，下面通流API，
使用`Apple`的筛选大苹果的方法，从所有苹果中筛选出大苹果并返回。
```
import static java.util.stream.Collections.toList;
List<Apple> bigApples = apples.stream()
                              .filter((apple) -> apple.isBigEnough())
                              .collect(toList());
```
上面的示例也可通过传递方法引用表示
```
List<Apple> bigApples = apples.stream()
                              .filter(Apple::isBigEnough)
                              .collect(toList());
```
注意其中并没有使用for循环，循环由流隐式内部实现了，由此带来的好处就是可以非常简单地
获得多处理机上的并行能力，在此之前java并不能主动利用多核，通过多线程提高效率。而使用
并行流，我们能轻松获得类似流水线的并行处理能力。而其他代码完全无需变更。
```
List<Apple> bigApples = apples.parallelStream()
                              .filter(Apple::isBigEnough)
                              .collect(toList());
```

## 接口默认方法
接口通过`default`关键字可以支持接口中实现方法体，如：
```
public interface Sample {
    String getName();

    default void doSomething() {
        System.out.println("print default method.");
    }
}
```
通过接口默认方法，当我们需要对接口进行变更时，可以避免让接口的所有实现类以及子类进行修改
和重新编译。接口实现类和其子类可以直接使用接口默认方法。

#### 了解更多
* [《java8实战》](https://book.douban.com/subject/26772632/)

[url-invokedynamic]: http://www.infoq.com/cn/articles/Invokedynamic-Javas-secret-weapon