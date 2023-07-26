---
title: day01-WebDriverAgent安装
date: 2023-7-27 16:31:29
tags: 
	- 自动化测试
categories: 
	- 自动化测试

---



## 一、iOS真机安装WebDriverAgent图文详解

### 背景

在做iOS自动化测试的时候，一般都需要确保手机上已经安装有WebDriverAgent应用，这个WDA应用可以是Airtest修改版、Appium修改版也可以是Facebook原版，今天我们以Appium修改版为例来进行说明，其他版本同样适用。



安装依赖：‍

```
pip3 install -U tidevice
```

拉取代码：

```
git clone https://github.com/appium/WebDriverAgent
```



### 证书设置

1、进入WebDriverAgent项目根目录，双击打开WebDriverAgent.xcodeproj，然后在Xcode中的TARGETS里选中WebDriverAgentLib，按照下图数字序号依次点击，注意步骤4要开启自动管理签名。

![图片](day01-iOS真机搭建自动化测试环境/9312e2472c604fbeba9ea7070866501e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)



2、证书配置好之后，分别选择WebDriverAgentRunner和目标设备，<font color="red">**以Test方式运行，切记Test方式**</font>

![图片](day01-iOS真机搭建自动化测试环境/91381d4c490e4f789450fdcbf92f42c4~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)



3、运行Test，就可以在Xcode控制台看到下面的输出信息：

![图片](day01-iOS真机搭建自动化测试环境/2e6f38f1338d4e6f8b1ebd750e328aa5~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)



通过上面给出的IP和端口，加上/status合成一个url地址，例如[http://10.0.0.1:8100/status，然后浏览器打开，如果出现下图的输出，就说明WDA安装成功了。](https://link.juejin.cn/?target=http%3A%2F%2F10.0.0.1%3A8100%2Fstatus%EF%BC%8C%E7%84%B6%E5%90%8E%E6%B5%8F%E8%A7%88%E5%99%A8%E6%89%93%E5%BC%80%EF%BC%8C%E5%A6%82%E6%9E%9C%E5%87%BA%E7%8E%B0%E4%B8%8B%E5%9B%BE%E7%9A%84%E8%BE%93%E5%87%BA%EF%BC%8C%E5%B0%B1%E8%AF%B4%E6%98%8EWDA%E5%AE%89%E8%A3%85%E6%88%90%E5%8A%9F%E4%BA%86%E3%80%82)



![图片](day01-iOS真机搭建自动化测试环境/861b028a43784597a8e68a1194a6293f~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)



<font color="red">但是有些国产的iPhone机器通过手机的IP和端口还不能访问，此时需要将手机的端口转发到Mac上，这个时候执行下面的命令即可</font>：

```
tidevice relay 8100 8100
```



参考文章：

https://juejin.cn/post/6993892294705283080

