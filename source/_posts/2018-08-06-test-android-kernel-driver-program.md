---
title:      "在Android源码中编写C程序测试Linux内核驱动程序"
date:       2018-08-06 22:33:11
author:     "lixiang"
categories: 工具
tags:
    - Android
---

> 在mydev的Linux内核驱动程序中，创建三个不同的文件节点来供用户空间访问，分别是传统的设备文件/dev/mydev、proc系统文件/proc/mydev和devfs系统属性文件/sys/class/mydev/mydev/val。这里通过编写的C可执行程序来访问设备文件/dev/mydev。<br><br>

---

- 驱动程序目录结构如下：
```
~/Android
----external
    ----mydev
        ----mydev.c
        ----Android.mk
```

- 使用Android模拟器加载包含这个Linux驱动程序的内核文件，并且使用adb shell命令连接上模拟，验证在/dev目录中存在设备文件mydev。

- 进入到Android源代码工程的external目录，创建mydev目录
* ~/Android$ cd external
* ~/Android/external$ mkdir mydev

- 在mydev目录中新建mydev.c文件
    - 打开/dev/mydev文件，然后先读出/dev/mydev文件中的值，接着写入值5到/dev/mydev中去，最后再次读出/dev/mydev文件中的值，看看是否是我们刚才写入的值5。从/dev/mydev文件读写的值实际上就是我们虚拟的硬件的寄存器val的值。

``` cpp
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#define DEVICE_NAME "/dev/mydev"

int main(int argc, char** argv)
{
    int fd = -1;
    int val = 0;
    fd = open(DEVICE_NAME, O_RDWR);
    if(fd == -1) {
        printf("Failed to open device %s.\n", DEVICE_NAME);
        return -1;
    }

    printf("Read original value:\n");
    read(fd, &val, sizeof(val));
    printf("%d.\n\n", val);

    val = 5;
    printf("Write value %d to %s.\n\n", val, DEVICE_NAME);
    write(fd, &val, sizeof(val));

    printf("Read the value again:\n");
    read(fd, &val, sizeof(val));
    printf("%d.\n\n", val);

    close(fd);

    return 0;
}
```

- 在mydev目录中新建Android.mk文件，其中“BUILD_EXECUTABLE”表示编译的是可执行程序
```
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE_TAGS := optional
LOCAL_MODULE := mydev
LOCAL_SRC_FILES := $(call all-subdir-c-files)
include $(BUILD_EXECUTABLE)
```

- 使用mmm命令进行编译
* ~/Android$ mmm ./external/mydev
若编译成功，则可以在out/target/product/gerneric/system/bin目录下看到可执行文件mydev。

- 重新打包Android系统文件system.img，重新打包后的system.img文件包含了刚才编译好的mydev可执行文件。
* ~/Android$ make snod

- 运行模拟器，使用/system/bin/mydev可执行程序来访问Linux内核驱动程序
* ~/Android$ emulator -kernel ./kernel/goldfish/arch/arm/boot/zImage &
* ~/Android$ adb shell
* root@android:/ # cd system/bin
* root@android:/system/bin # ./mydev
```
Read the original value:
0.
Write value 5 to /dev/mydev.
Read the value again:
5.
```
