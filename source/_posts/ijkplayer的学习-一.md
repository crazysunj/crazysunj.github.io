---
title: ijkplayer的学习(一)
toc: true
date: 2017-05-13 12:38:38
tags: [FFmpeg,ijkplayer]
---

ijkplayer的github地址:[https://github.com/Bilibili/ijkplayer](https://github.com/Bilibili/ijkplayer)

## 简介

ijkplayer是一个基于FFmpeg的轻量级视频播放器。FFmpeg的是全球领先的多媒体框架，能够解码，编码， 转码，复用，解复用，流，过滤器和播放大部分的视频格式。它提供了录制、转换以及流化音视频的完整解决方案。它包含了非常先进的音频/视频编解码库libavcodec，为了保证高可移植性和编解码质量，libavcodec里很多code都是从头开发的

<!--  more-->

## 环境搭建（MAC环境）

### 安装前准备工作，安装homebrew, git, yasm

```
# install homebrew, git, yasm
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
brew install git
brew install yasm

# add these lines to your ~/.bash_profile or ~/.profile
# export ANDROID_SDK=<your sdk path>
# export ANDROID_NDK=<your ndk path>

# on Cygwin (unmaintained)
# install git, make, yasm
```
### clone代码

```
git clone https://github.com/Bilibili/ijkplayer.git ijkplayer-android
cd ijkplayer-android
git checkout -B latest k0.7.9
```

### 初始化android 

```
./init-android.sh
```

### 初始化android支持https（可选）

```
./init-android-openssl.sh
```

### 切到ijkplayer/android/contrib(编译脚本)

```
cd android/contrib
```

### 编译 openssl

支持https，与./init-android-openssl.sh同步

```
./compile-openssl.sh clean
./compile-openssl.sh all
```

### 修改编译配置

在ffmpeg编译前可以修改支持的编码格式，请切换到config目录，修改module.sh，或者让module.sh链接到module-default.sh（可以自己测试一下是否链接过去）

```
cd config
rm module.sh
ln -s module-default.sh module.sh
cd android/contrib
# cd ios
sh compile-ffmpeg.sh clean
```

### 开始编译 ffmpeg

```
./compile-ffmpeg.sh clean
./compile-ffmpeg.sh all
```

### 编译 ijkplayer native code

返回到 ijkplayer/android 目录

```
cd ..
```

### 编译生成各CPU架构的so

如果不加 all 默认只生成 armv7a 架构的 so

```
./compile-ijk.sh all
```

### 编译好的项目地址：

[https://github.com/crazysunj/ijkplayerDemo](https://github.com/crazysunj/ijkplayerDemo)

