---
title: Ubuntu18.04配置搜狗输入法
date: 2020-01-17 19:17:35
categories: Ubuntu
tags: Ubuntu
---
# 前言
`ibus`是`Ubuntu`自带的中文输入法，可是实际体验体验并不佳，反应速度慢，还会崩溃，最重要的是词库更新缓慢，经常找不到想要的字，经过折腾后，决定使用搜狗拼音。本文介绍具体使用方法：
<!--more-->
# 卸载
## 卸载ibus

``` bash
sudo apt remove ibus
```

## 清除ibus配置

``` bash
sudo apt purge ibus
```

## 卸载任务栏上的ibus键盘提示

``` bash
sudo apt remove indicator-keyboard
```
完成后，接来下要下载搜狗拼音依赖的[fcitx](https://github.com/fcitx)输入法框架。

# 安装
## 安装fcitx框架

``` bash
sudo apt install fcitx-table-wbpy fcitx-config-gtk
```

## 切换fcitx为默认输入法

``` bash
im-config -n fcitx
```

## 重启系统来更新配置

``` bash
sync sync reboot
```

## 下载输入法安装包并安装

前往[官网](https://pinyin.sogou.com/linux/?r=pinyin)下载最新的安装包，手动安装。


## 修复损坏缺少的包

```bash
sudo apt install -f
```
# 个性化
接下来如果在任务栏上依然存在`ibus`的输入法提示，可以使用[`Icon hider`](https://extensions.gnome.org/extension/351/icon-hider/)这个拓展插件来移除，这个插件同样可以隐藏其它的任务栏图标。
最后可以使用搜狗拼音的配置来个性化定制输入法
Enjoy.





​	