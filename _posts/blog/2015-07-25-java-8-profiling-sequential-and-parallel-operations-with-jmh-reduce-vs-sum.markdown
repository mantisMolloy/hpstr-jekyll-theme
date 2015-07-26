---
layout: post
title: "Java 8: Profiling Sequential and Parallel Operations With JMH - A Lesson in JIT Inlining"
modified:
categories: blog
description: Profiling different methods of suming numbers 1 to 100,000,000
tags: [Jave 8, JMH, JIT]
image:
  feature: code.jpg
share: true
date: 2015-07-25T15:50:33+01:00
---

# Profilng Java 8 - When JIT Inlining Goes Wrong

By profling some simple methods we can get a good insight into how the Java 8 JVM works. Here we will compare and contrast methods that sum numbers using 

1. A thread safe AtomicLong
2. A custom reduce operation 
3. Java 8's built in sum() operation 

We will then see how using parallel() doesn't always produce better performance from these methods. Furthermore, we will see how the JIT compiler may not optimise your code well resulting in a degradation in performance - but we can over come this by setting certain flags in the JVM. 

## The Testcases

We will use JMH to profile the JVM. JMH is produced in house at Oracle and is one of the most reliable and accurate profilers. The 6 methods we want to test are

### Using an AtomicLong

Sum an AtomicLong.

{% highlight java %}
@Benchmark
@OutputTimeUnit(TimeUnit.SECONDS)
public long atomicSeq() {
  AtomicLong sum = new AtomicLong(0);
  LongStream.rangeClosed(1, LIMIT)
            .forEach(i -> sum.addAndGet(i));
  return sum.get();
}
{% endhighlight %}

{% highlight java %}
@Benchmark
@OutputTimeUnit(TimeUnit.SECONDS)
public long atomicPar(){
  AtomicLong sum = new AtomicLong(0);
  LongStream.rangeClosed(1, LIMIT)
            .parallel()
            .forEach(i -> sum.addAndGet(i));
  return sum.get();
}
{% endhighlight %}

### Using Custom Reduce

Adding numbers is associative so we can easily split up adding a stream of values using a reduce operation.

{% highlight java %}
@Benchmark
@OutputTimeUnit(TimeUnit.SECONDS)
public long reduceSeq(){
  return 
    LongStream.rangeClosed(1, LIMIT)
              .reduce((i,  j) -> i + j)
              .getAsLong();
}
{% endhighlight %}

{% highlight java %}	
@Benchmark
@OutputTimeUnit(TimeUnit.SECONDS)
public long reducePar(){
  return 
    LongStream.rangeClosed(1, LIMIT)
              .parallel()
              .reduce((i, j) -> i + j)
              .getAsLong();
}
{% endhighlight %}

### Using sum()

Java 8 has a built in reducing sum operation.

{% highlight java %}
@Benchmark
@OutputTimeUnit(TimeUnit.SECONDS)
public long sumSeq(){
  return 
    LongStream.rangeClosed(1, LIMIT)
              .sum();
}
{% endhighlight %}

{% highlight java %}
@Benchmark
@OutputTimeUnit(TimeUnit.SECONDS)
public long sumPar(){
  return 
    LongStream.rangeClosed(1, LIMIT)
              .parallel()
              .sum();
}
{% endhighlight %}

# Execution

I built the JMH test project with maven as per the <a href="http://openjdk.java.net/projects/code-tools/jmh//">JMH website</a>.

I added my own class `testStreams.Bench` to the projects src folder

{% highlight perl %}
mantis@mantis:~/test$ tree src
src
└── main
    └── java
        ├── org
        │   └── sample
        │       └── MyBenchmark.java
        └── testStreams
            └── Bench.java
{% endhighlight %}

I added the above methods to `Bench.java` and set `LIMIT = 1000_000_000L`

I built the project with `mvn clean install` and then used the follwong parameters when runnign the benchmark

{% highlight perl %}
mantis@mantis: java -jar target/benchmarks.jar testStreams.Bench -i 5 -wi 5 -f 1
{% endhighlight %}

# Results

### AtomicLong

|Benchmark		            |Mode |Iterations |Score(ops/s)|
|:--------------------------|-----|-----------|------------|
|testStreams.Bench.atomicPar|thrpt|5          |0.530       |
|testStreams.Bench.atomicSeq|thrpt|5          |1.801       |
{: .table}

As multiple threads are trying to access our sum variable we must use an AtomicLong - this prevents multiple threads accessing the variable and results in threads having to wait to access it. We see that using `parallel()` actually results in a worse performance due to the overhead of threads having to wait. This is abad way of summing values using streams.

### Custom Reduce Operation

|Benchmark		            |Mode |Iterations |Score(ops/s)|
|:--------------------------|-----|-----------|------------|
|testStreams.Bench.reducePar|thrpt|5          |15.166      |
|testStreams.Bench.reduceSeq|thrpt|5          |5.871       |
{: .table}

When we break the process of adding the numbers up we see much better results. In sequential mode we see a 5x improvement over using a single varaibale. I have allocated 3 cores to the virtual machine runnign the testbench. When we apply `parallel()` we see ~3x improvement in ops/s.

### Java 8 sum()

|Benchmark		         |Mode |Iterations |Score(ops/s)|
|:-----------------------|-----|-----------|------------|
|testStreams.Bench.sumPar|thrpt|5          |27.714      |
|testStreams.Bench.sumSeq|thrpt|5          |21.374      |
{: .table}

#### Analysis

Using Java 8's `sum()` operation results in better performance. The sequential `sumSeq()` method actually outperforms our custom parallel method `reducePar()`. However, we do not see a big improvement over `sumSeq()` when using `sumPar()`. Looking at the running of this benchmark we see

{% highlight  perl%}
# Run progress: 22.21% complete, ETA 00:02:32
# Fork: 1 of 1
# Warmup Iteration   1: 53.385 ops/s
# Warmup Iteration   2: 57.026 ops/s
# Warmup Iteration   3: 58.708 ops/s
# Warmup Iteration   4: 58.495 ops/s
# Warmup Iteration   5: 58.644 ops/s
Iteration   1: 58.574 ops/s
Iteration   2: 51.956 ops/s
Iteration   3: 9.345 ops/s
Iteration   4: 9.364 ops/s
Iteration   5: 9.332 ops/s
{% endhighlight %}

So why the sudden drop in ops/s? Well, the JIT compiler will inline frequently called methods to avoid the overhead of method invocation. However, methods that are too big cannot be inlined without bloating the call sites.

We can see how the JIT compiler compiles our code by using the following JVM flags `-XX:+UnlockDiagnosticVMOptions` `-XX:+PrintCompilation` `-XX:+PrintInlining`. We can see that once the inline limit is reached the JIT does not attempt to inline the pipeline denoted by `callee is to large`. For a full explantion and for  berevities sake see <a href="http://stackoverflow.com/questions/25847397/erratic-performance-of-arrays-stream-map-sum">This Stack Overflow post</a>.

We can improve performance by adding the `-XX:MaxInlineLevel=12` JVM flag.

{% highlight  perl%}
mantis@mantis:~/test$ java -XX:MaxInlineLevel=12 -jar target/benchmarks.jar testStreams.Bench -i 5 -wi 5 -f 1
# JMH 1.10.3 (released 8 days ago)
# VM version: JDK 1.8.0_45, VM 25.45-b02
# VM invoker: /usr/lib/jvm/java-8-oracle/jre/bin/java
# VM options: -XX:MaxInlineLevel=12
# Warmup: 5 iterations, 1 s each
# Measurement: 1000 iterations, 1 s each
# Timeout: 10 min per iteration
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Throughput, ops/time
# Benchmark: testStreams.Bench.sumPar

# Run progress: 0.00% complete, ETA 00:50:16
# Fork: 1 of 1
# Warmup Iteration   1: 53.888 ops/s
# Warmup Iteration   2: 58.328 ops/s
# Warmup Iteration   3: 58.468 ops/s
# Warmup Iteration   4: 58.455 ops/s
# Warmup Iteration   5: 57.937 ops/s
Iteration   1: 58.717 ops/s
Iteration   2: 59.494 ops/s
Iteration   3: 60.013 ops/s
Iteration   4: 59.506 ops/s
Iteration   5: 51.543 ops/s
{% endhighlight %}

Not shown here, but we can also improve things by turning off tiered compilation which `-XX:-TieredCompilation`.

### Java 8 sum() With Incresed Inline Level

|Benchmark		         |Mode |Iterations |Score(ops/s)|
|:-----------------------|-----|-----------|------------|
|testStreams.Bench.sumPar|thrpt|5          |57.364      |
|testStreams.Bench.sumSeq|thrpt|5          |21.052      |
{: .table}

We see ~3x in performance once the inlining issue has been resolved.