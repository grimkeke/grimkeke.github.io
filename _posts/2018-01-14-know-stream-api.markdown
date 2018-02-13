---
layout: post
title:  Stream读书笔记
date:   2018-01-14 11:57:44 +0800
categories: java8
---
## 什么是流
流（Stream）是一组Java API，通过声明式的方式处理数据集合（类似于SQL，通过编写查询
的语句，而不是亲自实现一个查询功能）。对比之前面向集合，亲自来实现集合操作，通过流，
开发者可以关注与实现的目的，而非实现操作本身。对比如下两段功能一样的代码，查询Dish列表
中卡路里小于400的菜品名称，并且根据菜品卡路里排序。java7的方法即为根据语义进行实现，
即首先遍历所有的菜品，找出小于400卡路里的菜品添加到临时列表，进行排序，然后再创建一个
列表用于提取菜品的名称。
```
public List<String> getLowCaloricDishesNamesInJava7(List<Dish> dishes){
    List<Dish> lowCaloricDishes = new ArrayList<>();
    for(Dish d: dishes){
        if(d.getCalories() < 400){
            lowCaloricDishes.add(d);
        }
    }

    Collections.sort(lowCaloricDishes, new Comparator<Dish>() {
        public int compare(Dish d1, Dish d2){
            return Integer.compare(d1.getCalories(), d2.getCalories());
        }
    });

    List<String> lowCaloricDishesName = new ArrayList<>();
    for(Dish d: lowCaloricDishes){
        lowCaloricDishesName.add(d.getName());
    }
    return lowCaloricDishesName;
}
```
而java8的流操作就很直观，通过`.stream()`方法，将菜品列表
转换为菜品流，接着通过`filter`过滤，通过`sorted`进行排序，通过`map`将菜品列表映射
为名称列表，最后通过`collect`收集保存为List。
```
public List<String> getLowCaloricDishesNamesInJava8(List<Dish> dishes){
    return dishes.stream()
            .filter(d -> d.getCalories() < 400)
            .sorted(Comparator.comparing(Dish::getCalories))
            .map(Dish::getName)
            .collect(Collectors.toList());
}
```
而将流操作转变为并行操作也非常简单，只需将`.stream()`方法修改为`.parallelStream()`
方法即可。流默认会根据当前运行机器的内核数来进行并行执行。
```
public List<String> getLowCaloricDishesNamesInJava8(List<Dish> dishes){
    return dishes.parallelStream()
            .filter(d -> d.getCalories() < 400)
            .sorted(Comparator.comparing(Dish::getCalories))
            .map(Dish::getName)
            .collect(Collectors.toList());
}
```
除了生命式的表述数据操作方式之外，流的编写模式是链式的，即通过一系列方法组合成操作
流水线。而在流的内部，每个处理节点的操作也像流水线一样并行处理。
```
           lamdba             lambda             lambda
             +                   +                  +
             |                   |                  |
menu    +----v-----+       +-----v----+       +-----v----+      +---------+
+------->  filter  +------->  sorted  +------->   map    +------> collect |
        +----------+       +----------+       +----------+      +---------+
```
即流有如下三个特点：声明式、可复合（链式）、可并行。
## 流与集合的理解
集合可以理解为空间上的一组序列，首先集合中的元素必须都存在才能产生集合，如我们无法创建
一个偶数的集合，因为偶数是无穷的，因此集合是无穷的，空间上无法确定集合大小。而流则可以
理解为时间上的一组序列，流中的元素不必全部产生后才可使用，因此，流有按需生产、延迟生产
的特点，我们可以很轻易地创建一个包含所有偶数的流，只不过这个流中的元素永远也输出不完。
### 外部迭代和内部迭代
集合中的元素是确定的，并且我们需要自行创建迭代来遍历集合中的所有元素；而流中的元素则
是由内部迭代完成的，迭代交给流来进行有很多好处，首先带来了声明式的简洁，且因为迭代在
内部进行，流的实现者可以为我们提供了迭代优化和并行优化（具体通过java7提供的fork-join
框架实现）
### 流的操作
流分为中间操作和终端操作两种操作类型。中间操作并不真正操作流中的元素，而是建设起操作流
的流水线，这种流的延迟特性带来了`循环合并`以及`短路`等优化技巧的用武之地。而终端操作则会
从流水线生成结果。这个结果是任何不是流的值，比如void，List，int等。并且流只能够遍历
一次，所以无法一次链式处理的流操作，需要再次打开流，进行第二次遍历处理。在上述例子中，
filter、sorted、map为中间操作，collect为终端操作。
## 使用流
流提供了许多操作，能够给你快速完成复杂的数据处理，如筛选、切片、映射、查找、匹配和规约。
### 筛选
通过谓词筛选，`filter`接口`Stream<T> filter(Predicate<? super T> predicate)`需要
提供一个谓词，而谓词是一个返回`true`或者`false`的接口，用于根据指定元素t进行条件判断
```
public interface Predicate<T> {
    /**
     * Evaluates this predicate on the given argument.
     *
     * @param t the input argument
     * @return {@code true} if the input argument matches the predicate,
     * otherwise {@code false}
     */
    boolean test(T t);
}
```
在本例中，我们通过传入`Dish`对象的方法引用`Dish::isVegetarian`来告诉菜品流，我们要
过滤出所有的素食菜品
```
List<Dish> vegetarianMenu = menu.stream()
                                .filter(Dish::isVegetarian)
                                .collect(toList());
```
`Dish`对象中判断是否是素食的方法
```
public boolean isVegetarian() {
        return vegetarian;
}
```
而根据lambda表达式，过滤方法也可以按如下形式表达
```
List<Dish> vegetarianMenu = menu.stream()
                                .filter(dish -> dish.isVegetarian())
                                .collect(toList());
```
### 筛选各异的元素
通过`distinct()`方法，我们可以筛选出不重复的元素
```
List<Integer> numbers = Arrays.asList(1, 2, 1, 3, 3, 2, 4);
numbers.stream()
       .filter(i -> i % 2 == 0)
       .distinct()
       .forEach(System.out::println);
```
### 截短流
通过`limit(3)`将流的输出限制为3个，在流的内部实现中，会采用一种叫做`短路`的优化，即
无需完整过滤完整个流，只要`filter()`方法过滤出3个菜品，即马上停止流的内部遍历，通过
`collect()`方法返回列表。
```
List<Dish> dishesLimit3 = menu.stream()
                .filter(d -> d.getCalories() > 300)
                .limit(3)
                .collect(toList());
```
### 跳过元素
通过`skip(2)`方法，返回一个丢弃了前两个元素的流。
```
List<Dish> dishesSkip2 = menu.stream()
                .filter(d -> d.getCalories() > 300)
                .skip(2)
                .collect(toList());
```
### 映射
映射类似于SQL里筛选出一张表中的某一列数据，通过`map`和`flatMap`方法，我们可以将流
中的元素结合映射为另一种类型的集合，比如从菜品流`List<Dish>`选出所有菜品的名称`List<String>`。
`map`接口`<R> Stream<R> map(Function<? super T, ? extends R> mapper)`需要
传入一个`Function`类型的函数式接口，这个接口的作用是把传入的参数从一种类型转换为另一种类型，
就菜品转换而言，流内部的循环会通过这个接口把单个`Dish`元素转变为`String`元素，然后
通过内部迭代器收集对象，最终通过`collect`输出为列表或者做其他处理。
```
public interface Function<T, R> {
    /**
     * Applies this function to the given argument.
     *
     * @param t the function argument
     * @return the function result
     */
    R apply(T t);
}
```
在本例中，我们通过`map`方法，将`Dish`对象映射为`String`对象，即获取菜品的名称，最后
通过`Collectors.toList()`方法采集输出为字符串列表。
```
List<String> dishNames = menu.stream()
                             .map(Dish::getName)
                             .collect(toList());
```
`Dish`对象中获取菜品名称的方法如下
```
public String getName() {
        return name;
}
```
下面的例子展示了计算数组中每个单词的长度，通过方法引用`String::length`获得每个单词
的长度，返回类型`int`通过java的自动装箱操作转换为`Integer`类型（自动装箱操作有额外
操作成本，需要进行关注，后续会介绍java8提供的几个原生Stream类型用于提高性能），
```
List<String> words = Arrays.asList("Hello", "World");
List<Integer> wordLengths = words.stream()
                                 .map(String::length)
                                 .collect(toList());
```
#### 流的扁平化
`flatMap`方法`<R> Stream<R> flatMap(Function<? super T, ? extends Stream<? extends R>> mapper)`
需要传入一个输出转换结果为流的函数方法。即`flatMap`首先将输入流中的每个元素，
通过`Function`接口转换为对应的流，然后再将所有得到的流连接起来成为一个流。
为了输出单词列表中所有出现的不同字母，我们首先通过`Arrays.stream()`方法将输入字符串
转换为`Stream<String>`，然后每个单词转换的流进行数据整合，归集为一个`Stream<String>`
流，最后通过`distinct()`方法筛选流中所有不重复的字母。
```
words.stream()
     .flatMap((String line) -> Arrays.stream(line.split("")))
     .distinct()
     .forEach(System.out::println);
```
### 匹配
- 检查谓词是否至少匹配一个元素：`boolean anyMatch(Predicate<? super T> predicate);`
```
menu.stream()
    .anyMatch(Dish::isVegetarian);  // 菜单中是否有一个素菜
```
- 检查谓词匹配所有元素：`boolean allMatch(Predicate<? super T> predicate);`
```
menu.stream()
    .allMatch(d -> d.getCalories() < 1000);  // 所有菜品卡路里是否小于1000
```
- 检查谓词对所有元素均不匹配：`boolean noneMatch(Predicate<? super T> predicate);`
```
menu.stream()
    .noneMatch(d -> d.getCalories() >= 1000); // 所有菜品卡路里是否不大于1000
```

这三个查找方法均使用到了短路优化，即在满足条件时，无需完整遍历流中所有元素，即可返回结果。
### 查找元素
- 查找任意元素：`Optional<T> findAny();`
```
Optional<Dish> dish = menu.stream()
            .filter(Dish::isVegetarian).findAny();  // 找出菜单任意一个素菜
dish.ifPresent(d -> System.out.println(d.getName())); // 如果有素菜则打印菜名
```
- 查找第一个元素：`Optional<T> findFirst();`
```
Optional<Dish> dish = menu.stream()
            .filter(Dish::isVegetarian).findFirst();  // 找出菜单第一个素菜
```

查找元素返回的是一个`Optional<T>`类型，用于表示对象是否存在，通过
`ifPresent(Consumer<? super T> consumer)`方法可以传入一个消费者，当对象存在时，执行消费者的消费动作。

```
public interface Consumer<T> {
    /**
     * Performs this operation on the given argument.
     *
     * @param t the input argument
     */
    void accept(T t);
}
```

### 归约
归约用来将一系列元素折叠成一个元素，比如对1至100的数字求和，通过归约操作，我们可以不断
对累积值和流中的数字使用加法，从而完成所有数字的累加，最终累积值即为所有元素之和。归约
操作通过`reduce()`方法实现。我们先来看一下它的用法。
```
List<Integer> numbers = Arrays.asList(3,4,5,1,2);
int sum = numbers.stream().reduce(0, (a, b) -> a + b);

// 通过Integer提供的求和静态方法sum完成累加
int sum2 = numbers.stream().reduce(0, Integer::sum);
```
该方法有如下两种重载：
- 提供初始值，以及一个二元操作的方法引用，上面例子使用的是该用法，通过提供初始值以及
一个函数引用，我们向流提供一种累加的方法，即我们需要一个整数，初始值为0，
通过`(a, b) -> a + b`方法 （即`Integer::sum`）不断与流中的元素进行归约计算，
最终得到累加和的输出。
```
T reduce(T identity, BinaryOperator<T> accumulator);
```
- 仅提供二元操作的方法引用，当流为空时，可能会返回空结果，因此方法签名的返回值为
一个可选的`Optional<T>`。
```
Optional<T> reduce(BinaryOperator<T> accumulator);
```

同时，我们也可以通过归约来求出流中的最大值最小值，因为求出流中的最大值最小值也是分别
两两比较流内的数字。下面例子可以计算如何求流的最大值，当流为空时，最大值为0。
```
int max = numbers.stream().reduce(0, (a, b) -> Integer.max(a, b));
```
这不一定是你期望的结果，因此可以采用`reduce()`方法的单参数重载方法。并根据流非空的情况
进行相应处理。
```
Optional<Integer> min = numbers.stream().reduce(Integer::min);
min.ifPresent(System.out::println);
```

关于流的操作，诸如map或者filter等操作会从输入中获取每一个元素进行处理，并且得到0个
或者1个结果，这些操作一般是无状态的，因此可以很容易地并行化。而对于reduce、sum、max
这类操作则需要内部维护一个状态来进行累计结果，因此被称为有状态，所以内部实现的并行策略
会完全不一样，会使用分支/合并框架，即先讲数据分块，分块求和后，最后再将数据合并起来。
对于sort或者distinct这类有状态的操作是通过一个输入流产出对应的输出流，每一个输入数据
进入处理时还需要知道先前处理的历史，因此当流比较大或者无界时，有一些操作就会有问题，比如
将一个质数流进行倒序（数学告诉我们，无穷多的质数无法倒序排序）
### 数值流
数值流是原始类型流的特化流，专门用于处理数值类型，如`InputStream`、`DoubleStream`、
`LongStream`，使用数值流有两个原因，一是通用的流并没有提供数值操作才有的类似sum等方法，
两个对象的sum并没有任何意义，虽然你可以使用`Stream<Integer>`的方式处理数值，但是
java的自动装箱机制会带来额外的开销，所以通过专门处理int、double和long型的数值流，
可以带来更高的效率。
```
IntStream intStream = menu.stream().mapToInt(Dish::getCalories);
int calories = intStream.sum();

// 数值流转换为对象流
Stream<Integer> stream = intStream.boxed();

// 数值型默认值，原生类型需使用对应的特异化Optional版本，之所以是需要OptionalInt
// 是因为流可能为空，所以不一定存在最大值，所以如果没有最大值时，我们可以显式提供一个。
OptionalInt maxCalories = intStream.max();
int max = maxCalories.orElse(1);
```
#### 数值流的范围
通过`rangeClosed(m, n)`方法生成从m到n（不包括n）范围的数值流。
```
IntStream evenNumbers = IntStream.rangeClosed(1, 100)
                                 .filter(n -> n % 2 == 0);
System.out.println(evenNumbers.count());
```
### 构建流
构建一个流有多种办法，通过从值序列、数组、文件来创建，甚至是生成函数来创建无限流。
- 生成空流
```Stream<String> emptyStream = Stream.empty();```
- 显示创建流
```Stream<String> stream = Stream.of("hello", " world", " my", " friends");```
- 由数组创建流
```
int[] numbers = {2, 3, 5, 7, 11, 13};
int sum = Arrays.stream(numbers).sum();
```
- 由文件生成流
```
// 读取data.txt文件并计算使用了多少种不同的单词。
Paths paths = Paths.get("path/data.txt");
long uniqueWords = Files.lines(paths, Charset.defaultCharset())
                        .flatMap(line -> Arrays.stream(line.split(" ")))
                        .distinct()
                        .count();
```
- 通过函数生成无限流
通过`Stream.iterate`和`Stream.generate`方法可以生成无限流，对于无限流的使用，一般
会通过`limit(n)`来限制流的大小，避免出现无穷打印。
`public static<T> Stream<T> iterate(final T seed, final UnaryOperator<T> f);`
iterate通过传入一个初始值seed和一元操作函数引用来迭代产生新的值，首先返回初始值seed，
然后通过初始值seed输入函数引用f，生成第二个值s1，接着通过s1输入f，得到第三个值s2，
以此类推。
```
// 打印10个偶数
Stream.iterate(0, n -> n + 2)
      .limit(10)
      .forEach(System.out::println);
// 创建斐波那契数列
Stream.iterate(new int[]{0, 1}, t -> new int[]{t[1],t[0] + t[1]})
      .limit(10)
      .forEach(t -> System.out.println("(" + t[0] + ", " + t[1] + ")"));
```
与`iterate`方法类似，`generate`方法是通过接收一个`Supplier<T>`的函数引用来生成
新的值。`public static<T> Stream<T> generate(Supplier<T> s);`，通过方法引用，
我们可以内部维护一个状态，在每次生成数据之后可以更新该状态，从而实现有状态的供应源。
但是在并行流中使用有状态的流是不安全的，应答尽量避免。
```
// 产生10个随机数
Stream.generate(Math::random)
      .limit(10)
      .forEach(System.out::println);

// 避免装箱操作，通过DoubleStream实现
DoubleStream.generate(Math::random)
            .limit(5)
            .forEach(System.out::println);
```
### 收集
```
<R> R collect(Supplier<R> supplier,
                  BiConsumer<R, ? super T> accumulator,
                  BiConsumer<R, R> combiner);
```

### 附录A 中间操作和终端操作表
| 操作      | 类型            | 返回类型           | 使用的类型/函数式接口 | 函数描述符    |
| -------- | --------------- | ----------------- | ------------------ | ------------ |
| filter   | 中间             | `Stream<T>`       | `Predicate<T>`     | T -> boolean |
| distinct | 中间（有状态-无界）| `Stream<T>`       |                    |              |