---
title: OpenCV-面试题库大全
topic: 大模型专项面试题库
sub_topic: ''
doc_type: md
source_url: https://lcnipys9u0xh.feishu.cn/wiki/JaWAwe9dmial6Mk0r7pcd3zDnRc
source_path: 大模型专项面试题库/OpenCV-面试题库大全.md
questions: 0
chars: 2071
build_date: '2026-11-20'
images_removed: 0
---

# OpenCV-面试题库大全

> 主题：大模型专项面试题库　｜　原始文件：大模型专项面试题库/OpenCV-面试题库大全.md

## 第一部分：基础与数据结构

1. OpenCV 中 cv::Mat 的存储机制是什么？解释 depth() 和 channels() 的区别。

cv::Mat 是 OpenCV 中用于存储图像和矩阵数据的核心结构，它本质上是一个头信息 + 数据指针的结构。

- 存储机制：数据在内存中通常是行优先连续存储的，但对于多通道图像（如 RGB），每个像素的多个通道是交错存储的（即 BGR BGR BGR ...）。step 成员变量表示每一行的字节数（包含填充对齐）。
- depth()：返回单个通道的像素深度（数据类型），如 CV_8U (0)、CV_32F (5) 等，仅代表一个通道的数据类型。
- channels()：返回图像的通道数（如灰度图为1，彩色图为3）。
- 组合：type() 返回 CV_8UC3 这种完整类型，结合了深度和通道数。
1. 解释 OpenCV 中 cv::Rect 的成员变量 x, y, width, height 以及 tl() 和 br() 的含义。

- x, y：表示矩形左上角的坐标（在图像坐标系中，原点位于左上角，x轴向右，y轴向下）。
- width, height：矩形的宽度和高度（像素单位）。
- tl() (top-left)：返回左上角点 Point(x, y)。
- br() (bottom-right)：返回右下角点 Point(x + width, y + height)。
- 注意：在 OpenCV 中，矩形区域通常包含左上角像素，但不包含右下角像素（左闭右开区间 [x, x+width)），这在进行图像 ROI（感兴趣区域）操作时非常重要，以避免索引越界。
1. OpenCV 中 BGR 色彩空间与 RGB 的区别是什么？为什么 OpenCV 默认使用 BGR？

- 区别：两者只是通道存储顺序不同。在 RGB 中，内存顺序为 R、G、B；在 OpenCV 的 BGR 中，内存顺序为 B、G、R。
- 原因：OpenCV 最初由 Intel 开发，其底层图像处理库（IPL）为了兼容 Windows 的 GDI（图形设备接口）以及早期摄像头的硬件输出格式，采用了 BGR 格式。这一历史原因被保留下来。
- 影响：当使用 matplotlib（默认 RGB）显示 OpenCV 图像时，如果不做转换（cv2.cvtColor(img, cv2.COLOR_BGR2RGB)），颜色会显示异常（红蓝颠倒）。

## 第二部分：图像处理与操作

1. 图像的卷积操作（Filter2D）是如何计算的？简述边界处理（Border Handling）的常见模式。

卷积操作是通过一个核（Kernel，通常为奇数大小）在图像上滑动，将核覆盖下的像素值与核系数相乘后求和，得到中心像素的输出值。

常见边界处理模式：

1. BORDER_CONSTANT：用常数填充（常用黑色 0）。
1. BORDER_REPLICATE：复制边缘像素（如 aaaaaa|abcdefgh|hhhhhhh）。
1. BORDER_REFLECT：镜像反射（如 fedcba|abcdefgh|hgfedcb）。
1. BORDER_REFLECT_101 (或 BORDER_DEFAULT)：镜像反射但不重复边界（gfedcb|abcdefgh|gfedcba），这是高斯模糊、Sobel 等算子的默认方式。
1. BORDER_WRAP：环绕，对侧像素拼接。
1. 解释高斯模糊（GaussianBlur）与中值模糊（medianBlur）的原理区别及适用场景。

- 高斯模糊：
- 原理：线性滤波。使用符合高斯分布的核进行加权平均。权重随距离中心距离增加而减小。
- 特点：平滑效果好，能有效抑制服从正态分布的噪声，但会模糊边缘。
- 适用：去噪预处理、图像缩放前的抗锯齿。
- 中值模糊：
- 原理：非线性滤波。取核窗口内所有像素值的中位数作为输出。
- 特点：对椒盐噪声（随机黑白点）有极佳的去除效果，且能较好地保留边缘细节（因为不涉及线性加权）。
- 适用：去除椒盐噪声、文档图像预处理。
1. 什么是图像直方图均衡化（Histogram Equalization）？它的作用是什么？

图像直方图均衡化是一种通过拉伸像素强度分布范围来增强图像对比度的方法。

- 原理：对于灰度图像，计算累积分布函数（CDF），然后将原始像素值映射到新的值，使得输出图像的直方图尽可能平坦（均匀分布）。
- 作用：对于曝光过度或曝光不足的图像，能够增强细节，使暗部变亮，亮部变暗，从而提升视觉对比度。
- 局限：全局均衡化可能过度增强噪声或导致图像失真。对于局部光照不均的情况，通常使用 CLAHE（对比度受限自适应直方图均衡化）。
1. 图像金字塔（Image Pyramid）是什么？简述高斯金字塔与拉普拉斯金字塔的区别。

图像金字塔是同一图像不同分辨率的集合。
