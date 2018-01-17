---
layout: post
title:  know stream api
date:   2018-01-14 11:57:44 +0800
categories: java8
---
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

