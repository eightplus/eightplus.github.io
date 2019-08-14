---
title:      "安装Anbox"
date:       2019-08-14 21:44:03
author:     "lixiang"
categories: 安卓开发
tags:
    - Android Anbox
---

> Anbox是一种基于容器在常规GNU/Linux系统上启动完整Android系統的方法，如Ubuntu。
Anbox 使用 Linux 命名空间机制（user，pid，uts，net，mount，ipc），在容器中运行完整的 Android 系统，容器内的 Android 没有直接访问任何硬件的权限，所有的硬件访问通过主机上的 anbox 守护进程。Anbox复用基于 QEMU 的模拟器中为 Android 所做的 OpenGL ES 加速渲染的实现。容器内的 Android 系统使用不同的管道与主机系统通信，并通过它们发送所有的硬件访问命令。<br>

---

## 使用apt安装Anbox

- [安装内核模块](https://docs.anbox.io/userguide/install_kernel_modules.html)

    - anbox-modules-dkms包含ashmem和binder内核模块，安装anbox-modules-dkms后，必须手动加载内核模块。下次系统启动时，它们将自动加载。

`$ sudo add-apt-repository ppa:morphis/anbox-support`

`$ sudo apt update`

`$ sudo apt install linux-headers-generic anbox-modules-dkms`

`$ sudo modprobe ashmem_linux`

`$ sudo modprobe binder_linux`

`$ ls -1 /dev/{ashmem,binder}`
```
/dev/ashmem
/dev/binder
```

- 安装和更新Anbox snap

`$ sudo apt install snapd`

`$ sudo snap install --devmode --beta anbox`

`$ snap refresh --beta --devmode anbox`

- 查看当前可用的Anbox信息

`$ snap info anbox`

- 启动Anbox

`$ anbox.appmgr`

Anbox启动后，应用管理器界面如下所示：
![](2019-08-14-install-anbox/01.png)

点击应用管理器中的Settings的图标，启动Settings，界面如下所示：
![](2019-08-14-install-anbox/02.png)

- 安装安卓应用

`$ sudo apt install android-tools-adb`

`$ adb install my-app.apk`

- 卸载Anbox

`$ snap remove anbox`

`$ sudo apt install ppa-purge`

`$ sudo ppa-purge ppa:morphis/anbox-support`


## 参考文档

[Anbox主页](https://anbox.io/)

[Anbox代码托管](https://github.com/anbox)

[Android 硬件 OpenGL ES 模拟设计概述](https://www.wolfcstech.com/2017/09/16/opengles_android_emulation/)

[Android QEMU fast pipes](https://android.googlesource.com/platform/external/qemu/+/emu-master-dev/android/docs/ANDROID-QEMU-PIPE.TXT)

[The Android "qemud" multiplexing daemon](https://android.googlesource.com/platform/external/qemu/+/emu-master-dev/android/docs/ANDROID-QEMUD.TXT)

[Android qemud services](https://android.googlesource.com/platform/external/qemu/+/emu-master-dev/android/docs/ANDROID-QEMUD-SERVICES.TXT)
