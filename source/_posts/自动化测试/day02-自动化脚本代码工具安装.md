---
title: day02-自动化脚本代码工具安装
date: 2023-7-27 17:31:29
tags: 
	- 自动化测试
categories: 
	- 自动化测试

---



## 一、安装自动化工具Airtest

Airtest是网易开源的一个自动化测试工具，支持iOS、安卓、Windows、Mac

去Airtest官网下载：http://airtest.netease.com/





# 二、运行方式

参考官方的介绍：https://airtest.readthedocs.io/zh_CN/latest/README_MORE.html



## 三、核心API

核心API：https://airtest.readthedocs.io/en/latest/all_module/airtest.core.api.html



### 1、touch 用于触发点击事件

可以用图片识别的方式点击

```
>>> touch(Template(r"tpl1606730579419.png", target_pos=5))
```

可以点击指定坐标

```
>>> touch((100, 100))
```



