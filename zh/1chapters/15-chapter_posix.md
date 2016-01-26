# POSIX接口 #

## 简介 ##

[描述POSIX的历史情况，POSIX API情况]

[RT-Thread中对POSIX API的支持情况]

## 在RT-Thread中使用POSIX ##

在RT-Thread中使用POSIX API接口包括几个部分：libc（例如newlib），file system，pthread等。

需要在rtconfig.h中打开相关的选项：

    #define RT_USING_LIBC
    #define RT_USING_DFS
    #define RT_USING_DFS_DEVFS
    #define RT_USING_PTHREADS

## POSIX Thread介绍 ##

[API介绍]
