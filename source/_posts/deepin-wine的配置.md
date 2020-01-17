---
title: deepin-wine的配置
date: 2020-01-17 19:17:51
categories: Ubuntu
tags: Ubuntu
---
# 前言
`deepin-wine`是深度公司为了使`Ubuntu/ArchLinux`系用户能使用最稳定qq等`win`上软件而努力开发的环境，关于如何使用该环境和对应的`deepin.com`容器（就是各类应用），可以前往它的[github主页](https://github.com/wszqkzqk/deepin-wine-ubuntu)学习。本文主要记录自己在使用该环境遇到的问题和解决方案。
<!--more-->



# 托盘问题

`Ubuntu18.04`下，qq，微信等软件不会自动显示在顶栏上，但安装`Gnome Shell`插件：[TopIcons Plus](https://extensions.gnome.org/extension/1031/topicons/)，问题便可迎刃而解。




# dpi问题

高分屏下，qq，微信等软件的字体可能会很小。解决该问题可以参照`github`上给出的解决方案：

>因为wine对HiDPI不会默认适配dpi值。解决方案:
>
>```
>WINEPREFIX=~/.deepinwine/你的wine容器目录  /usr/bin/deepin-wine  winecfg
>```
>
>注意`WINEPREFIX`这个环境变量指向你的deepin wine容器目录，比如TIM在`~/.deepinwine/Deepin-TIM`，微信在`~/.deepinwine/Deepin-WeChat`，用`tab`补全一下这个目录即可。
>
>打开wine设置页面，在`显示`选项卡中调整`屏幕分辨率`的dpi值即可。比如想实现win 10的150% DPI只需要将`96`改到`144`即可，125%放大则对应`120`。手工调整下合适的DPI就可以了




# 卸载问题

关于卸载整个环境，开发者wszqkzqk同学已提供`deepin-wine`环境的[uninstall.sh](https://github.com/wszqkzqk/deepin-wine-ubuntu/blob/master/uninstall.sh)脚本。至于应用容器，可以用`sudo apt remove 软件包主名`命令来删除，比如`deepin.com.qq.office_2.0.0deepin4_i386.deb`的卸载命令是`sudo apt remove deepin.com.qq.office`



# 修改内置浏览器

有时候点击qq的邮箱或者空间，会使用`wine`内置的`IE浏览器`打开，非常辣眼睛，修改它！

先找到IE所在的目录，我的是`~/.deepinwine/Deepin-QQ/drive_c/Program Files/Internet Explorer`，里面的那个`iexplore.exe`就是浏览器

然后只需要用`vim`打开它，将所有东西删掉，然后加上：
> ``` bash
> #!/bin/bash
> # Allow users to override command-line options
> if [[ -f ~/.config/chrome-flags.conf ]]; then
>  CHROME_USER_FLAGS="$(cat ~/.config/chrome-flags.conf)"
> fi
> # Launch
> exec /opt/google/chrome/google-chrome $CHROME_USER_FLAGS "$@"
>```

这样当qq调用`iexplore.exe`的时候就会找到这个`bash`脚本，从而调用`Ubuntu`上的`google-chrome`了。

