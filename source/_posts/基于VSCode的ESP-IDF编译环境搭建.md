---
title: 基于VSCode的ESP-IDF编译环境搭建
toc: true
date: 2025-02-25 20:51:08
categories: 嵌入式
tags:
    - ESP32
---

## 前言
项目要求，需要搭建一个ESP32的开发环境。目前，主流的开发方式都是使用VSCode搭配ESP-IDF插件，但是我在本地搭建环境时，遇到了各种问题，在本文记录下来，作为以后开发的参考。

<!-- more -->

## 自以为是的搭建方法
按照此前VSCode的开发习惯，在VSCode中安装好插件之后，应该就已经能够进行开发了。ESP-IDF的VSCode插件安装好后，会自动提示进行初始化的Config，按理来说，根据提示一步步操作之后，就能够自动下载好ESP-IDF的库和编译工具链，以及所需要的Python虚拟环境等。我按照这个提示操作完成之后，几个项目都无法编译，提示五花八门，有时候提示缺少python的库，有时候提示语法错误，总之就是不能编译

## 正确的搭建方法
参考<https://blog.csdn.net/shengzhe8688/article/details/132331713>的搭建方法。
1. 删除所有的ESP-IDF路径和文件。
2. 去官网下载离线安装包，这里由于网站网址更改了，不能用CSDN博文的网址，应该使用<https://dl.espressif.com/dl/esp-idf/>。
3. 使用离线安装包安装ESP-IDF
4. 在VSCode中新建一个配置文件（我这才第一次知道有这个东西，还挺好用），也就是新开一个环境，不会有之前安装的扩展。
5. 安装ESP-IDF扩展
6. 在ESP-IDF配置的时候，选择Using Existing Setup。
7. 选择刚才ESP-IDF安装路径
8. 等着就行了
这样安装好之后可以正常编译了。