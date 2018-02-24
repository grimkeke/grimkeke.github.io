---
layout: post
title:  并行数据处理与性能读书笔记
date:   2018-02-23 14:31:44 +0800
categories: java8
---
## 并行数据处理
java7之前的数据集合并行处理非常麻烦，通常我们首先需要明确地将数据划分为若干子部分，
第二需要为每一个子部分分配一个独立的线程，第三我们还需要在恰当的时候对这些线程进行同步
来避免不希望出现的竞争条件，等待所有线程完成，最后把这些部分结果合并起来。java7开始
引入了一个分支/合并的框架，让这些操作更稳定、更不易出错。
### 并行流
并行流可以通过对集合调用`parallelStream`方法来生成。通过并行流，我们可以将集合中的
数据分成多个块，使用不同的线程分别处理各个数据块的流，这样可以将工作分配到多核处理器
的所有内核，让同一个任务操作并行起来。下面以对数字1到n进行累加和的计算为例，说明并行流
的操作和性能优化。
```
// java7之前的迭代式对小于等于n的数字求和
public static long iterativeSum(long n) {
    long result = 0;
    for (long i = 0; i <= n; i++) {
        result += i;
    }
    return result;
}

// java8之后通过流的归约完成数字求和
public static long sequentialSum(long n) {
    return Stream.iterate(1L, i -> i + 1)  // 通过顺序迭代产生流
                 .limit(n)                 // 限制只产生n个数字
                 .reduce(Long::sum)        // 通过求和方法计算流中元素总和
                 .get();                   // 因为不会为空，从Optional得到值
}
```
### 将顺序流转换为并行流
通过对顺序流调用`parallel`方法，可以将其标记转换为并行流。
```
public static long parallelSum(long n) {
    return Stream.iterate(1L, i -> i + 1)
                 .limit(n)
                 .parallel()  // 将顺序流转换为并行流
                 .reduce(Long::sum)
                 .get();
}
```
在实现中，对顺序流调用`parallel`方法并不会改变流本身，而只是在流的内部设置了一个表示
是否期望并行执行的`boolean`标志位，如果设置为`true`，则表示我们期望流操作按照并行的
方式执行。而通过对并行流调用`sequential`方法，我们可以将并行流转换为顺序流。注意我们
已经了解流的实现机制，所以如下在同一个流中反复设置流的并行化和串行化的最终结果是以最后
一次调用设置方法为准。
```
// 我们期望先对filter使用串行，然后对map使用并行，但实际上整个流并行执行
// 因为整个流会按最后parallel方法指定的标志进行操作
stream.parallel()
      .filter(...).sequential()
      .map(...).parallel()
      .reduce();
```
### 并行流的线程池
并行流使用`ForkJoinPool`提供的线程执行子集合的操作，它默认的线程数量是运行环境的处理器
数量，这个值由`Runtime.getRuntime().availableProcessors()`获得。该方法实际返回的
是可用内核的数量，包括了超线程生成的虚拟内核。使用该默认值对绝大多数情况都是不错的。

### 测量流性能
对于上面通过归约完成数字并行求和的例子，我们期望顺序流可以划分成若干子部分，每个子流
分别在不同的内核执行计算，最后，同一个归约操作会将每个子流的结果合并起来，得到整个
原始流的最终结果。而由于并行操作，我们期望执行时间会比顺序执行要快，下面我们通过一个
性能测试来直观感受一下。
```
// 通过10次测试取最短耗时评估求和方法的性能。
public static <T, R> long measurePerf(Function<T, R> f, T input) {
    long fastest = Long.MAX_VALUE;
    for (int i = 0; i < 10; i++) {
        long start = System.nanoTime();
        R result = f.apply(input);
        long duration = (System.nanoTime() - start) / 1_000_000;
        System.out.println("Result: " + result);
        if (duration < fastest) fastest = duration;
    }
    return fastest;
}

// 串行求和1到1千万
measurePerf(ParallelStreams::sequentialSum, 10_000_000L);

// 并行求和1到1千万
measurePerf(ParallelStreams::parallelSum, 10_000_000L);
```
运行结果可以看到`sequentialSum`方法耗时161ms，而`parallelSum`耗时493ms。居然并行
执行的耗时更长，这个结果和我们期望的不一样，而实际上这里有两个问题。
1. `iterate`生成的是装箱操作的对象，而对象必须进行拆箱操作才能求和，装箱拆箱操作极大
增加了运算量。
2. `iterate`方法很难划分为多个独立子数据块来进行并行操作，因为iterate方法是有状态的
顺序迭代，每一次新迭代结果的计算都要依赖前一次的迭代结果。
因此，将`iterate`操作并行化会导致多个线程同步顺序执行，要求每次操作在不同的线程上执行，
实际上增加了处理开销。所以我们在调用`parallel`方法之前，需要清除知晓背后的数据操作，
是否适合并行化。
#### 更为有效的方法
通过`LongStream.rangeClosed`方法，我们首先避免了装箱拆箱操作，其次`rangeClosed`
方法产生的数字范围是很容易拆分为独立的小块，如生成范围[1,20]的数字序列，可以轻易划分为
[1,5],[6,10],[11,15],[16,20]的独立子序列，从而可以分配到多个线程并行执行。

```
// 使用特化流并使用无状态的rangeClosed方法进行顺序求和
public static long rangedSum(long n) {
    return LongStream.rangeClosed(1, n)
                     .reduce(Long::sum)
                     .getAsLong();
}

// 使用特化流并使用无状态的rangeClosed方法进行并行求和
public static long parallelRangedSum(long n) {
    return LongStream.rangeClosed(1, n)
                     .parallel()
                     .reduce(Long::sum)
                     .getAsLong();
}
```
运行结果可以看到`rangedSum`方法耗时28ms，而`parallelRangedSum`耗时9ms。从而我们
得到了一个更为快捷的并行归约算法。
### 使用并行流的注意点
1. 对流做递归划分，把每个子流的归纳操作分配到不同的线程上去，然后再把这些操作的结果合并
成一个值。在多个内核之间移动数据的代价可能会比想象的大，因此需要保证任务在内核中并行
执行工作的时间要比在内核之间传输数据的时间长。
2. 对流做并行处理，一定要保证划分的任务是无状态的，以免出现竞争问题，而对竞争资源使用同步
机制就会大大减弱并行的执行效率，所以选择正确的数据结构以及正确的数据划分方式是正确并行
优化的前提。要考虑流背后的数据结构是否易于分解。
3. 如果有疑问，就测量耗时。因为将顺序流转换为并行流是轻而易举的事情（但并不一定能提高效率），
所以在考虑使用顺序流和并行流时，可以采用适量的基准来检查其性能。
4. 对数字留意装箱机制。自动装箱和拆箱操作会大大降低性能，所以应该使用原始流来避免装箱。
5. 有一些操作本身在并行流上执行的性能就比顺序执行差，比如`limit`和`findFirst`等依赖
元素顺序的操作。而`findAny`会比`findFirst`性能好，因为它不一定要按照顺序来执行，所以
我们可以通过对流调用`unordered`方法来把流标记为无序流，这样通过对无序流调用如`limit`
操作就会比对有序流调用该操作更高效。
6. 对于较小的数据量，选择并行流几乎从来都不是一个很好的决定，并行处理少数数据的好处还
抵不上并行化造成的额外开销。
7. 流自身的特点，以及流水线的中间操作修改流的方式，都可能会改变分解过程的性能。比如将
一个流分成相等大小的两部分，这样每一部分都能高效并行执行，但是筛选操作可能丢弃的元素个数
是未知的，因此划分出来的两个子流经过筛选后的大小是未知的。
8. 还要考虑终端操作中合并步骤的代价是大是小（比如`Collector`接口中的`combiner`方法）。
如果合并结果的代价很大，那么组合每个子流产生的部分结果所付出的代价可能就会超过并行流
得到的性能提升。

### 分支/合并框架
分支/合并框架的目的是用递归的方式将可以并行的任务拆分为更小的任务，然后每个子任务的结果
合并起来生成最终结果。它是`ExecutorService`接口的一个实现，它把任务分配给线程池
（默认为`ForkJoinPool`）中的线程进行工作。
#### 使用RecursiveTask
通过继承抽象类`RecursiveTask<V>`，我们可以创建能够进行分支/合并的任务，并且提交到
这个线程池中并行执行。泛型`V`为任务的返回值类型，如果任务无返回值，则可以继承抽象类
`RecursiveAction`。两者仅需要实现唯一的抽象方法`compute`。
```
/**
 * A recursive result-bearing {@link ForkJoinTask}.
 * For a classic example, here is a task computing Fibonacci numbers:
 * class Fibonacci extends RecursiveTask<Integer> {
 *   final int n;
 *   Fibonacci(int n) { this.n = n; }
 *   Integer compute() {
 *     if (n <= 1)
 *       return n;
 *     Fibonacci f1 = new Fibonacci(n - 1);
 *     f1.fork();
 *     Fibonacci f2 = new Fibonacci(n - 2);
 *     return f2.compute() + f1.join();
 *   }
 * }}
 */
public abstract class RecursiveTask<V> extends ForkJoinTask<V> {
    /**
     * The result of the computation.
     */
    V result;

    /**
     * The main computation performed by this task.
     * @return the result of the computation
     */
    protected abstract V compute();
    ...
}
```
这个方法定义了将任务拆分为子任务的逻辑，以及当任务无法拆分或不方便拆分时，生成单个
子任务结果的逻辑。其结果就是我们需要在`compute`方法中按如下伪代码模板来编写实现。
```
if (任务足够小或者不可分) {
    顺序计算该任务
} else {
    将任务分成两个子任务
    递归调用本方法，拆分每个子任务，等待所有子任务完成
    合并每个子任务的结果
}

// 对上例求斐波那契数列套用compute模板的实现如下
if (n <= 1) {   // 初始两个数字0和1是不可分任务，直接返回
    return n;
} else {        // 其他可分割任务的情况
    Fibonacci f1 = new Fibonacci(n - 1);  // 求比n小1的斐波那契数值
    f1.fork();                            // 把该任务分配到线程池处理
    Fibonacci f2 = new Fibonacci(n - 2);  // 求比n小2的斐波那契数值
    Integer r2 = f2.compute();            // 递归调用compute方法拆分任务拿结果
    Integer r1 = f1.join();               // 等待先前提交到线程池的子任务的结果
    return r2 + r1;                       // 合并子任务的结果
}
```
对于使用分支/合并框架并行计算数字1到n的和。我们使用的数据划分方式为当数组大小不超过
10000时，通过顺序累加计算得到结果，否则进行任务划分。以下为完整的对小于等于n的自然数
并行求和的代码实现。
```
public class ForkJoinSumCalculator extends RecursiveTask<Long> {
    public static final ForkJoinPool FORK_JOIN_POOL = new ForkJoinPool();
    public static final long THRESHOLD = 10_000;
    private final long[] numbers;
    private final int start;
    private final int end;

    public ForkJoinSumCalculator(long[] numbers) {
        this(numbers, 0, numbers.length);
    }

    private ForkJoinSumCalculator(long[] numbers, int start, int end) {
        this.numbers = numbers;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        int length = end - start;
        if (length <= THRESHOLD) {
            return computeSequentially();
        }
        ForkJoinSumCalculator leftTask = new ForkJoinSumCalculator(numbers
                                                , start, start + length/2);
        leftTask.fork();
        ForkJoinSumCalculator rightTask = new ForkJoinSumCalculator(
                                            numbers, start + length/2, end);
        Long rightResult = rightTask.compute();
        Long leftResult = leftTask.join();
        return leftResult + rightResult;
    }

    private long computeSequentially() {
        long sum = 0;
        for (int i = start; i < end; i++) {
            sum += numbers[i];
        }
        return sum;
    }

    public static long forkJoinSum(long n) {
        long[] numbers = LongStream.rangeClosed(1, n).toArray();
        ForkJoinTask<Long> task = new ForkJoinSumCalculator(numbers);
        return FORK_JOIN_POOL.invoke(task);
    }
}
```
### 使用分支/合并框架的注意点
1. 因为对一个任务调用`join`方法会阻塞调用方，直到被调用任务返回结果。因此我们需要在
两个子任务的计算都开始后再调用它，否则每个子任务都需要等待先前子任务执行完成后才开始执行，
无法达到并行执行的效果。
2. `ForkJoinPool`的`invoke`方法只是针对顺序代码来启动计算的，在递归任务的内部，应该
严格按照`compute`代码框架，直接调用`compute`和`fork`方法，来直接将任务添加到线程池中。
3. 在`compute`的实现中，我们划分了两个子任务，对其中一个子任务调用了`fork`，使其在
线程池中执行，但是对第二个子任务，我们直接调用了`compute`在当前进程中递归执行，这样
的目的是因为在当前线程中执行可以减少一次从线程池分配线程进行任务调度执行的开销，在当前
线程中继续执行另一个子任务的效率会更高。
4. 使用分支/合并框架的并行计算对调试会比较棘手，因为调用`compute`的线程并不是概念上
的调用方，而调用方是调用`fork`方法的那个。
5. 和并行流一样，使用分支/合并框架并不一定能比顺序计算快。一个任务只有能独立划分为若干
子任务，才能让性能在并行计算时有所提升，并且并行计算任务的时间需要长于任务划分的时间才
可能发挥并行优势。一个常用的方法是将输入/输出操作和计算操作分别放入不同的任务。此外比较
同一算法的顺序和并行执行的性能时还需要考虑"预热"等其他因素，比如测试算法性能之前我们都
会重复跑几遍程序，这样字节码才会被JIT编辑器优化为机器码，因为我们需要比较的是机器码的
执行效率，而不是字节码解释执行的运行效率。

### 工作窃取
通常，我们将任务划分为大量执行时间相同的小任务是一个好的选择。在理想的情况下，划分
出并行任务时，我们希望所有的CPU内核都能同样繁忙，而不是有的内核完成所有任务的执行，而
其他内核正忙于处理队列中的任务，因为每个任务的执行时间实际上会依赖磁盘甚至外部服务，造成
运行时间并不可知。因此将任务划分为大量小任务有利于所有内核持续同时执行任务。并且在分支/合并
框架的实现中，采用了一种叫做工作窃取（work stealing）的技术，即所有的内核均维护一个
双向任务队列，当某个内核执行完成它的任务队列后，它可以随机从其他还在运行任务的内核队列中
窃取队尾的一个任务，从而继续执行下去，直到所有的任务均执行完成，这样可以有助于更好地
平衡工作线程的负载。
### Spliterator拆分任务
在上述例子中，我们都是通过手工决定如何拆分任务，而对于并行流，我们并没有指定如何把流拆分
为子流，这说明有一种自动拆分机制完成了任务划分。这就是java8新加入的接口`Spliterator`。
它的名称为"可分迭代器"，和`Iterator`一样，也是用来遍历数据源中的元素，但是它是为了
并行执行而设计，而且我们大多数情况下也无需编写自己的`Spliterator`，但了解一下它的实现
方式会让我们对流的工作原理有更深入的了解。java8已经为所有的集合元素提供了一个默认的
可分迭代器的实现，如`ArrayList`内部实现了一个`ArrayListSpliterator`用于对顺序列表的
自动划分。下面看一下`Spliterator`接口的定义。
```
public interface Spliterator<T> {
    boolean tryAdvance(Consumer<? super T> action);
    Spliterator<T> trySplit();
    long estimateSize();
    int characteristics();
}
```
泛型`T`为`Spliterator`需要遍历的元素类型，`tryAdvance`方法当还有数据可以划分是返回
`true`，`trySplit`方法可以把一些元素划分出去给第二个`Spliterator`，让两个任务并行
执行；同时`estimateSize`方法会用来评估还剩下多少元素需要遍历，即使无法精确计算，能够
快速计算一个值也有利于让任务划分均匀一点。`characteristics`类似于收集器接口中的同名
方法，用于描述可分迭代器的一些划分特性，只不过返回的是`int`型而不是枚举集合。
#### Spliterator的拆分过程
将数据源拆分为多个子部分是一个递归的操作过程，首先对一个`Spliterator`调用`trySplit`，
生成第二个`Spliterator`，第二步对这两个`Spliterator`再分别调用`trySplit`，结果得到
四个`Spliterator`，最终框架会递归调用直到`trySplit`返回`null`，于是任务不会继续拆分，
整个任务的递归划分结束。而在任务拆分过程中，`characteristics`方法中的特性会对任务的
划分造成影响。
### 实现自己的Spliterator
todo
### 附录A 通过Benchmark框架评估性能
```
@State(Scope.Thread)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@Fork(value=2, jvmArgs={"-Xms4G", "-Xmx4G"})
@Measurement(iterations=2)
@Warmup(iterations=3)
public class ParallelStreamBenchmark {
    private static final long N = 10_000_000L;

    @Benchmark
    public long iterativeSum() {
        long result = 0;
        for (long i = 1L; i <= N; i++) {
            result += i;
        }
        return result;
    }

    @Benchmark
    public long sequentialSum() {
        return Stream.iterate( 1L, i -> i + 1 ).limit(N)
                     .reduce( 0L, Long::sum );
    }

    @Benchmark
    public long parallelSum() {
        return Stream.iterate(1L, i -> i + 1).limit(N)
                     .parallel().reduce( 0L, Long::sum);
    }

    @Benchmark
    public long rangedSum() {
        return LongStream.rangeClosed( 1, N ).reduce( 0L, Long::sum );
    }

    @Benchmark
    public long parallelRangedSum() {
        return LongStream.rangeClosed(1, N).parallel().reduce( 0L, Long::sum);
    }

    @TearDown(Level.Invocation)
    public void tearDown() {
        System.gc();
    }
}

<dependency>
  <groupId>org.openjdk.jmh</groupId>
  <artifactId>jmh-core</artifactId>
  <version>1.17.4</version>
</dependency>
<dependency>
  <groupId>org.openjdk.jmh</groupId>
  <artifactId>jmh-generator-annprocess</artifactId>
  <version>1.17.4</version>
</dependency>

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>3.1.0</version>
    <executions>
      <execution>
        <phase>package</phase>
        <goals>
          <goal>shade</goal>
        </goals>
        <configuration>
          <finalName>benchmarks</finalName>
          <transformers>
            <transformer implementation="org.apache.maven.plugins.shade.
                                         resource.ManifestResourceTransformer">
              <mainClass>org.openjdk.jmh.Main</mainClass>
            </transformer>
          </transformers>
        </configuration>
      </execution>
    </executions>
  </plugin>
```
使用`maven package`打包，然后通过运行`java -jar benchmarks.jar`执行基准测试。
或通过`IntelliJ IDEA`添加`org.openjdk.jmh.Main`类为执行主函数，在IDE中执行基准测试。
在我的机器运行最终运行结果如下。从下面测试可以看到，使用原生顺序累加方式是最快的，甚至
比通过分支/合并框架并行任务还要快。（疑问：why？我们期望`parallelRangedSum`最快）
```
# Run complete. Total time: 00:01:49
Benchmark                                  Mode  Cnt    Score    Error  Units
ParallelStreamBenchmark.iterativeSum       avgt    4    4.834 ±  0.727  ms/op
ParallelStreamBenchmark.parallelRangedSum  avgt    4    5.355 ± 22.860  ms/op
ParallelStreamBenchmark.parallelSum        avgt    4  153.851 ± 80.647  ms/op
ParallelStreamBenchmark.rangedSum          avgt    4    6.218 ±  0.669  ms/op
ParallelStreamBenchmark.sequentialSum      avgt    4  138.467 ± 64.056  ms/op
```
