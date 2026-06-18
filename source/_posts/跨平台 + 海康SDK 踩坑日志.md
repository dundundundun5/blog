---
title: 跨平台 + 海康SDK 踩坑日志
date: 2026-06-18 11:13:02
categories:
 - Java
 - 开发
 - 设备网络SDK
---
## 前言
- 此处的海康sdk是使用[海康开放平台](https://open.hikvision.com/#home)的设备网络sdk对网络摄像机、网络录像机、超脑等设备进行二次开发

- 海康明确支持
  - C++ 把cmake写好，windows调用dll，linux调用so，再构建个十几次调一调
  - Java 最稳，要用官方demo的jna.jar和example.jar做外部调用。要指定路径，指定dll或者so文件
  - C# dllimport最舒服的一集，那么代价是so怎么import，跨平台out
  - Python 像java那样读取dll或者so
- 设备网络sdk在2021年后，也就是6.9.x版本都是稳定版，没有什么功能大改，新老版本兼容性非常好
## 海康sdk到底有哪些
- 海康旗下有非常多的子产品
  - HikVision主要是监控摄像头，官方软件是`设备网络搜索`和`IVMS-4200`
  - HikMicro和HikRobot主要是工业相机，官方软件是`MVS`
- HikVision的SDK，列举一下可能用到的
  - `设备网络sdk` - 跨平台 最需要的各种监控摄像头生态相关的二次开发
  - `播放库sdk` - 可以和设备`网络sdk`配合，用于视频解码推流、解码播放
  - `websdk插件版` - 纯windows，是H5的静态js库，插件是一个exe的推流后台服务
  - `websdk无插件版` - 虽然没有说跨不跨平台，但H5的静态js库+一个自制的exe推流后台服务，你有exe那肯定限windows了
  - `视频web插件` - 纯windows解决老三样
  - `H5视频播放器` - - 虽然没有说跨不跨平台，但H5的静态播放器控件+一个自制的exe推流后台服务，你有exe那肯定限windows了
  - `视频SDK` - 跨平台，但是跨windows、ios、鸿蒙、安卓。还是解决实时预览、录像回放、语音对讲
- 其他子牌的sdk就不列举了  
## 如果要用麒麟OS V10 SP1 2403 X86_64 桌面版
- 如果要用语音对讲功能，一定要用6.9.X版本的sdk。2025年的6.11.x的sdk会找不到音频驱动，而2021年的6.9.x版本的sdk就行，因为这个版本的麒麟是2020年底的内核定制版，音频驱动都是老的很。

## 如果要做web后端
- C# 跨平台不行，而且如果接信创有成为红色组件的可能，pass
- Java SpringBoot最稳的选择，但是官方demo功能不全，需要自己查明sdk里面到底有哪些方法，然后往HcNetSDK.java里面加方法
- Python 算了吧，解释型语言有一万个坑
- C++ 说实话不是很熟悉
## 如果要做跨平台桌面端
桌面端最绕不开的就是实时预览、录像回放

每一个都有坑，接下来细品
## 要做跨平台桌面端的实时预览和录像回放
- 如果不做跨平台的，那就真是条条条条大路通罗马了，立刻关闭这篇文章
- 实时预览通常除了画面本体，海康还会附带智能信息。也就是事件里的规则框，图例里有隐私信息，因此截取一半
    
    ![](blue.png)
### 要做不带智能信息的实时预览和录像回放
如果不要求带智能信息，那就是最大的福利，基本上是条条大路通罗马，简单罗列几条  
1. JavaScript - electron
   1. Canvas/Video/Div + ffmpeg转码推流 + rtsp地址
   2. Canvas/Video/Div + ffmpeg转码 + webrtc推流 + rtsp地址
   3. VlcVideo控件 + vlc播放器 + rtsp地址 - node有vlc支持的依赖 缺点是要自带一个200多MB的linux或者windows的vlc包
2. C++ - Qt
   1. QWidget + 设备网络sdk 渲染句柄方式/(回调方式+播放库解码) - 我只能说C++是海康二次开发钦定的开发语言
3. C# - Avalonia
   1. vlc播放器 + vlc nuget依赖 + rtsp地址
4. 前后端分离式  - electron + SpringBoot后端
   1. SpringBoot + 设备网络sdk rtsp地址 ffmpeg转码成mpeg推流到websocket，前端放一个mpeg播放器即可
### 要做带智能信息的实时预览和录像回放
如果要带智能信息，只能说目前处处碰壁，AI处处胡说八道，就连海康售后支持人员也一知半解

1. C++ - Qt  
   1. QWidget + 设备网络sdk 渲染句柄方式 - 还得是海康钦定的开发语言

### 要做能倍速播放的录像回放
录像回放要支持倍速播放，2倍速~16倍速，
1. C++ - Qt
   1. 是的还是C++，不要和sdk为敌，让你用你就用
2. 前后端分离式  - electron + SpringBoot后端
   1. 设备网络sdk + 回调方式 + 播放库解码 （文件模式）+ 播放库播放控制 + ffmpeg推流 websocket
   