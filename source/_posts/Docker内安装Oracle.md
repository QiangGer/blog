---
title: Docker内安装Oracle
date: 2020-01-17 18:11:06
categories: [ [Docker], [ Oracle] ]
tags: [Docker, Oracle]
---
# 前言
Oracle的安装较为复杂,本文讲述Oracle在Docker中的安装步骤

<!--more-->
# 下载

`docker`提供的`oracle`镜像有两种，一种是已经安装好的，另一种是配置好环境并提供脚本，由自己安装的，这里我们选择自己安装的类型。

``` bash
# 查找镜像
docker search oracle

# 拉取到本地
docker pull jaspeen/oracle-11g
```

**镜像下载完毕后，需要下载`oracle`的安装包**，这里建议在桌面环境下载后用`scp`等工具上传至服务器，因为`oracle`的下载链接都需要用户验证，在服务器上下载较麻烦。

安装包的下载地址为：[oracle11g](https://www.oracle.com/database/technologies/112010-linx8664soft.html)

# 安装

首先将安装包拷贝至服务器，并解压两个文件。

``` bash
# 拷贝至/opt目录
scp .../linux.x64_11gR2_database_* username@server:/opt

# 解压
unzip linux.x64_11gR2_database_1of2.zip
unzip linux.x64_11gR2_database_2of2.zip
```

解压完成后，会生成一个`database`文件夹。接下来要新建一个文件夹，将`database`文件夹置于此文件夹下，这一步是**不可或缺**的，不然安装脚本会找不到安装文件，并且`database`这个文件名也**不能**修改。

``` bash
# 新建一个文件夹，这个文件夹名字可以任意取
mkdir oracle_install_files

# 移动安装文件
mv database oracle_install_files
```

接下来就可以让启动容器并安装数据库了。

``` bash
docker run --privileged --name oracle11g -p 1521:1521 -v ~/opt/oracle_install_files:/install jaspeen/oracle-11g
```

安装过程若提示交换空间不足，可以参考附录。

# 使用 

等上面的安装过程结束，我们就可以使用数据库了。

上面的安装进程不会输出install successfully这样的提示，而是会在控制台打出进程开启的端口号等信息后就结束，这时候我们需要退出这个容器重新进入。（ctrl C ctrl D等方法似乎都不顶用，我直接退出了和服务器的连接终端...）

``` bash
# 进入容器
docker exec -it oracle11g /bin/bash

# 切换用户
su - oracle

# 登录
sqlplus / as sysdba 

# 解锁用户
SQL> alter user scott account unlock;
SQL> commit;
SQL> conn scott/tiger

# 退出
ctrl + D
```

这里`ctrl + D`不会导致容器停止，可以放心退出

# 连接

在桌面环境使用`Navicat`连接即可可视化操作

![1](http://pic.xuecq.cc/docker-aliyun.png)



# 附录

部分机器安装过程中，可能出现如下报错

`Checking swap space: 0 MB available, 150 MB required. Failed <<<<`

这是机器没有配置交换空间导致的。

解决办法： [参考链接](https://www.cnblogs.com/a9999/p/6957280.html)

1、检查` Swap` 空间在设置 `Swap` 文件之前，有必要先检查一下系统里有没有既存的` Swap `文件。运行以下命令：

``` bash
swapon -s
```

如果返回的信息概要是空的，则表示 `Swap `文件不存在。

2、检查文件系统在设置` Swap` 文件之前，同样有必要检查一下文件系统，看看是否有足够的硬盘空间来设置 `Swap` 。运行以下命令：

``` bash
df -hal
3、创建并允许 Swap 文件下面使用 dd 命令来创建 Swap 文件。检查返回的信息，还剩余足够的硬盘空间即可。
```

 3、创建并允许` Swap `文件下面使用` dd` 命令来创建` Swap` 文件。检查返回的信息，还剩余足够的硬盘空间即可。

``` bash
dd if=/dev/zero of=/swapfile bs=1024 count=512k
```

参数解读：if=文件名：输入文件名，缺省为标准输入。即指定源文件。< if=input file >of=文件名：输出文件名，缺省为标准输出。即指定目的文件。< of=output file >bs=bytes：同时设置读入/输出的块大小为bytes个字节count=blocks：仅拷贝blocks个块，块大小等于bs指定的字节数。

4、格式化并激活 `Swap `文件上面已经创建好` Swap `文件，还需要格式化后才能使用，运行命令

``` bash
mkswap /swapfile
```


激活 `Swap` ，运行命令

``` bash
swapon /swapfile
```

以上步骤做完，再次运行命令 

``` bash
swapon -s
```

你会发现返回的信息概要：

``` bash
Filename                Type        Size    Used    Priority
/swapfile               file        524284    0     -1
```

如果要机器重启的时候自动挂载 `Swap` ，那么还需要修改 `fstab` 配置。用` vim` 打开 `/etc/fstab` 文件，在其最后添加如下一行：

``` bash
 /swapfile          swap            swap    defaults        0 0
```

最后，赋予 `Swap` 文件适当的权限：

``` bash
chown root:root /swapfile 
chmod 0600 /swapfile
```


