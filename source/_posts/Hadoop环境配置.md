---
title: Hadoop环境配置
date: 2020-01-10 11:53:38
categories: hadoop
tags: hadoop
---

# 前言
本文配置hadoop的**单机模式**和**伪分布式模式**
hadoop依赖于Java环境，本文默认Java环境配置完毕。

<!--more-->

# 下载安装

## 下载
下载地址： <http://hadoop.apache.org/releases.html> 
我选用时3.1.2版本，点击binary进入下载页面。 binary是已经编译好的程序，解压即可使用。

## 解压

在`/opt`目录下创建一个文件夹，并将下载后的文件解压至此

``` bash
sudo mkdir /opt/modules
sudo tar -zxvf hadoop-3.1.2.tar.gz -C /opt/modules
```

这里多说两句`/opt`。opt有可选的意思，`/opt`目录用来安装附加软件包，是用户级的程序目录，安装到/opt目录下的程序，它所有的数据、库文件等等都是放在同个目录下面。不需要它时，可以直接使用`rm -rf`删除，硬盘不够也可以将该目录挂在在其它硬盘。

# 单机模式搭建

## 配置环境变量

配置hadoop环境就是配置`hadoop-env.sh`，命令如下:

``` bash
cd /opt/modules/hadoop-3.1.2/etc/hadoop
vim hadoop-env.sh
# 可以在相应的注释部分添加以下内容，也可以直接添加在文件任何地方
# java和hadoop的具体位置应以实际为准
export JAVA_HOME=/usr/lib/jdk1.8.0_211
export HADOOP_HOME=/opt/modules/hadoop-3.1.2
```

内容如下：

![hadoop-1](http://pic.xcq5120.xyz/hadoop/1.png)

配置完成后，可以使用如下命令检验：

```  bash
cd /opt/modules/hadoop-3.1.2
bin/hadoop version
# 能正常返回版本信息则安装完成
```

## 配置bin目录至系统环境变量

由上述命令可以看出，想要执行hadoop命令还是有一点麻烦的（需要进入hadoop的安装目录），因此可以将`bin`目录配置到shell变量里，使得命令可以在任何地方运行：

``` bash
cd ~
# 我使用的是zsh，bash修改的是.bashrc
vim .zshrc
# 添加hadoop路径，并修改PATH变量
export HADOOP_HOME=/opt/modules/hadoop-3.1.2
export PATH=........${HADOOP_HOME}/bin
# 保存退出后刷新配置
source .zshrc
# 这时就可以在任意位置使用hadoop命令了
cd ~
hadoop version
```

具体内容为：

![hadoop-2](http://pic.xcq5120.xyz/hadoop/2.png)

这里再多说一句，就是要注意`PATH`和其余变量的位置，如果`PATH`放置在第一行，`source`后可以正常使用，但重启系统后变量就会失效。

至此。hadoop的本地模式就配置完成了，可以使用自带的demo测试一下。

## 测试本地模式

### wordcount

准备工作:

1.创建input文件夹作为要测试的输入文件。 
2.将hadoop目录里的etc/hadoop目录下的所有.xml结尾的文件复制到input里 

wordcount是自带的一个demo，该例子是搜索input文件夹内所有文件，统计所有单词出现的次数，并输出在output/wordcount文件夹里。 

具体使用方法为：

``` bash
hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.1.2.jar wordcount input output/wordcount
```

结果可以使用`cat`查看：

``` bash
cat output/wordcount/part-r-00000
```

至此，hadoop的本地模式配置完成。




# 伪分布式搭建

## 安装SSH、配置SSH无密码登陆

集群、伪分布模式都需要用到 SSH 登陆，Ubuntu 默认已安装了 SSH client，此外还需要安装 SSH server：

``` bash
sudo aptitude install openssh-server
## 笔者的电脑安装会出现依赖冲突，解决办法为卸载openssh-client后重新安装
```

安装完成后可以使用如下命令查看：

``` bash
ps -e|grep ssh
# 正常安装后会有两个服务：sshd和ssh-agent
# sshd是服务端进程；ssh-agent是客户端进程
```

检查正常后，可以使用如下命令登陆本机：

``` bash
ssh localhost
```

一般情况下，上一个命令登录需要密码，我们需要将其配置成**免密码模式**

``` bash
ssh-keygen -t rsa              			# 生成一个秘钥。会有提示，都按回车就可以
cat ./id_rsa.pub >> ./authorized_keys   # 加入授权
chmod 644 authorized_keys 				# 设置文件权限
chmod 700 ~/.ssh						# 设置目录权限
# 到此为止，需求应该可以解决，但笔者的电脑仍然无法实现免密登录，查阅大量资料后发现再执行如下命令就行了。
# （笔者怀疑这命令和第二条命令的目的是一样的，但不确定，有空时去学习学习
ssh-copy-id localhost					
```

## 伪分布环境配置

Hadoop 可以在单节点上以伪分布式的方式运行，Hadoop 进程以分离的 Java 进程来运行，**节点既作为 NameNode 也作为 DataNode**，同时，读取的是 HDFS 中的文件。

Hadoop 的配置文件位于 .../hadoop/etc/hadoop/ 中，伪分布式需要修改2个配置文件 `core-site.xml` 和 `hdfs-site.xml` 。Hadoop的配置文件是 xml 格式，每个配置以声明 property 的 name 和 value 的方式来实现。

``` bash
# 修改配置文件 core-site.xml
cd /opt/modules/hadoop-3.1.2
vim etc/hadoop/core-site.xml
```

将其中的```<configuration></configuration>```配置成：

``` xml
<configuration>
    <property>
        <name>hadoop.tmp.dir</name>
            <value>file:/opt/modules/hadoop-3.1.2/tmp</value>
            <description>Abase for other temporary directories.</description>
    </property>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
```

同样的，修改配置文件 `hdfs-site.xml`：

``` xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:/opt/modules/hadoop-3.1.2/tmp/dfs/name</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:/opt/modules/hadoop-3.1.2/tmp/dfs/data</value>
    </property>
</configuration>
```

**注**：

>Hadoop 的运行方式是由配置文件决定的（运行 Hadoop 时会读取配置文件），因此如果需要从伪分布式模式切换回非分布式模式，需要**删除** core-site.xml 中的配置项。
>
>此外，伪分布式虽然只需要配置 fs.defaultFS 和 dfs.replication 就可以运行（官方教程如此），不过若没有配置 hadoop.tmp.dir 参数，则默认使用的临时目录为 /tmp/hadoo-hadoop，而这个目录在重启时有可能被系统清理掉，导致必须重新执行 format 才行。所以我们进行了设置，同时也指定 dfs.namenode.name.dir 和 dfs.datanode.data.dir，否则在接下来的步骤中可能会出错。

配置完成后，使用如下命令进行NameNode的格式化：

``` bash
./bin/hdfs namenode -format
# 如果配置过环境变量，可以使用：
hdfs namenode -format
```

正常情况下，会出现如下结果：

![hadoop-3](http://pic.xcq5120.xyz/hadoop/3.png)

格式化完成后，就可以启动服务了：

``` bash
cd /opt/modules/hadoop-3.1.2
./sbin/start-dfs.sh
# 可以将sbin目录配置进环境变量，简化启动命令
```

启动完成后，可以通过命令 `jps` （这是用于 JVM 进程的 `ps` 实用程序）来判断是否成功启动，若成功启动则会列出如下进程: `NameNode`、`DataNode` 和 `SecondaryNameNode`

![hadoop-4](http://pic.xcq5120.xyz/hadoop/4.png)

成功启动后，可以访问 Web 界面 [http://localhost:9870](http://localhost:9870/) 查看 NameNode 和 Datanode 信息，还可以在线查看 HDFS 中的文件。

至此，hadoop的伪分布式搭建完毕，下面使用一个小demo来测试

## 测试伪分布式

在伪分布式环境下，系统读取的是hdfs文件系统数据，为了使用hdfs，首先需要创建用户目录。

``` bash
start-dfs.sh 								 # 开启服务
cd /opt/modules/hadoop-3.1.2
./bin/hdfs dfs -mkdir -p /user/xcq
./bin/hdfs dfs -mkdir input
./bin/hdfs dfs -put ./etc/hadoop/*.xml input # 将xml文件复制到input
./bin/hdfs dfs -ls input					 # 查看目录内的文件列表
```

在执行上述命令时，我们使用的是 xcq 用户，因此创建的相应用户目录为` /user/xcq` ，因此在命令中就可以使用相对路径如 input，其对应的绝对路径就是 `/user/xcq/input`

准备工作好了后，就可以使用demo测试了：


``` bash
cd /opt/modules/hadoop-3.1.2
hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.1.2.jar grep input output 'dfs[a-z.]+'
hdfs dfs -get output ./output   # 将 hdfs 上的 output 文件夹拷贝到本机
hdfs dfs -rm -r output    		# 删除 output 文件夹
./sbin/stop-dfs.sh				# 关闭服务
```

这里需要注意的是，Hadoop 运行程序时，输出目录不能存在，否则会提示错误 `org.apache.hadoop.mapred.FileAlreadyExistsException: Output directory hdfs://localhost:9000/user/hadoop/output already exists`，因此若要正常执行，需要删除 output 文件夹。

输出结果已经拷贝至系统文件夹。

至此，hadoop的伪分布式部署和测试也已完成。

