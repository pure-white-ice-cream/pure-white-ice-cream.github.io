---
title: YOLOv7的人工智能目标检测实现
tags: []
id: "74"
categories:
  - - uncategorized
date: 2022-09-26 11:12:30
---

## 安装 python 环境

1、下载并安装 python3.X（3.7-3.10 任意版本均可），红框为 3.10.7 的 windows 64 位安装包 [点击进入官网](https://www.python.org/downloads/)

![](/images/YOLOv7的人工智能目标检测实现/python下载.png)

2、配置 python 环境，右键此电脑，点击属性，进入高级系统设置

![](/images/YOLOv7的人工智能目标检测实现/高级系统设置.png)

3、点击环境变量，双击 path，新建一条并填入 python 安装路径（如果已有，则不用填写）

![](/images/YOLOv7的人工智能目标检测实现/环境变量.jpg)

## 安装[PyTorch](https://pytorch.org/)

在官网选择对应 Cuda 版本获安装代码（如果没有 Cuda，在在[NVIDIA](https://www.nvidia.cn/)官网下载 [快速下载地址](https://developer.nvidia.com/cuda-downloads?target_os=Windows&target_arch=x86_64&target_version=10&target_type=exe_local)）

![](/images/YOLOv7的人工智能目标检测实现/安装PyTorch.png)

## 下载[YOLOv7](https://github.com/WongKinYiu/yolov7)

1、下载源码 [点击进入下载地址](https://github.com/WongKinYiu/yolov7)

![](/images/YOLOv7的人工智能目标检测实现/下载YOLOv7.png)

2、往下翻并下载权重，为了快速测试可用性，不用全下，下第一个即可

![](/images/YOLOv7的人工智能目标检测实现/下载权重.png)

3、用控制台 cd 到源码目录，执行以下代码（使用国内清华源，下载速度快）

```
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
```

![](/images/YOLOv7的人工智能目标检测实现/下载库.png)

## 运行

之前下载的权重放在源码目录，用控制台 cd 到源码目录，执行以下代码。

```
python detect.py
```

![](/images/YOLOv7的人工智能目标检测实现/运行.png)

结果保存在 “源码目录\\runs\\detect\\expX“

![](/images/YOLOv7的人工智能目标检测实现/输出.png)

## 可视化界面

安装可视化界面可以快速操作（尚未开发完）

![](/images/YOLOv7的人工智能目标检测实现/可视化.png)
