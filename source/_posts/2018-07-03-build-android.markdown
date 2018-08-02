---
title:      "Ubuntu下Android开发的基础知识"
date:       2018-07-03 11:01:16 +0800
categories: 工具
tags:
    - Android
---

> 这里使用Ubuntu 16.04的64位系统上进行Android源码的编译。<br><br>

---

## 一、环境准备
1. 安装ubuntu 16.04 amd64 系统
2. 安装Git
* ~$ sudo apt-get install git-core gnupg
3. 安装Java SDK(官方JDK下载地址：http://www.oracle.com/technetwork/java/javase/downloads/index.html)：
* $ sudo add-apt-repository "deb http://us.archive.ubuntu.com/ubuntu/ hardy multiverse"
* $ sudo apt-get update
* $ sudo apt-get install sun-java6-jre sun-java6-plugin
* $ sudo apt-get install sun-java6-jdk
4. 安装其他依赖包(valgrind为调试工具)
* $ sudo apt-get install flex bison gperf libsdl-dev libesd0-dev libwxgtk2.6-dev build-essential zip curl valgrind

## 二、下载Android源码
1. 下载repo工具
* $ wget https://dl-ssl.google.com/dl/googlesource/git-repo/repo
* $ chmod a+x repo
* $ sudo mv repo /bin/
2. 下载Android源码
* $ mkdir Android
* $ cd Android
* Android$ repo init -u https://android.googlesource.com/platform/mainifest(如果想下载Android 2.3.1，执行：repo init -u https://android.googlesource.com/platform/mainifest -b android-2.3.1_r1)
* Android$ repo sync

## 三、编译Android源码
* Android$make (make -j18)
编译结果的输出目录为out/target/product/$(TARGET_PRODUCT),TARGET_PRODUCT是一个环境变量，默认值为generic

## 四、编译SDK(可选)
* Android$make sdk
打包成功后，可以看到如下输出($USER$表示当前登录的用户名)：
```
Package SDK: out/host/linux-x86/sdk/android-sdk_eng.$USER$_linux-x86.zip
```

## 五、安装编译好的Android镜像到模拟器上
1. 设置环境变量(Android模拟器命令emulator位于Android/out/host/linux-x86/bin中，~/Android/out/target/product/generic是Android镜像存放目录)
* Android$export PATH=$PATH:~/Android/out/host/linux-x86/bin
* Android$export ANDROID_PRODUCT_OUT=~/Android/out/target/product/generic
2. 运行Android模拟器
* Android$emulator

启动Android模拟器需要四个文件，分别为zImage、system.img、userdata.img和ramdisk.img，其中zImage为Linux内核镜像文件，三个.img文件为Android系统镜像文件。如果不带任何参数来运行emulator命令，则Android模拟器默认使用的zImage文件是Android/out/host/linux-x86/bin中的kernel-qemu文件，默认使用的三个.img文件位于ANDROID_PRODUCT_OUT目录中。ANDROID_PRODUCT_OUT是一个环境变量，这里将它的值设置为Android源码编译结果的输出目录，如果不设置ANDROID_PRODUCT_OUT环境变量，就需要指定上述四个文件来启动Android模拟器，如下：
* Android$emulator - kernel ./prebuilt/android-arm/kernel/kernel-qemu -sysdir ./out/target/product/generic -system system.img -data userdata.img -ramdisk ramdisk.img

## 六、下载、编译和安装Android最新内核源代码
Android源代码工程默认是不包括它所使用的Linux内核源代码的，而是使用预先编译好的内核，即prebuilt/android-arm/kernel/kernel-qemu文件。如果我们需要运行定制的Linux内核，则需要下载Linux kernel源代码进行编译。
1. Linux kernel源代码下载
* ~/Android$ mkdir kernel
* ~/Android$ cd kernel
* ~/Android/kernel$ git clone http://android.googlesource.com/kernel/goldfish.git
2. 支线下载
下载完成后，在kernel目录下可以看到一个空的goldfish子目录，此时需要执行git checkout来指定需要的支线代码，在执行git checkout之前，可先执行git branch -a查看有哪些支线代码。这里选择android-gldfish-2.6.29
* ~/Android/kernel/goldfish$git branch -a
* ~/Android/kernel/goldfish$git checkout remotes/origin/archive/android-gldfish-2.6.29
3. 修改Android内核源码目录下Makefile文件中的CROSS_COMPILE变量
```
#ARCH 		?= (SUBARCH)
#CROSS_COMPILE	?=
ARCH		?= arm  #体系结构为arm
CROSS_COMPILE	?= arm-eabi- #交叉编译工具链前缀
```
4. 将交叉编译工具目录添加到环境变量PATH中
* ~/Android/kernel/goldfish$ export PATH=$PATH:~/Android/prebuilt/linux-x86/toolchain/arm-eabi-4.4.3/bin
5. 修改硬件配置文件goldfish_defconfig
* ~/Android/kernel/goldfish$ make goldfish_defconfig
6. 开始编译(在make之前，可以执行make menuconfig先配置一下编译选项)
* ~/Android/kernel/goldfish$ make
编译成功后，可看到下面两行：
```
OBJCOPY arch/arm/boot/zImage
Kernel: arch/arm/boot/zImage is ready
```

## 七、在模拟器中运行编译好的内核
1. 在启动模拟器之前，先设置模拟器的目录到环境变量$PATH中去
* ~/Android$export PATH=$PATH:~/Android/out/host/linux-x86/bin
2. 设置ANDROID_PRODUCT_OUT环境变量
* ~/Android$export ANDROID_PRODUCT_OUT=~/Android/out/target/product/generic
3. 在后台中指定内核文件启动模拟器
* ~/Android$emulator -kernel ./kernel/goldfish/arch/arm/boot/zImage &
4. 用adb工具连接模拟器，查看内核版本信息，看看模拟器上跑的内核是不是刚才编译出来的内核(adb位于~/Android/out/host/linux-x86/bin目录中)
````
~/Android$ adb shell
root@android:/ # cd proc
root@android:/proc # cat version
````

## 八、单独编译Android源代码中的模块
Android源码第一次执行make会耗时很久，之后如果修改了Android源码中的某个模块或者在Android源码工程新增一个自己的模块，则可以单独编译该模块，以及重新打包system.img。
1. 在Android源码目录下的build目录下，有个脚本文件envsetup.sh，执行这个脚本文件可以获得一些有用的工具，其中就包括mmm，该命令用来编译指定目录的所有模块，通常这个目录只包含一个模块。
2. 使用mmm命令来编译指定的模块，如Email应用程序
* ~/Android$mmm packages/apps/Email/
编译完成之后，就可以在out/target/product/generic/system/app目录下看到Email.apk文件了。Android系统自带的App都放在这具目录下。另外，Android系统的一些可执行文件，例如C编译的可执行文件，放在out/target/product/generic/system/bin目录下，动态链接库文件放在out/target/product/generic/system/lib目录下，out/target/product/generic/system/lib/hw目录存放的是硬件抽象层（HAL）接口文件.
3. 编译好模块后，重新打包system.img文件
* ~/Android$make snod
4. 运行模拟器：
* ~/Android$emulator

## 九、日志系统
Android系统在用户空间中提供了轻量级的logger日志系统，它是在内核中实现的一种设备驱动，与用户空间的logcat工具配合使用能够方便地跟踪调试程序。在Android系统中，分别为C/C++ 和Java语言提供两种不同的logger访问接口。C/C++日志接口一般是在编写硬件抽象层模块或者编写JNI方法时使用，而Java接口一般是在应用层编写APP时使用。
1. Android系统中的C/C++日志接口是通过宏来使用的。在system/core/include/android/log.h定义了日志的级别, 在system/core/include/cutils/log.h中，定义了对应的宏，如对应于ANDROID_LOG_VERBOSE的宏LOGV。
如果使用C/C++日志接口，只要定义自己的LOG_TAG宏和包含头文件system/core/include/cutils/log.h就可以了。
```
#define LOG_TAG "MY LOG TAG"
#include <cutils/log.h>
```
就可以了，例如使用LOGV：
LOGV("This is the log printed by LOGV in android user space.");//ALOGI

2. Android系统在Frameworks层中定义了Java日志接口（frameworks/base/core/java/android/util/Log.java）
如果使用Java日志接口，只要在类中定义的LOG_TAG常量和引用android.util.Log就可以了。
```
private static final String LOG_TAG = "MY_LOG_TAG";
Log.i(LOG_TAG, "This is the log printed by Log.i in android user space.");
```
**要查看这些LOG的输出，可以配合logcat工具**
- 如果是在自己编译的Android源代码工程中使用，则在后台中运行模拟器
    - ~/Android$ emulator &
- 启动adb shell工具
    - ~/Android$ adb shell
- 使用logcat命令查看日志
    - root@android:/ # logcat
