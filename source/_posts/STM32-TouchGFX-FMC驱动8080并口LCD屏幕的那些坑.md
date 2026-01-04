---
title: STM32+TouchGFX+FMC驱动8080并口LCD屏幕的那些坑
toc: true
date: 2025-12-28 23:29:18
categories:
tags:
---

我也是疯了，想用8080并口的LCD屏幕。更疯狂的是，想用TouchGFX实现GUI。一路遇到了数不清的坑，最终也是实现了基本功能了。
<!-- more -->

## 基本思路 
我使用的是一块170*320的LCD屏幕，ST7789驱动，通过30Pin FPC排线与PCB连接。
ST7789的驱动，我想用CubeMX里，X-CUBE-DISPLAY这个库直接生成。GUI部分，想用TouchGFX实现。
需要解决的问题有三：
1. FMC的设置问题
2. TouchGFX与X-CUBE-DISPLAY的联合问题
3. TouchGFX的设置问题
## 第一大坑：FMC的设置问题

### 坑1：地址设置

FMC设置包括：Chip Select, Memory Type, LCD Register Select, Data width。以上设置内容都只能算基础。
Chip Select不同，会导致CS的IO口不同。例如Chip Select NE1对应PC7。
Memory Type区分SDRAM，PSRAM，NOR FLASH，LCD等。这里显然选择LCD。
LCD Register Select对应LCD接口的DC引脚。若LCD Register Select为0，则向寄存器写入。若LCD Register Select为1，则向内存写入。
Data width顾名思义。这里设置成8位。

其中最关键的有两个，一个是Chip Select，一个是LCD Register Select。
Bank0对应的地址范围，是从0x6000 0000到0x6FFF FFFF，NE1-NE4分别对应其中的四个子区域，即NE1的首地址位0x6000 0000。NE2的首地址是0x6400 0000。以此类推。因此Chip Select决定了LCD地址的段地址。

LCD Register Select有A0-A25可选。这个A0-A25对应到了地址的第0位到第25位。例如，如果选择A0，那么寄存器首地址就是0x6000 0000，内存首地址就是0x6000 0001，如果选择A1，则寄存器首地址就是0x6000 0000，内存首地址是寄存器首地址就是0x6000 0002。不论是地址哪一位接到LCD，对LCD来说都是可用的，但是对STM32而言，对应的地址完全不同。因此需要谨慎选择。我倾向于选择最低位。

![FMC地址设置](STM32-TouchGFX-FMC驱动8080并口LCD屏幕的那些坑/image.png)

NE1+A0的选择，得到的内存范围是0x6000 0000~0x63FF FFFF，寄存器是偶地址，内存是奇地址。实际上，因为地址线只有8位，加上A0，只有九位，因此理论上最多使用到0x6000 01FF

### 坑2：时序设置

