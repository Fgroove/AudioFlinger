---
title: Android HAL 与 HIDL 开发实例
author: Mista
date: 2022-07-14 00:55:00 +0800
categories: [Android, build]
tags: [android HIDL]
---

HAL 是最初的硬件抽象方案，在 Android 8 之后废弃并被 HIDL 取代。

HIDL，HAL Interface Definition Language，和AIDL类似，是用来描述硬件接口的语言。HIDL 设计的初衷是更新 frameworks 时避免重新编译 HAL，HAL 可以由厂商单独编译并定义在 vendor 分区中单独更新，以及进行版本管理。

# HIDL 开发实例

以下参考[AOSP相关实现](https://cs.android.com/android/platform/superproject/+/master:hardware/interfaces/audio/)，从零创建自己的HIDL服务，

