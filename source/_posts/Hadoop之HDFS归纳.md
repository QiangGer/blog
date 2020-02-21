---
title: Hadoop之HDFS归纳
date: 2020-02-21 12:00:38
categories: hadoop
tags: hadoop
---

# 前言

> Hadoop1.x可简单分为HDFS和MapReduce
> 本文归纳了自己在学习过程中有关HDFS的知识

<!--more-->

# 分布式文件系统

HDFS（**H**adoop **D**istributed **F**ile **S**ystem）,它是为以流的方式存取大文件而设计的。优缺点明显：

 **优点：**

   1）适合存储非常大的文件

   2）适合流式数据读取，即适合“**只写一次，读多次**”的数据处理模式

   3）适合部署在廉价的机器上

**缺点：**

   1）不适合存储大量的小文件，因为受Namenode内存大小限制

   2）不适合实时数据读取，高吞吐量和实时性是相悖的，HDFS选择前者

   3）不支持多用户写入和任意文件修改，不适合需要经常修改数据的场景

#相关概念

## 块（block）

HDFS的块和普通文件系统的块设计目的一样，就是为了分摊磁盘读写开销，但是HDFS的一个普通块默认大小是64MB。

块的大小可以调整，但需要考虑两个方面：

1.块设计的越大，越可以降低系统的寻址开销，提高检索效率

2.块的设计需要考虑mapreduce，因为mapreduce在处理数据时是以块为单位处理数据，块过大会降低MR任务的并行度，发挥不了分布式并行处理的优势。

## 名称节点（NameNode）

在HDFS中，名称节点（NameNode）负责管理分布式文件系统的命名空间（Namespace），保存了两个核心的数据结构，即**FsImage**和**EditLog**

### Fsimage

FsImage文件包含文件系统中所有目录和文件inode的序列化形式。每个inode是一个文件或目录的元数据的内部表示，并包含此类信息：
		对于目录，存储修改时间、权限和配额元数据（目录所属用户，所在组等）。
		对于文件，包含文件的块信息，复制等级、修改和访问时间、访问权限等。

FsImage文件没有记录文件包含哪些块以及每个块存储在哪个数据节点。而是由名称节点把这些映射信息保留在内存中，当数据节点加入HDFS集群时，数据节点会把自己所包含的块列表告知给名称节点，此后会定期执行这种告知操作，以确保名称节点的块映射是最新的。

更多关于Fsimage的信息可参照[此处](https://www.cnblogs.com/miner007/p/3745332.html)

### Editlog

客户端对hdfs所有的更新操作，比如说移动数据，或者删除数据，都会记录在editlog中。

### 名称节点的启动

在名称节点启动的时候，它会将FsImage文件中的内容加载到内存中，之后再执行EditLog文件中的各项操作，使得内存中的元数据和实际的同步，存在内存中的元数据支持客户端的读操作。

一旦在内存中成功建立文件系统元数据的映射，则创建一个新的FsImage文件和一个空的EditLog文件。

### 名称节点的运行

名称节点建立之后，HDFS中的更新操作会重新写到EditLog文件中，因为FsImage文件一般都很大（GB级别的很常见），如果所有的更新操作都往FsImage文件中添加，这样会导致系统运行的十分缓慢，但是，如果往EditLog文件里面写就不会这样，因为EditLog 要小很多。每次执行写操作之后，且在向客户端发送成功代码之前，edits文件都需要同步

### 第二名称节点

从上面的过程可以看见，名称节点的运行面临一个很大的问题：**editlog不断增大**

虽然这对名称节点运行时候是没有什么明显影响的，但是，当名称节点重启的时候，名称节点需要先将FsImage里面的所有内容映像到内存中，然后再一条一条地执行EditLog中的记录，当EditLog文件非常大的时候，会
**导致名称节点启动操作非常慢**，而在这段时间内HDFS系统处于安全模式，一直无法对外提供写操作，影响了用户的使用。

第二名称节点就是为了解决这个问题而设立的，另外，它还可以当做NameNode的冷备份。

### 第二名称节点的工作流程

1. **SecondaryNameNode**会定期和**NameNode**通信，请求其停止使用**EditLog**文件，暂时将新的写操作写到一个新的文件**edit.new**上来，这个操作是瞬间完成，上层写日志的函数完全感觉不到差别；
2. **SecondaryNameNode**通过**HTTP GET**方式从**NameNode**上获取到**FsImage**和**EditLog**文件，并下载到本地的相应目录下；
3. **SecondaryNameNode**将下载下来的**FsImage**载入到内存，然后一条一条地执行**EditLog**文件中的各项更新操作，使得内存中的**FsImage**保持最新；这个过程就是**EditLog**和**FsImage**文件合并；
4. **SecondaryNameNode**执行完操作3之后，会通过**post**方式将新的**FsImage**文件发送到**NameNode**节点上
5. **NameNode**将从**SecondaryNameNode**接收到的新的**FsImage**替换旧的**FsImage**文件，同时将**edit.new**替换**EditLog**文件，通过这个过程**EditLog**就变小了

## 数据节点 （DataNode）

数据节点是分布式文件系统HDFS的工作节点，负责数据的存储和读取，会根据客户端或者是名称节点的调度来进行数据的存储和检索，并且向名称节点定期发送自己所存储的块的列表。

每个数据节点中的数据会被保存在各自节点的本地Linux文件系统中。

# HDFS的数据存储和读写过程

## 数据存储

**冗余备份**没什么好说的，这里提一个名词叫**机架感知策略**，机架感知是需要去手动配置的，我还没配置过，将来有需要手动google一下就好，配置完成后的副本位置为：

•第一个副本：放置在上传文件的数据节点；如果是集群外提交，则选一台磁盘不太满、CPU不太忙的节点

•第二个副本：放置在与第一个副本不同的机架的节点上

•第三个副本：与第一个副本相同机架的其他节点上

•更多副本：随机

## HDFS的读取

``` java
public class Read {
  public static void main(String[] args) {
    try {
      Configuration conf = new Configuration();
      FileSystem fs = FileSystem.get(conf);
        //FileSystem是一个通用文件系统的抽象基类，可以被分布式文件系统继承，所有可能使用Hadoop文件系统的代码，都要使用这个类
        //Hadoop为FileSystem这个抽象类提供了多种具体实现
        //DistributedFileSystem就是FileSystem在HDFS文件系统中的具体实现
        
      Path path = new Path("hdfs://localhost:9000/user/hadoop/test.txt");
      FSDataInputStream is = fs.open(path);
        //FileSystem的open()方法返回的是一个输入流FSDataInputStream对象，在HDFS文件系统中，具体的输入流就是DFSInputStream
        //FSDataInputStream封装了DFSDataInputStream
        //DFSDataInputStream才是和NameNode沟通的主体
        //他们通过clientprotocal.getlocations来和NameNode建立RPC调用获取文件位置
      BufferedReader r = new BufferedReader(new InputStreamReader(is));

      String content = r.readLine();
      System.out.println(content);
        
      r.close();
      fs.close();
    } catch (Exception e) {
      e.printStackTrace();
    }
  }
}
```

根据代码和注释，我们可以将过程归纳为：

1. 客户端向远程的NameNode发起RPC请求，请求读取文件数据。
2. NameNode检查文件是否存在，如果存在则获取文件的元信息。
3. 客户端收到元信息后选取一个网络距离最近的DataNode，依次请求读取每个数据块。客户端首先要校检文件是否损坏，如果损坏，客户端会选取另外的DataNode请求。
4. datanode与客户端建立socket连接，传输对应的数据块，客户端收到数据缓存到本地，之后写入文件。
5. 依次传输剩下的数据块，直到整个文件合并完成。

![hdfs-read](http://pic.xcq5120.xyz/hdfs-read.png)

## HDFS的写

写和读有些类似，还是先看一个demo

``` java
public class Write {
  public static void main(String[] args) {
    try {
      Configuration conf = new Configuration();
      FileSystem fs = FileSystem.get(conf);
        
      Path file = new Path("hdfs://localhost:9000/user/hadoop/test.txt");
      FSDataOutputStream outStream = fs.create(file);
      // FileSystem中的create()方法返回的是一个输出流FSDataOutputStream对象，在HDFS文件系统中，具体的输出流就是DFSOutputStream。
      outStream.writeUTF("Welcome to HDFS Java API!!!");
        
      outStream.close();
      fs.close();
    } catch (Exception e) {
      e.printStackTrace();
    }
  }
}
```

过程可以归纳为：

1. 客户端向远程的NameNode发起RPC请求；
2. NameNode会检查要创建的文件是否已经存在，创建者是否有权限进行操作，成功则会为文件创建一个记录，否则会让客户端抛出异常；
3. 当客户端开始写入文件的时候，开发库会将文件切分成多个packets，packets被放入DFSOutputStream对象的内部队列，DFSOutputStream再向名称节点申请保存数据块的若干数据节点。
4. 这些数据节点形成一个数据流管道，队列中的分包最后被打包成数据包发往数据流管道中的第一个数据节点
   第一个数据节点将数据包发送到第二个节点，第二个发给第三个，依此类推，形成“流水线复制。
5. 为了保证节点数据准确，最后一个接收到数据的数据节点要向发送者发送“确认包”，确认包沿着数据流管道逆流而上，经过各个节点最终到达客户端，客户端收到应答时，它将对应的分包从内部队列移除。
6. DFSOutputStream调用ClientProtocal.complete()方法通知名称节点关闭文件，结束写入。

![hdfs-write](http://pic.xcq5120.xyz/hdfs-write.png)



