---
title: Android编译so库
date: 2017-11-25 15:10:30
tags:
---
Android开发中很多时候需要使用第三方开源库，当它是用C++写的时就需要编译成SO库，本文就是编译之前需要做的准备！
配置NDK、SDK环境！
配置NDK、SDK环境！
配置NDK、SDK环境！


### 搭建虚拟机
我在这里选择VMware12虚拟机VMware Workstation Pro[下载地址](https://www.vmware.com/cn.html)

### 安装Linux系统
选择Ubuntu[下载地址](http://www.ubuntu.org.cn/download/desktop)

### 下载AndroidNDK及SDK
android NDK选择Linux版本。[下载地址](https://developer.android.google.cn/ndk/downloads/index.html)
android SDK选择高一点的Linux版本。[下载地址](http://tools.android-studio.org/index.php/sdk/)
> 下载完成后在目录下会看到我们下载的ndk和sdk压缩包我们把它们解压出来，一个是.zip的另一个是.tgz的。

```
unzip xxx.zip
tar zxvf xxx.taz
```
> 将两个压缩文件解压到当前目录即可。

### 下载openjdk
```
sudo apt-get install openjdk-8-jre-headless
```
> 会自动安装

### 配置SDK和NDK全局环境变量
下载的linux版本的SDK缺少一点东西，需要运行命令
```
sh /android-sdk-linux/tools/android
```
下载最新的Android SDK Tools、Android SDK Platform-tools、Android SDK Build-tools、Android SDK Platform即可。
```
sudo gedit /etc/profile
```

在文件最后加上
```
export PATH=/你的路径/android-sdk-linux/platform-tools:$PATH
export PATH=/你的路径/android-sdk-linux/tools:$PATH
export ANDROID_NDK=/你的路径/android-ndk-r14b
export PATH=/你的路径/android-ndk-r14b:$PATH
```
> 路径可能会有所不同！

-------------
**重启系统**既可以生效

**接下来就可以编译你的ANdroid SO库了!!!**