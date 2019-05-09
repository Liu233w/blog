---
layout: blog
date: 2019-05-09 20:37:52
categories:
- blog
tags:
- java
- 并发编程
title: Java 8 parallelStream 在大批量小任务上的性能陷阱
---

# tl;dr
Java 8 的 parallelStream 在运行大量的短任务（短时间执行的任务）之后会变成单线程运行，这是 parallelStream 自带的性能优化功能。但是在短任务和长任务交替执行时，这种优化会显著拖慢运行速度。

# 测试代码
我们使用下面的测试函数来进行说明。
```java
    private static void run(int shortTaskNumber, int longTaskNumber) {

        final ConcurrentHashMap<Integer, String> workingThreads = new ConcurrentHashMap<>();

        final ArrayList<Supplier<Integer>> inputs = new ArrayList<>();
        // 直接收集结果（大量的快速计算）
        inputs.addAll(IntStream.range(0, shortTaskNumber)
                .boxed()
                .map(a -> (Supplier<Integer>) () -> a)
                .collect(Collectors.toList()));
        // 模拟长时间运算（少量的慢速计算）
        inputs.addAll(IntStream.range(shortTaskNumber, shortTaskNumber + longTaskNumber)
                .boxed()
                .map(a -> (Supplier<Integer>) () -> {
                    final long beg = System.currentTimeMillis();
                    final long end = beg + 1000 * 10;
                    // 模拟计算密集型操作
                    // 其实这里换成 Thread.sleep(10000) 对结果也没影响
                    while (System.currentTimeMillis() < end) ;
                    return a;
                })
                .collect(Collectors.toList()));

        // 开始并发执行
        IntStream.range(0, inputs.size())
                .boxed()
                .parallel()
                .forEach(i -> {
                    workingThreads.put(i, Thread.currentThread().getName());
                    // 不显示不必要的信息
                    if (i >= shortTaskNumber) {
                        System.out.println("On " + i);
                        System.out.println(workingThreads);
                        System.out.println("-------------------------------------");
                    }
                    inputs.get(i).get();
                    workingThreads.remove(i);
                });
    }
```

根据代码所示，我们创建了`shortTaskNumber`个立刻能得到结果的任务，和`longTaskNumber`个需要长时间计算的任务，然后把这些任务放到一个 parallelStream 中并发执行。

# 少量任务的情况
本测试程序使用 OpenJDK 11 运行。

我们使用 `run(20, 20)` 来运行测试函数，得到结果：
```text
On 26
{26=main}
-------------------------------------
On 36
{36=ForkJoinPool.commonPool-worker-3, 26=main}
-------------------------------------
On 22
{36=ForkJoinPool.commonPool-worker-3, 22=ForkJoinPool.commonPool-worker-5, 26=main}
-------------------------------------
On 21
On 38
On 32
{32=ForkJoinPool.commonPool-worker-11, 36=ForkJoinPool.commonPool-worker-3, 21=ForkJoinPool.commonPool-worker-9, 22=ForkJoinPool.commonPool-worker-5, 38=ForkJoinPool.commonPool-worker-7, 26=main}
-------------------------------------
{32=ForkJoinPool.commonPool-worker-11, 36=ForkJoinPool.commonPool-worker-3, 21=ForkJoinPool.commonPool-worker-9, 22=ForkJoinPool.commonPool-worker-5, 38=ForkJoinPool.commonPool-worker-7, 26=main}
-------------------------------------
{32=ForkJoinPool.commonPool-worker-11, 36=ForkJoinPool.commonPool-worker-3, 21=ForkJoinPool.commonPool-worker-9, 22=ForkJoinPool.commonPool-worker-5, 38=ForkJoinPool.commonPool-worker-7, 26=main}
-------------------------------------
On 35
{32=ForkJoinPool.commonPool-worker-11, 35=ForkJoinPool.commonPool-worker-13, 36=ForkJoinPool.commonPool-worker-3, 21=ForkJoinPool.commonPool-worker-9, 22=ForkJoinPool.commonPool-worker-5, 38=ForkJoinPool.commonPool-worker-7, 26=main}
-------------------------------------
On 24
{32=ForkJoinPool.commonPool-worker-11, 35=ForkJoinPool.commonPool-worker-13, 36=ForkJoinPool.commonPool-worker-3, 21=ForkJoinPool.commonPool-worker-9, 22=ForkJoinPool.commonPool-worker-5, 38=ForkJoinPool.commonPool-worker-7, 24=ForkJoinPool.commonPool-worker-15, 26=main}
-------------------------------------
On 28
{32=ForkJoinPool.commonPool-worker-11, 35=ForkJoinPool.commonPool-worker-13, 21=ForkJoinPool.commonPool-worker-9, 22=ForkJoinPool.commonPool-worker-5, 38=ForkJoinPool.commonPool-worker-7, 24=ForkJoinPool.commonPool-worker-15, 28=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 20
On 25
{32=ForkJoinPool.commonPool-worker-11, 35=ForkJoinPool.commonPool-worker-13, 20=ForkJoinPool.commonPool-worker-5, 21=ForkJoinPool.commonPool-worker-9, 38=ForkJoinPool.commonPool-worker-7, 24=ForkJoinPool.commonPool-worker-15, 25=main, 28=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 34
{34=ForkJoinPool.commonPool-worker-11, 35=ForkJoinPool.commonPool-worker-13, 20=ForkJoinPool.commonPool-worker-5, 21=ForkJoinPool.commonPool-worker-9, 38=ForkJoinPool.commonPool-worker-7, 24=ForkJoinPool.commonPool-worker-15, 25=main, 28=ForkJoinPool.commonPool-worker-3}
-------------------------------------
{32=ForkJoinPool.commonPool-worker-11, 35=ForkJoinPool.commonPool-worker-13, 20=ForkJoinPool.commonPool-worker-5, 21=ForkJoinPool.commonPool-worker-9, 38=ForkJoinPool.commonPool-worker-7, 24=ForkJoinPool.commonPool-worker-15, 25=main, 28=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 37
{34=ForkJoinPool.commonPool-worker-11, 35=ForkJoinPool.commonPool-worker-13, 20=ForkJoinPool.commonPool-worker-5, 37=ForkJoinPool.commonPool-worker-9, 39=ForkJoinPool.commonPool-worker-7, 24=ForkJoinPool.commonPool-worker-15, 25=main, 28=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 39
{34=ForkJoinPool.commonPool-worker-11, 35=ForkJoinPool.commonPool-worker-13, 20=ForkJoinPool.commonPool-worker-5, 37=ForkJoinPool.commonPool-worker-9, 39=ForkJoinPool.commonPool-worker-7, 24=ForkJoinPool.commonPool-worker-15, 25=main, 28=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 23
{34=ForkJoinPool.commonPool-worker-11, 35=ForkJoinPool.commonPool-worker-13, 20=ForkJoinPool.commonPool-worker-5, 37=ForkJoinPool.commonPool-worker-9, 39=ForkJoinPool.commonPool-worker-7, 23=ForkJoinPool.commonPool-worker-15, 25=main, 28=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 27
{34=ForkJoinPool.commonPool-worker-11, 20=ForkJoinPool.commonPool-worker-5, 37=ForkJoinPool.commonPool-worker-9, 39=ForkJoinPool.commonPool-worker-7, 23=ForkJoinPool.commonPool-worker-15, 25=main, 27=ForkJoinPool.commonPool-worker-13, 28=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 29
{34=ForkJoinPool.commonPool-worker-11, 20=ForkJoinPool.commonPool-worker-5, 37=ForkJoinPool.commonPool-worker-9, 39=ForkJoinPool.commonPool-worker-7, 23=ForkJoinPool.commonPool-worker-15, 25=main, 27=ForkJoinPool.commonPool-worker-13, 29=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 33
{33=ForkJoinPool.commonPool-worker-11, 20=ForkJoinPool.commonPool-worker-5, 37=ForkJoinPool.commonPool-worker-9, 39=ForkJoinPool.commonPool-worker-7, 23=ForkJoinPool.commonPool-worker-15, 25=main, 27=ForkJoinPool.commonPool-worker-13, 29=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 31
{33=ForkJoinPool.commonPool-worker-11, 37=ForkJoinPool.commonPool-worker-9, 39=ForkJoinPool.commonPool-worker-7, 23=ForkJoinPool.commonPool-worker-15, 27=ForkJoinPool.commonPool-worker-13, 29=ForkJoinPool.commonPool-worker-3, 31=ForkJoinPool.commonPool-worker-5}
-------------------------------------
On 30
{33=ForkJoinPool.commonPool-worker-11, 23=ForkJoinPool.commonPool-worker-15, 27=ForkJoinPool.commonPool-worker-13, 29=ForkJoinPool.commonPool-worker-3, 30=ForkJoinPool.commonPool-worker-9, 31=ForkJoinPool.commonPool-worker-5}
-------------------------------------
```

可以看到，这些长时间的任务（index大于20的）都是并发运行的，这是我们需要的结果。

然而，在运行大量的任务时，情况却并不是这样。

# 大量任务的情况

使用 `run(20000, 100)` 得到的运行结果：
```text
On 20000
{20000=ForkJoinPool.commonPool-worker-15, 12934=main, 3081=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 20001
{20001=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 20002
{20002=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 20003
{20003=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 20004
{20004=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 20005
{20005=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 20006
{20006=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 20007
{20007=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 20008
{20008=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 20009
{20009=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 20010
{20010=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 20011
{20011=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 20012
{20012=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 20013
{20013=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 20014
{20014=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 20015
{20015=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 20016
{20016=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 20017
{20017=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 20018
{20018=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 20019
{20019=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 20020
{20020=ForkJoinPool.commonPool-worker-15}
-------------------------------------

（省略剩下的输出）
```

我们在 20000 个轻计算量的任务中插入了 200 个需要长时间计算的任务，却发现这些任务并没有并行执行。我推测这是因为 parallelStream 在发现前面的任务执行速度很快之后，会推测全部的任务都是这样的短任务，并将剩下的任务放到单个线程中进行计算。

# 该行为的触发时机

为了印证这个想法，我完善了一下测试程序，得到了下面的运行结果。

其中四个数值（最小值，最大值，平均数，中位数）指的是“在运行某个长任务时，并发线程的数量”。

|shortTaskNumber|longTaskNumber|最小值|最大值|平均数|中位数|
|--|--|--|
|5|5|5|5|5.0|5.0|
|20|5|5|5|5.0|5.0|
|50|5|2|5|3.8|5.0|
|100|5|1|3|2.4|3.0|
|200|5|1|1|1.0|1.0|
|100|10|1|5|3.9|4.5|
|200|10|1|2|1.6|2.0|
|100|20|3|6|4.95|5.0|
|200|20|2|3|2.8|3.0|
|300|20|2|2|2.0|2.0|
|400|20|1|2|1.6|2.0|
|500|20|1|2|1.3|1.0|
|600|20|1|1|1.0|1.0|

可见，parallelStream 并不是完全按照处理器的数量来做的并行。

本测试的结果并不稳定，因为 parallelStream 不是按照顺序进行计算的，
短任务和长任务完全可能混在一起运行。同时，`shortTaskNumber/longTaskNumber` 的结果越大，parallelStream 就会使用越小的线程数来运行任务。

另外，我也尝试过把 parallelStream 使用的线程池更换成 FixedThreadPool，得到的结果和上面的相似，可见这也不是 ForkJoinPool 的问题。

# 该特性对计算密集型任务的影响

一般来说，大家都是使用 parallelStream 来进行计算密集型任务的。
如果拆分出来的每个任务的运行时间都差不多，那么使用 parallelStream
没有什么问题，一般的计算密集型任务可能都有这个特点。

我是在给一个科学计算项目编写代码时发现的这个问题。在那个代码中，由于每个任务的运行时间都比较长，我们会把中间结果缓存到文件中，比如下面的伪代码：
```java
list.parallelStream()
.map(...)
.map(input -> {
    if (isCached(input)) {
        return loadCache(input);
    } else {
        var res = resolve(input);
        saveCache(input, res);
        return res;
    }
})
.map(...)
```

如果 list 中有几千个数据，同时其中大部分都被缓存进了文件中，则中间部分的 map 会包含大量的短任务（从文件中读取数据），而jdk的优化会让其中的长任务（直接计算）变成单线程执行。这样是非常麻烦的。

对于这种情况，我还没有什么比较好的解决办法。现在只能把部分受影响的代码直接改成调用 ForkJoinPool 来获取稳定的并发。

# 附录：测试程序和完整的运行结果

测试程序：
```java
import java.util.ArrayList;
import java.util.Arrays;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.function.Supplier;
import java.util.stream.Collectors;
import java.util.stream.IntStream;

public class Main {

    public static void main(String[] args) {

        run(5, 5);
        run(20, 5);
        run(50, 5);
        run(100, 5);
        run(200, 5);

        run(100, 10);
        run(200, 10);

        run(100, 20);
        run(200, 20);
        run(300, 20);
        run(400, 20);
        run(500, 20);
        run(600, 20);
    }

    private static void run(int shortTaskNumber, int longTaskNumber) {

        System.out.println();
        System.out.println("shortTaskNumber: " + shortTaskNumber + ", longTaskNumber: " + longTaskNumber);

        final ConcurrentHashMap<Integer, String> workingThreads = new ConcurrentHashMap<>();
        final ConcurrentLinkedQueue<Integer> parallelThreadNumber = new ConcurrentLinkedQueue<>();

        final ArrayList<Supplier<Integer>> inputs = new ArrayList<>();
        // 直接收集结果（大量的快速获得的数据）
        inputs.addAll(IntStream.range(0, shortTaskNumber)
                .boxed()
                .map(a -> (Supplier<Integer>) () -> a)
                .collect(Collectors.toList()));
        // 模拟长时间运算（少量的慢速计算的数据）
        inputs.addAll(IntStream.range(shortTaskNumber, shortTaskNumber + longTaskNumber)
                .boxed()
                .map(a -> (Supplier<Integer>) () -> {
                    final long beg = System.currentTimeMillis();
                    final long end = beg + 1000 * 10;
                    // 模拟计算密集型操作
                    // 其实这里换成 Thread.sleep(10000) 对结果也没影响
                    while (System.currentTimeMillis() < end) ;
                    return a;
                })
                .collect(Collectors.toList()));

        IntStream.range(0, inputs.size())
                .boxed()
                .parallel()
                .forEach(i -> {
                    workingThreads.put(i, Thread.currentThread().getName());
                    inputs.get(i).get();
                    if (i >= shortTaskNumber) {
                        // 固定一下当前状态下的线程，防止统计的数量和输出的日志不一致
                        final String output = workingThreads.toString();
                        // 字符串中等号的数量就是当前的并发线程数
                        parallelThreadNumber.add((int) output.chars().filter(c -> c == '=').count());
                        System.out.println("On " + i);
                        System.out.println(workingThreads);
                        System.out.println("-------------------------------------");
                    }
                    workingThreads.remove(i);
                });

        final int[] ints = parallelThreadNumber
                .stream()
                .mapToInt(a -> a)
                .toArray();
        Arrays.sort(ints);

        final int min = ints[0];
        final int max = ints[ints.length - 1];
        final double average = Arrays.stream(ints).average().getAsDouble();
        final double middle;
        if (ints.length % 2 == 0) {
            final int i = ints.length / 2;
            middle = (ints[i - 1] + ints[i + 1]) / 2.0;
        } else {
            middle = ints[ints.length / 2];
        }

        System.out.println("长任务中的并发线程数： Min: " + min + ", Max: " + max + ", 平均数: " + average + ", 中位数: " + middle);
    }
}
```

完整运行结果：
```text
shortTaskNumber: 5, longTaskNumber: 5
On 7
On 8
On 5
On 6
{5=ForkJoinPool.commonPool-worker-9, 6=main, 7=ForkJoinPool.commonPool-worker-3, 8=ForkJoinPool.commonPool-worker-5, 9=ForkJoinPool.commonPool-worker-7}
-------------------------------------
{5=ForkJoinPool.commonPool-worker-9, 6=main, 7=ForkJoinPool.commonPool-worker-3, 8=ForkJoinPool.commonPool-worker-5, 9=ForkJoinPool.commonPool-worker-7}
-------------------------------------
{5=ForkJoinPool.commonPool-worker-9, 6=main, 7=ForkJoinPool.commonPool-worker-3, 8=ForkJoinPool.commonPool-worker-5, 9=ForkJoinPool.commonPool-worker-7}
-------------------------------------
On 9
{5=ForkJoinPool.commonPool-worker-9, 6=main, 7=ForkJoinPool.commonPool-worker-3, 8=ForkJoinPool.commonPool-worker-5, 9=ForkJoinPool.commonPool-worker-7}
-------------------------------------
{6=main, 9=ForkJoinPool.commonPool-worker-7}
-------------------------------------
长任务中的并发线程数： Min: 5, Max: 5, 平均数: 5.0, 中位数: 5.0

shortTaskNumber: 20, longTaskNumber: 5
On 24
On 23
On 20
On 21
On 22
{20=ForkJoinPool.commonPool-worker-7, 21=ForkJoinPool.commonPool-worker-5, 22=ForkJoinPool.commonPool-worker-3, 23=ForkJoinPool.commonPool-worker-15, 24=ForkJoinPool.commonPool-worker-9}
-------------------------------------
{20=ForkJoinPool.commonPool-worker-7, 21=ForkJoinPool.commonPool-worker-5, 22=ForkJoinPool.commonPool-worker-3, 23=ForkJoinPool.commonPool-worker-15, 24=ForkJoinPool.commonPool-worker-9}
-------------------------------------
{20=ForkJoinPool.commonPool-worker-7, 21=ForkJoinPool.commonPool-worker-5, 22=ForkJoinPool.commonPool-worker-3, 23=ForkJoinPool.commonPool-worker-15, 24=ForkJoinPool.commonPool-worker-9}
-------------------------------------
{20=ForkJoinPool.commonPool-worker-7, 21=ForkJoinPool.commonPool-worker-5, 22=ForkJoinPool.commonPool-worker-3, 23=ForkJoinPool.commonPool-worker-15, 24=ForkJoinPool.commonPool-worker-9}
-------------------------------------
{20=ForkJoinPool.commonPool-worker-7, 21=ForkJoinPool.commonPool-worker-5, 22=ForkJoinPool.commonPool-worker-3, 23=ForkJoinPool.commonPool-worker-15, 24=ForkJoinPool.commonPool-worker-9}
-------------------------------------
长任务中的并发线程数： Min: 5, Max: 5, 平均数: 5.0, 中位数: 5.0

shortTaskNumber: 50, longTaskNumber: 5
On 52
On 53
On 50
{50=ForkJoinPool.commonPool-worker-9, 51=ForkJoinPool.commonPool-worker-15, 52=ForkJoinPool.commonPool-worker-13, 53=ForkJoinPool.commonPool-worker-11, 54=ForkJoinPool.commonPool-worker-7}
-------------------------------------
{50=ForkJoinPool.commonPool-worker-9, 51=ForkJoinPool.commonPool-worker-15, 52=ForkJoinPool.commonPool-worker-13, 53=ForkJoinPool.commonPool-worker-11, 54=ForkJoinPool.commonPool-worker-7}
-------------------------------------
{50=ForkJoinPool.commonPool-worker-9, 51=ForkJoinPool.commonPool-worker-15, 52=ForkJoinPool.commonPool-worker-13, 53=ForkJoinPool.commonPool-worker-11, 54=ForkJoinPool.commonPool-worker-7}
-------------------------------------
On 51
{51=ForkJoinPool.commonPool-worker-15, 54=ForkJoinPool.commonPool-worker-7}
-------------------------------------
On 54
{54=ForkJoinPool.commonPool-worker-7}
-------------------------------------
长任务中的并发线程数： Min: 2, Max: 5, 平均数: 3.8, 中位数: 5.0

shortTaskNumber: 100, longTaskNumber: 5
On 101
{100=ForkJoinPool.commonPool-worker-5, 101=ForkJoinPool.commonPool-worker-11, 103=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 100
On 103
{100=ForkJoinPool.commonPool-worker-5, 102=ForkJoinPool.commonPool-worker-11, 103=ForkJoinPool.commonPool-worker-9}
-------------------------------------
{100=ForkJoinPool.commonPool-worker-5, 102=ForkJoinPool.commonPool-worker-11, 103=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 102
{102=ForkJoinPool.commonPool-worker-11, 104=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 104
{104=ForkJoinPool.commonPool-worker-9}
-------------------------------------
长任务中的并发线程数： Min: 1, Max: 3, 平均数: 2.4, 中位数: 3.0

shortTaskNumber: 200, longTaskNumber: 5
On 200
{200=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 201
{201=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 202
{202=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 203
{203=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 204
{204=ForkJoinPool.commonPool-worker-15}
-------------------------------------
长任务中的并发线程数： Min: 1, Max: 1, 平均数: 1.0, 中位数: 1.0

shortTaskNumber: 100, longTaskNumber: 10
On 103
On 101
On 106
On 100
{100=ForkJoinPool.commonPool-worker-7, 101=ForkJoinPool.commonPool-worker-5, 103=ForkJoinPool.commonPool-worker-9, 106=ForkJoinPool.commonPool-worker-13, 108=ForkJoinPool.commonPool-worker-11}
-------------------------------------
{100=ForkJoinPool.commonPool-worker-7, 101=ForkJoinPool.commonPool-worker-5, 103=ForkJoinPool.commonPool-worker-9, 106=ForkJoinPool.commonPool-worker-13, 108=ForkJoinPool.commonPool-worker-11}
-------------------------------------
{100=ForkJoinPool.commonPool-worker-7, 101=ForkJoinPool.commonPool-worker-5, 103=ForkJoinPool.commonPool-worker-9, 106=ForkJoinPool.commonPool-worker-13, 108=ForkJoinPool.commonPool-worker-11}
-------------------------------------
{100=ForkJoinPool.commonPool-worker-7, 101=ForkJoinPool.commonPool-worker-5, 103=ForkJoinPool.commonPool-worker-9, 106=ForkJoinPool.commonPool-worker-13, 108=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 108
{102=ForkJoinPool.commonPool-worker-5, 104=ForkJoinPool.commonPool-worker-9, 107=ForkJoinPool.commonPool-worker-13, 108=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 107
On 104
On 102
{102=ForkJoinPool.commonPool-worker-5, 104=ForkJoinPool.commonPool-worker-9, 107=ForkJoinPool.commonPool-worker-13, 109=ForkJoinPool.commonPool-worker-11}
-------------------------------------
{102=ForkJoinPool.commonPool-worker-5, 104=ForkJoinPool.commonPool-worker-9, 107=ForkJoinPool.commonPool-worker-13, 109=ForkJoinPool.commonPool-worker-11}
{102=ForkJoinPool.commonPool-worker-5, 104=ForkJoinPool.commonPool-worker-9, 107=ForkJoinPool.commonPool-worker-13, 109=ForkJoinPool.commonPool-worker-11}
-------------------------------------
-------------------------------------
On 109
{105=ForkJoinPool.commonPool-worker-9, 109=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 105
{105=ForkJoinPool.commonPool-worker-9}
-------------------------------------
长任务中的并发线程数： Min: 1, Max: 5, 平均数: 3.9, 中位数: 4.5

shortTaskNumber: 200, longTaskNumber: 10
On 203
{200=ForkJoinPool.commonPool-worker-13, 203=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 200
{200=ForkJoinPool.commonPool-worker-13, 204=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 204
{201=ForkJoinPool.commonPool-worker-13, 204=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 201
{201=ForkJoinPool.commonPool-worker-13, 205=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 205
{202=ForkJoinPool.commonPool-worker-13, 205=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 202
{202=ForkJoinPool.commonPool-worker-13, 206=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 206
{206=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 207
{207=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 208
{208=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 209
{209=ForkJoinPool.commonPool-worker-11}
-------------------------------------
长任务中的并发线程数： Min: 1, Max: 2, 平均数: 1.6, 中位数: 2.0

shortTaskNumber: 100, longTaskNumber: 20
On 108
{112=ForkJoinPool.commonPool-worker-5, 100=ForkJoinPool.commonPool-worker-9, 116=ForkJoinPool.commonPool-worker-7, 101=ForkJoinPool.commonPool-worker-15, 105=ForkJoinPool.commonPool-worker-13, 108=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 112
On 100
On 105
On 101
On 116
{112=ForkJoinPool.commonPool-worker-5, 100=ForkJoinPool.commonPool-worker-9, 116=ForkJoinPool.commonPool-worker-7, 101=ForkJoinPool.commonPool-worker-15, 105=ForkJoinPool.commonPool-worker-13, 109=ForkJoinPool.commonPool-worker-11}
-------------------------------------
{112=ForkJoinPool.commonPool-worker-5, 100=ForkJoinPool.commonPool-worker-9, 116=ForkJoinPool.commonPool-worker-7, 101=ForkJoinPool.commonPool-worker-15, 105=ForkJoinPool.commonPool-worker-13, 109=ForkJoinPool.commonPool-worker-11}
-------------------------------------
{112=ForkJoinPool.commonPool-worker-5, 100=ForkJoinPool.commonPool-worker-9, 116=ForkJoinPool.commonPool-worker-7, 101=ForkJoinPool.commonPool-worker-15, 105=ForkJoinPool.commonPool-worker-13, 109=ForkJoinPool.commonPool-worker-11}
-------------------------------------
{112=ForkJoinPool.commonPool-worker-5, 100=ForkJoinPool.commonPool-worker-9, 116=ForkJoinPool.commonPool-worker-7, 101=ForkJoinPool.commonPool-worker-15, 105=ForkJoinPool.commonPool-worker-13, 109=ForkJoinPool.commonPool-worker-11}
-------------------------------------
{112=ForkJoinPool.commonPool-worker-5, 100=ForkJoinPool.commonPool-worker-9, 116=ForkJoinPool.commonPool-worker-7, 101=ForkJoinPool.commonPool-worker-15, 105=ForkJoinPool.commonPool-worker-13, 109=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 109
{113=ForkJoinPool.commonPool-worker-5, 117=ForkJoinPool.commonPool-worker-7, 102=ForkJoinPool.commonPool-worker-15, 106=ForkJoinPool.commonPool-worker-13, 109=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 106
{113=ForkJoinPool.commonPool-worker-5, 117=ForkJoinPool.commonPool-worker-7, 102=ForkJoinPool.commonPool-worker-15, 106=ForkJoinPool.commonPool-worker-13, 110=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 117
{113=ForkJoinPool.commonPool-worker-5, 117=ForkJoinPool.commonPool-worker-7, 102=ForkJoinPool.commonPool-worker-15, 107=ForkJoinPool.commonPool-worker-13, 110=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 102
On 113
{113=ForkJoinPool.commonPool-worker-5, 102=ForkJoinPool.commonPool-worker-15, 118=ForkJoinPool.commonPool-worker-7, 107=ForkJoinPool.commonPool-worker-13, 110=ForkJoinPool.commonPool-worker-11}
-------------------------------------
{113=ForkJoinPool.commonPool-worker-5, 102=ForkJoinPool.commonPool-worker-15, 118=ForkJoinPool.commonPool-worker-7, 107=ForkJoinPool.commonPool-worker-13, 110=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 110
{114=ForkJoinPool.commonPool-worker-5, 118=ForkJoinPool.commonPool-worker-7, 103=ForkJoinPool.commonPool-worker-15, 107=ForkJoinPool.commonPool-worker-13, 110=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 103
{114=ForkJoinPool.commonPool-worker-5, 118=ForkJoinPool.commonPool-worker-7, 103=ForkJoinPool.commonPool-worker-15, 107=ForkJoinPool.commonPool-worker-13, 111=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 114
On 107
{114=ForkJoinPool.commonPool-worker-5, 118=ForkJoinPool.commonPool-worker-7, 104=ForkJoinPool.commonPool-worker-15, 107=ForkJoinPool.commonPool-worker-13, 111=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 118
{114=ForkJoinPool.commonPool-worker-5, 118=ForkJoinPool.commonPool-worker-7, 104=ForkJoinPool.commonPool-worker-15, 107=ForkJoinPool.commonPool-worker-13, 111=ForkJoinPool.commonPool-worker-11}
-------------------------------------
{114=ForkJoinPool.commonPool-worker-5, 118=ForkJoinPool.commonPool-worker-7, 104=ForkJoinPool.commonPool-worker-15, 111=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 111
{115=ForkJoinPool.commonPool-worker-5, 119=ForkJoinPool.commonPool-worker-7, 104=ForkJoinPool.commonPool-worker-15, 111=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 119
{115=ForkJoinPool.commonPool-worker-5, 119=ForkJoinPool.commonPool-worker-7, 104=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 115
{115=ForkJoinPool.commonPool-worker-5, 104=ForkJoinPool.commonPool-worker-15}
-------------------------------------
On 104
{104=ForkJoinPool.commonPool-worker-15}
-------------------------------------
长任务中的并发线程数： Min: 3, Max: 6, 平均数: 4.95, 中位数: 5.0

shortTaskNumber: 200, longTaskNumber: 20
On 200
{213=ForkJoinPool.commonPool-worker-11, 200=ForkJoinPool.commonPool-worker-5, 206=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 213
{213=ForkJoinPool.commonPool-worker-11, 201=ForkJoinPool.commonPool-worker-5, 206=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 206
{214=ForkJoinPool.commonPool-worker-11, 201=ForkJoinPool.commonPool-worker-5, 206=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 201
{214=ForkJoinPool.commonPool-worker-11, 201=ForkJoinPool.commonPool-worker-5, 207=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 214
On 207
{214=ForkJoinPool.commonPool-worker-11, 202=ForkJoinPool.commonPool-worker-5, 207=ForkJoinPool.commonPool-worker-13}
-------------------------------------
{214=ForkJoinPool.commonPool-worker-11, 202=ForkJoinPool.commonPool-worker-5, 207=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 202
{208=ForkJoinPool.commonPool-worker-13, 215=ForkJoinPool.commonPool-worker-11, 202=ForkJoinPool.commonPool-worker-5}
-------------------------------------
On 215
{208=ForkJoinPool.commonPool-worker-13, 215=ForkJoinPool.commonPool-worker-11, 203=ForkJoinPool.commonPool-worker-5}
-------------------------------------
On 208
{208=ForkJoinPool.commonPool-worker-13, 216=ForkJoinPool.commonPool-worker-11, 203=ForkJoinPool.commonPool-worker-5}
-------------------------------------
On 203
{209=ForkJoinPool.commonPool-worker-13, 216=ForkJoinPool.commonPool-worker-11, 203=ForkJoinPool.commonPool-worker-5}
-------------------------------------
On 209
{209=ForkJoinPool.commonPool-worker-13, 216=ForkJoinPool.commonPool-worker-11, 204=ForkJoinPool.commonPool-worker-5}
-------------------------------------
On 216
{210=ForkJoinPool.commonPool-worker-13, 216=ForkJoinPool.commonPool-worker-11, 204=ForkJoinPool.commonPool-worker-5}
-------------------------------------
On 204
{210=ForkJoinPool.commonPool-worker-13, 217=ForkJoinPool.commonPool-worker-11, 204=ForkJoinPool.commonPool-worker-5}
-------------------------------------
On 210
{210=ForkJoinPool.commonPool-worker-13, 217=ForkJoinPool.commonPool-worker-11, 205=ForkJoinPool.commonPool-worker-5}
-------------------------------------
On 217
{211=ForkJoinPool.commonPool-worker-13, 217=ForkJoinPool.commonPool-worker-11, 205=ForkJoinPool.commonPool-worker-5}
-------------------------------------
On 205
{211=ForkJoinPool.commonPool-worker-13, 218=ForkJoinPool.commonPool-worker-11, 205=ForkJoinPool.commonPool-worker-5}
-------------------------------------
On 211
{211=ForkJoinPool.commonPool-worker-13, 218=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 218
{212=ForkJoinPool.commonPool-worker-13, 218=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 219
{212=ForkJoinPool.commonPool-worker-13, 219=ForkJoinPool.commonPool-worker-11}
-------------------------------------
On 212
{212=ForkJoinPool.commonPool-worker-13}
-------------------------------------
长任务中的并发线程数： Min: 2, Max: 3, 平均数: 2.8, 中位数: 3.0

shortTaskNumber: 300, longTaskNumber: 20
On 310
{310=ForkJoinPool.commonPool-worker-7, 300=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 300
{311=ForkJoinPool.commonPool-worker-7, 300=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 311
{311=ForkJoinPool.commonPool-worker-7, 301=ForkJoinPool.commonPool-worker-9}
On 301
{311=ForkJoinPool.commonPool-worker-7, 301=ForkJoinPool.commonPool-worker-9}
-------------------------------------
-------------------------------------
On 312
{312=ForkJoinPool.commonPool-worker-7, 302=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 302
{313=ForkJoinPool.commonPool-worker-7, 302=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 313
{313=ForkJoinPool.commonPool-worker-7, 303=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 303
{314=ForkJoinPool.commonPool-worker-7, 303=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 314
{304=ForkJoinPool.commonPool-worker-9, 314=ForkJoinPool.commonPool-worker-7}
-------------------------------------
On 304
{304=ForkJoinPool.commonPool-worker-9, 315=ForkJoinPool.commonPool-worker-7}
-------------------------------------
On 315
{305=ForkJoinPool.commonPool-worker-9, 315=ForkJoinPool.commonPool-worker-7}
-------------------------------------
On 305
{305=ForkJoinPool.commonPool-worker-9, 316=ForkJoinPool.commonPool-worker-7}
-------------------------------------
On 306
{306=ForkJoinPool.commonPool-worker-9, 316=ForkJoinPool.commonPool-worker-7}
-------------------------------------
On 316
{307=ForkJoinPool.commonPool-worker-9, 316=ForkJoinPool.commonPool-worker-7}
-------------------------------------
On 317
{307=ForkJoinPool.commonPool-worker-9, 317=ForkJoinPool.commonPool-worker-7}
-------------------------------------
On 307
{307=ForkJoinPool.commonPool-worker-9, 318=ForkJoinPool.commonPool-worker-7}
-------------------------------------
On 318
{308=ForkJoinPool.commonPool-worker-9, 318=ForkJoinPool.commonPool-worker-7}
-------------------------------------
On 308
{308=ForkJoinPool.commonPool-worker-9, 319=ForkJoinPool.commonPool-worker-7}
-------------------------------------
On 319
{309=ForkJoinPool.commonPool-worker-9, 319=ForkJoinPool.commonPool-worker-7}
-------------------------------------
On 309
{309=ForkJoinPool.commonPool-worker-9}
-------------------------------------
长任务中的并发线程数： Min: 2, Max: 2, 平均数: 2.0, 中位数: 2.0

shortTaskNumber: 400, longTaskNumber: 20
On 406
{400=ForkJoinPool.commonPool-worker-9, 406=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 400
{400=ForkJoinPool.commonPool-worker-9, 407=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 407
{401=ForkJoinPool.commonPool-worker-9, 407=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 401
{401=ForkJoinPool.commonPool-worker-9, 408=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 408
{402=ForkJoinPool.commonPool-worker-9, 408=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 402
{402=ForkJoinPool.commonPool-worker-9, 409=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 409
{403=ForkJoinPool.commonPool-worker-9, 409=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 403
{403=ForkJoinPool.commonPool-worker-9, 410=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 410
{404=ForkJoinPool.commonPool-worker-9, 410=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 404
{404=ForkJoinPool.commonPool-worker-9, 411=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 411
{405=ForkJoinPool.commonPool-worker-9, 411=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 405
{405=ForkJoinPool.commonPool-worker-9, 412=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 412
{412=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 413
{413=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 414
{414=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 415
{415=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 416
{416=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 417
{417=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 418
{418=ForkJoinPool.commonPool-worker-13}
-------------------------------------
On 419
{419=ForkJoinPool.commonPool-worker-13}
-------------------------------------
长任务中的并发线程数： Min: 1, Max: 2, 平均数: 1.6, 中位数: 2.0

shortTaskNumber: 500, longTaskNumber: 20
On 503
{500=ForkJoinPool.commonPool-worker-3, 503=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 500
{500=ForkJoinPool.commonPool-worker-3, 504=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 504
{501=ForkJoinPool.commonPool-worker-3, 504=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 501
{501=ForkJoinPool.commonPool-worker-3, 505=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 505
{502=ForkJoinPool.commonPool-worker-3, 505=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 502
{502=ForkJoinPool.commonPool-worker-3, 506=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 506
{506=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 507
{507=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 508
{508=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 509
{509=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 510
{510=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 511
{511=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 512
{512=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 513
{513=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 514
{514=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 515
{515=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 516
{516=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 517
{517=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 518
{518=ForkJoinPool.commonPool-worker-9}
-------------------------------------
On 519
{519=ForkJoinPool.commonPool-worker-9}
-------------------------------------
长任务中的并发线程数： Min: 1, Max: 2, 平均数: 1.3, 中位数: 1.0

shortTaskNumber: 600, longTaskNumber: 20
On 600
{600=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 601
{601=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 602
{602=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 603
{603=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 604
{604=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 605
{605=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 606
{606=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 607
{607=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 608
{608=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 609
{609=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 610
{610=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 611
{611=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 612
{612=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 613
{613=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 614
{614=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 615
{615=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 616
{616=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 617
{617=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 618
{618=ForkJoinPool.commonPool-worker-3}
-------------------------------------
On 619
{619=ForkJoinPool.commonPool-worker-3}
-------------------------------------
长任务中的并发线程数： Min: 1, Max: 1, 平均数: 1.0, 中位数: 1.0
```
