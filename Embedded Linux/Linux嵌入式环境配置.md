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
# 安装低版本 GCC（解决高版本编译旧 Linux 内核报错）

[Ubuntu安装低版本gcc详细教程（安装gcc6.3.0为例）-CSDN博客](https://blog.csdn.net/weixin_46584887/article/details/122527982)

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