---
title: 双系统安装Nvidia驱动
date: 2020-01-17 18:41:50
categories: [Ubuntu]
tags: [Ubuntu]
---
# 前言
在安装`Ubuntu`系统时，英伟达显卡会由于驱动不兼容会导致各种各样的系统问题，网上的资料参差不齐，本文记录自己在帮助室友安装系统后总结的方法，终结此问题。

<!--more-->

网上提供的显卡驱动安装方法基本上可归纳为三类：

* `ppa`源安装
* 官网驱动`.run`文件安装
* 标准`Ubuntu`仓库安装

事实上三种方法都可以成功，但这都是理论上的，而且充满玄学，有的人一条命令就能成功，有的人却会踩上各种各样奇奇怪怪的坑，什么认证失败，什么无法关机，什么clean 甚至黑屏......下面记录一条可以通用解决此类问题的安装方法，请确保智商上线，**每一步都不跳过**。另外，本文仅针对双系统或`Ubuntu`单系统，版本为`Ubuntu18.04`。


# 关闭安全模式

这一步最好在设置U盘启动时就去做掉，详细的说就是**`进入BIOS把UEFI的Secure Boot选项关闭**`，这样做的原因大体可以描述为：

>在支持`UEFI`的设备上打开`Secure Boot`后，`Ubuntu`对于添加到内核的模块更加保守，需要持有签名才能添加，而显卡驱动是要写进内核的，在安装过程中我们可以看见提示是否生成签名。如果生成成功则没问题，若是没有生成，则会出现各种问题，循环登录的问题可能与此有关。


# 安装时禁用集显

在`grub`安装界面，用`e`选择`Install Ubuntu`选项，之后会进入命令模式。然后在`quiet slash --`后面隔一个空格（`--`是否存在因人而异）添加`acpi_osi=linux nomodeset `。添加完后按`F10`保存并引导。

网上有部分教程给出添加`acpi=off`[^1]的参数，本质上都是为了禁用集显，但`acpi=off`可能导致[无法正常关机](https://askubuntu.com/questions/139157/booting-ubuntu-with-acpi-off-grub-parameter)的现象，因此并不推荐，请使用`acpi_osi=linux nomodeset`。


# 驱动安装

驱动安装有三种方法，应该都可以成功，我这里推荐两种方法：

* 使用标准仓库安装
  ​1.成功安装并重启后，打开终端

  2.输入`sudo apt update`

  3.进入自带GUI工具`软件和更新`

  4.在`附加驱动`中安装`Nvidia的专有闭源驱动`并等待其完成

  5.打开终端，输入`nvidia-smi`，若出现GPU信息，则表明安装成功，否则请重头来过

* 修改163源后安装
  1.成功安装并重启后，打开终端

  2.输入`sudo gedit /etc/apt/source.list`

  3.**注释掉原有的内容**，添加163源，网上给的163源内容有很大差异，效果也不尽相同，为确保效率，请使用以下的内容

  ```shell
  deb http://mirrors.163.com/ubuntu/ bionic main restricted universe multiverse
  deb http://mirrors.163.com/ubuntu/ bionic-security main restricted universe multiverse
  deb http://mirrors.163.com/ubuntu/ bionic-updates main restricted universe multiverse
  deb http://mirrors.163.com/ubuntu/ bionic-proposed main restricted universe multiverse
  deb http://mirrors.163.com/ubuntu/ bionic-backports main restricted universe multiverse
  deb-src http://mirrors.163.com/ubuntu/ bionic main restricted universe multiverse
  deb-src http://mirrors.163.com/ubuntu/ bionic-security main restricted universe multiverse
  deb-src http://mirrors.163.com/ubuntu/ bionic-updates main restricted universe multiverse
  deb-src http://mirrors.163.com/ubuntu/ bionic-proposed main restricted universe multiverse
  deb-src http://mirrors.163.com/ubuntu/ bionic-backports main restricted universe multiverse
  ```

  4.`sudo apt update`

  5.按第一种方法第三步往下走

是的，两种方法基本一样，区别在源的问题，这是为了解决网速问题。另外，无论选择了哪种方法，请**不要**中途停止后换方法，请硬着头皮等到底。

`ppa源`和`.run文件`还有`autoinstall`也能成功安装驱动，前提是你能解决依赖问题，否则请使用自带GUI工具。

只要输入`nvidia-smi`后，若出现GPU信息，则这一步骤成功


# 参数再修改

安装完驱动后，请先修改参数再重启。

​	1.打开终端`sudo gedit /etc/default/grub`

​	2.在`GRUB_CMDLINE_LINUX_DEFAULT`那一行，我们会发现`quiet splash nomodeset`，这时要删除`nomodeset`，然后加上`acpi_osi=linux`，也就是说，这一行最终要改成`GRUB_CMDLINE_LINUX_DEFAULT=“quiet splash acpi_osi=linux”`

​	3.`sudo update-grub`来更新配置更改

​	4.重新启动

解释一下为什么要这样修改参数：

> Grub引导了系统进行启动，所以它的参数被传入了，即nomodeset（调用集显）如果存在，系统就会一直调用集显，然后就出现循环登录或黑屏。由于刚刚安装系统一般没有驱动，很多人只能通过调用集显去进入图形界面（除非在命令行下安装了驱动），导致了nomodeset参数的加入。
>而acpi_osi=linux是告诉Grub，电脑将以Linux系统启动，调用其中驱动，所以可以用Nvidia的驱动进行显示了！



[^1]:acpi,**Advanced Configuration and Power Interface**

