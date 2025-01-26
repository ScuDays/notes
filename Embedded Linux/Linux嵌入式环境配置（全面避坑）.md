---
title: Linux嵌入式环境配置
date: 2025-01-25 18:47:37
modify: 2025-01-25 18:47:37
author: days
category: Embedded Linux
published: 2025-01-25
draft: false
description: ubuntu-vscode-linux驱动开发环境配置
---
# Ubuntu Linux 内核降级或升级到指定版本，更新引导

[Ubuntu Linux内核降级或升级到指定版本，更新引导 - 云吱 - 博客园](https://www.cnblogs.com/leebri/p/16786685.html)

# Ubuntu 22.04版本无法挂载NFS V2的解决方法

[Ubuntu 22.04版本无法挂载NFS V2的解决方法-OpenEdv-开源电子网](http://www.openedv.com/thread-345681-1-1.html)

# vscode免密连接ubuntu虚拟机

[vscode免密连接ubuntu虚拟机 - 一望天涯 - 博客园](https://www.cnblogs.com/berumottoX/articles/17826627.html)


# 解决 vscode 远程开发代码报错

参考[使用vscode编写linux驱动小技巧_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1Lv4y1V7eh/?spm_id_from=333.1007.top_right_bar_window_custom_collection.content.click&vd_source=fd1fa14d89a090434e77d898baf18cf0)

## 配置代码跳转

添加 c_cpp_properties. json 文件，在其中添加内核源码目录，使得 vscode 远程连接后包含了对应的内核文件，函数可以进行跳转

![image.png](https://raw.githubusercontent.com/ScuDays/MyImg/master/20250125185908.png)

``` c_cpp_properties.json
{

    "configurations": [

        {

            "name": "Linux",

            "includePath": [

                "${workspaceFolder}/**",

                "/home/days/linux/IMX6ULL_alientek/linux_source_code/linux-imx-rel_imx_4.1.15_2.1.0_ga_alientek/include",           "/home/days/linux/IMX6ULL_alientek/linux_source_code/linux-imx-rel_imx_4.1.15_2.1.0_ga_alientek/arch/arm/include",           "/home/days/linux/IMX6ULL_alientek/linux_source_code/linux-imx-rel_imx_4.1.15_2.1.0_ga_alientek/arch/arm/include/generated/"
           ],
            "defines": [],
            "compilerPath": "/usr/bin/clang",
            "cStandard": "c11",
            "cppStandard": "c++17",
            "intelliSenseMode": "clang-x64"
        }
    ],

    "version": 4
}
```

## 解决可跳转但仍报错问题


# ubuntu 图形界面配置 Clash

[Linux Clash 最速安装使用 - 知乎](https://zhuanlan.zhihu.com/p/2852384493)

# ubuntu 安装特定版本 gcc

![【正点原子】I.MX6U嵌入式Linux驱动开发指南V1.81-164-169_page-0001.jpg](https://raw.githubusercontent.com/ScuDays/MyImg/master/%E3%80%90%E6%AD%A3%E7%82%B9%E5%8E%9F%E5%AD%90%E3%80%91I.MX6U%E5%B5%8C%E5%85%A5%E5%BC%8FLinux%E9%A9%B1%E5%8A%A8%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%97V1.81-164-169_page-0001.jpg)

![【正点原子】I.MX6U嵌入式Linux驱动开发指南V1.81-164-169_page-0002.jpg](https://raw.githubusercontent.com/ScuDays/MyImg/master/%E3%80%90%E6%AD%A3%E7%82%B9%E5%8E%9F%E5%AD%90%E3%80%91I.MX6U%E5%B5%8C%E5%85%A5%E5%BC%8FLinux%E9%A9%B1%E5%8A%A8%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%97V1.81-164-169_page-0002.jpg)

![【正点原子】I.MX6U嵌入式Linux驱动开发指南V1.81-164-169_page-0003.jpg](https://raw.githubusercontent.com/ScuDays/MyImg/master/%E3%80%90%E6%AD%A3%E7%82%B9%E5%8E%9F%E5%AD%90%E3%80%91I.MX6U%E5%B5%8C%E5%85%A5%E5%BC%8FLinux%E9%A9%B1%E5%8A%A8%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%97V1.81-164-169_page-0003.jpg)

![【正点原子】I.MX6U嵌入式Linux驱动开发指南V1.81-164-169_page-0004.jpg](https://raw.githubusercontent.com/ScuDays/MyImg/master/%E3%80%90%E6%AD%A3%E7%82%B9%E5%8E%9F%E5%AD%90%E3%80%91I.MX6U%E5%B5%8C%E5%85%A5%E5%BC%8FLinux%E9%A9%B1%E5%8A%A8%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%97V1.81-164-169_page-0004.jpg)

![【正点原子】I.MX6U嵌入式Linux驱动开发指南V1.81-164-169_page-0005.jpg](https://raw.githubusercontent.com/ScuDays/MyImg/master/%E3%80%90%E6%AD%A3%E7%82%B9%E5%8E%9F%E5%AD%90%E3%80%91I.MX6U%E5%B5%8C%E5%85%A5%E5%BC%8FLinux%E9%A9%B1%E5%8A%A8%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%97V1.81-164-169_page-0005.jpg)

![【正点原子】I.MX6U嵌入式Linux驱动开发指南V1.81-164-169_page-0006.jpg](https://raw.githubusercontent.com/ScuDays/MyImg/master/%E3%80%90%E6%AD%A3%E7%82%B9%E5%8E%9F%E5%AD%90%E3%80%91I.MX6U%E5%B5%8C%E5%85%A5%E5%BC%8FLinux%E9%A9%B1%E5%8A%A8%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%97V1.81-164-169_page-0006.jpg)
