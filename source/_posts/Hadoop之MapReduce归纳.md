---
title: Hadoop之MapReduce归纳
date: 2020-02-21 19:30:31
categories: Hadoop
tags: Hadoop
---

# 前言
> Hadoop1.x可简单分为**HDFS**和**MapReduce**
> 本文归纳了自己在学习过程中有关**MapReduce**的知识

<!--more-->

# MapReduce的执行过程

- 首先是对文件进行**分块（split）**操作，块的大小一般和HDFS的块大小相同，每一个块对应一个Map任务
- 每一个块按照RecordReader（RR）定义的规则**读取**数据，再交给对应的Map端处理。其中，RR默认按行读取数据，读取的内容相对文件开头的偏移量作为key，读取的那一行作为value输入给Map。
- Map端按照业务逻辑，**处理**输入的kv，并生成新的kv交给Shuffle，生成<key，value list>
- Reduce端提取对应的数据，按照业务逻辑处理<key，value list>，并**写入**HDFS。

# Shuffle

Shuffle过程是MapReduce的核心，可分为Map端的Shuffle和Reduce端的Shuffle，但都是一个缓存-溢写（分区，排序，合并）-归并的过程，下面来详细看下。

## Map端Shuffle

### 分区（Partition）

对于Map输出的每一个键值对，系统都会给定一个Partition，Partition值默认是通过**计算key的hash**值后对Reduce task的数量取模获得。如果一个键值对的Partition值为1，意味着这个键值对会交给第一个Reducer处理。

### 缓存（Buff in memory）

分区完成后的数据将被写入到内存缓冲区，缓冲区的作用是批量收集Map结果，减少磁盘IO的影响。

缓冲区是一个**环形数据结构**，也叫作环形缓冲区，**默认分配100MB的大小，设置溢写比例为0.8**。当达到溢写比例或者Map端输入完毕后，就将进行溢写操作，将结果写入本地磁盘。

### 排序 （Sort）

溢写的操作并不是简单粗暴的写，而是首先要进行排序，这个排序是预先定义好的，按照**快排**的方式将key按照**字典序**进行排列，因此每一个分区的结果都会是有序的。

另外，Sort和接下来的溢写，是**单独启动一个进程**来执行，并不会影响环形缓冲区的继续写入。

### 合并（Combine）

合并（Combine）是为减小文件大小，缓解IO压力，将形如<a,1>,<a,1>这样的值求和变成<a,2>。

合并这个操作**不是一定会有**，程序中有**两个阶段**可能会执行combine操作：

1. Map输出数据根据分区排序完成后，在写入文件之前会执行一次combine操作（前提是作业中设置了这个操作）；
2. 如果Map输出比较大，溢出文件个数大于3，在后面说到的归并这个过程中，还会执行Combine操作；

另外要注意不能因为Combine影响业务逻辑，比如是为了求平均值，就不能进行合并。

### 溢写（Spill）

排序完成的数据就会被写入磁盘。如果文件较大，一个Map进程会产生很多溢写的小文件，因此，就会需要归并（Merge）操作。

### 归并（Merge）

待所有数据都溢写完毕后，会对任务产生的所有中间数据文件做一次合并操作，以确保一个Map Task最终只生成一个中间数据文件（准确的说是小于3个吧，因为若溢写文件过少，是不会进行归并操作的），在这个过程中，<k,v>会变归并成<k,v list>。

归并后的数据是分区的，排序的。这里用的排序方法一般是**多路归并排序**。

归并可能发生很多次，因为**每次归并的文件数默认为10**，另外如果用户定义了Combine，这个阶段还需要进行**Combine**。

上述完成后，Map端的Shuffle就算结束了，下面用一张图回顾一下。

<img src="http://pic.xcq5120.xyz/shuffle-mapShuffle.png" style="zoom: 50%;" />



## Reduce端Shuffle

### 拉取 （Copy）

Reduce的输入是Reduce主动去磁盘上拉取的。Reduce进程启动Fetcher线程，通过**HTTP**方式请求Map所在的TaskTracker获取Map的输出文件。

这些Copy的数据会首先保存的内存缓冲区中，当内存缓冲区的使用率达到一定**阀值**后，则写到磁盘上。

### 归并（Merge）

Copy过来的数据会先放入内存缓冲区中，这里的缓冲区大小要比 map 端的更为灵活，它基于 JVM 的heap size设置，因为 Shuffle 阶段 Reducer 不运行，所以应该把绝大部分的内存都给 Shuffle 用。

这里需要强调的是，**Merge 有三种形式**：1)内存到内存 2)内存到磁盘 3)磁盘到磁盘。

默认情况下第一种形式是不启用的。

当内存中的数据量到达一定阈值，就启动内存到磁盘的 Merge，之所以进行Merge是因为Reduce端在从多个Map端Copy数据的时候，并没有进行Sort，只是把它们加载到内存，当达到阈值写入磁盘时，需要进行Merge 。这和Map端的很类似，这实际上就是溢写的过程，在这个过程中**如果设置有Combiner，它也是会启用的**，然后在磁盘中生成了众多的溢写文件，这种Merge方式一直在运行，直到没有Map端的数据时才结束。

然后才会启动第三种磁盘到磁盘的Merge，生成最终的那个文件。这个过程和Map端的差不多，但这个时候**不会发生Combine。**

### 结束

最终的<key，value list>，将会交给Reduce，按照业务逻辑进行处理，下面用一张图回顾一下Reduce端的Shuffle过程。
![](http://pic.xcq5120.xyz/shuffle-reduceShuffle.png)

## 再看Shuffle

Shuffle过程比较繁琐复杂，再用一张图总体表示一下。

<img src="http://pic.xcq5120.xyz/shuffle.png" alt="Shuffle"  />

## Shuffle调优

- 调大环形缓冲区的大小和溢写比例，减小IO压力
- 调大默认归并因子，使得每一次归并可以处理的文件更多，减少Merge发生次数
- 如果map输出的数据量非常大，那么在写入磁盘时压缩数据往往是个很好的主意，因为这样会让写磁盘的速度更快，节约磁盘空间，并且减少传给reducer的数据量
- 待以后实战补充

# WordCount实例

`WcMapper.java`

``` java
package xyz.xcq5120.wordcount;

import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;

import java.io.IOException;


public class WcMapper extends Mapper<LongWritable, Text, Text, IntWritable> {

  private Text word = new Text();
  private IntWritable one = new IntWritable(1);

  @Override
  protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
    // 拿到这一行数据
    String line = value.toString();

    // 按照空格切分数据
    String[] words = line.split(" ");

    // 遍历数组,把单词变成（word, 1）的形式交给框架
    for (String word : words) {
      this.word.set(word);
      context.write(this.word, this.one);
    }
  }
}
```

`WcReducer.java`

``` java
package xyz.xcq5120.wordcount;

import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;

import java.io.IOException;

public class WcReducer extends Reducer<Text, IntWritable, Text, IntWritable> {

  private IntWritable total = new IntWritable();

  @Override
  protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
    // 遍历并累加
    int sum = 0;
    for (IntWritable value : values) {
      sum += value.get();
    }

    // 打包后交给框架
    total.set(sum);
    context.write(key, this.total);
  }
}
```

`WcDriver.java`

``` java
package xyz.xcq5120.wordcount;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

import java.io.IOException;

public class WcDriver {
  public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
    // 1.获取一个Job实例
    Job job = Job.getInstance(new Configuration());

    // 2.设置类路径（Classpath）
    job.setJarByClass(WcDriver.class);

    // 3.设置Mapper和Reducer
    job.setMapperClass(WcMapper.class);
    job.setReducerClass(WcReducer.class);

    // 4.设置Mapper和Reducer的输出类型
    job.setMapOutputKeyClass(Text.class);
    job.setMapOutputValueClass(IntWritable.class);

    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);

    // 5.设置输入输出数据
    FileInputFormat.setInputPaths(job, new Path(args[0]));
    FileOutputFormat.setOutputPath(job, new Path(args[1]));

    // 6.提交Job
    boolean b = job.waitForCompletion(true);
    System.exit(b ? 0 : 1);

  }
}
```

