---
layout:     post
title:      "ubuntu snap 应用"
subtitle:   "Qt5 + QtDbus + snapcraft"
date:       2018-06-29 10:58:12 +0800
author:     "lixiang"
header-img: "img/bg-01.png"
header-mask: 0.3
catalog:     true
tags:
    - 前端开发
    - Qt5
    - Snap
---

> Ubuntu 从 16.04开始，支持两种格式的安装包：debian 和 snap，其中 snap 不依赖于操作系统及其发布版本， 其可以安装同样一个软件的不同版本且不造成任何的干扰。<br><br>
> snap 是一种全新的软件包管理方式，它类似一个容器拥有一个应用程序所有的文件和库，各个应用程序之间完全独立，解决了应用程序之间的依赖问题，但因其包含应用需要的文件和库，导致snap包过大，占用更多的磁盘空间（snap包包含一个私有的root文件系统，里面包含了依赖的软件包）。

---

## Snap命令简介

- 安装Snap的相关包
    - sudo apt update
    - sudo apt-get install snapd snapcraft build-essential

- 在snap store中查找snap包
    - snap find

- 显示已安装的snap应用列表
    - snap list

- 安装一个snap包
    - sudo snap install <snap_pkg_name>

- 更新一个snap包
    - sudo snap refresh <snap_pkg_name>

- 将snap包还原到以前安装的版本
    - sudo snap revert <snap_pkg_name>

- 卸载一个snap包
    - sudo snap remove <snap_pkg_name>

## 打造Snap应用

> 我这里配置的system dbus相关的.service和.conf文件根本在snap包中没有起到作用，debian包中是可以的，原因暂时没找到比较官方的答案。最终使用plug和slot的方式，并且是手动启动system dbus的方法才让图形程序从dbus daemon获取到数据（貌似是目前snap还不支持dbus的自动启动？？？）。

- 1、编码编写完毕后，在源码第一级目录下打开终端，执行：$snapcraft init，会自动生成snap相关文件，其中模板文件snapcraft.yaml将描述snap包的整个构建过程

- 2、修改模板文件snapcraft.yaml，具体见[源码][1]中的写法

- 3、终端执行：$snapcraft ，生成snap包

- 4、安装snap包：$sudo snap install system-tool_1.0.2_amd64.snap --devmode --dangerous

- 5、安装完成后，终端运行 $snap interfaces 可以查看到plug和slot。有资料显示，如果要让我的dbus服务程序和图形程序进行通信，还需要终端执行：$sudo snap connect system-tool:daemon-plug system-tool:daemon-slot ，对应的disconnect操作是：$sudo snap disconnect system-tool:daemon-plug system-tool:daemon-slot。此处不执行上述操作仍可以，有些不知所依然？？？可能是我这里在snap容器中关于dbus的使用方法没找到正解的原因。
![](/img/2018/snap/01.png)

- 6、手动启动dbus服务，打开终端执行：$sudo system-tool.system-tool-daemon

- 7、启动图形程序：$system-tool
![](/img/2018/snap/02.png)
*图形程序运行结果图*

- 8、如果想将自己开发的snap发布到snap商店，首先注册一个Ubuntu One帐号，[注册地址][2]

- 9、有了帐号之后，自然是一条流水线生产操作步骤：登录->注册应用名->上传snap包->退出

    - snapcraft login
    - snapcraft register system-tool  (**只有第一次上传该应用时才需要注册应用名**)
    - snapcraft push system-tool_1.0.2_amd64.snap --release beta
    - snapcraft logout

- 10、上传后，你可以在邮箱等着snapcraft系统给你回邮件啦！很遗憾，由于我使用了自定义的plug和slot，自动审核失败了，需要人工审核（之前验证过没有自定义slot和plug时，上传可以自动审核成功）。
![](/img/2018/snap/03.png)
![](/img/2018/snap/04.png)

## 参考文档

[https://forum.snapcraft.io/t/the-dbus-interface/2038](https://forum.snapcraft.io/t/the-dbus-interface/2038)

[https://github.com/snapcore/snapd/wiki/Interfaces#dbus](https://github.com/snapcore/snapd/wiki/Interfaces#dbus)

[https://forum.snapcraft.io/t/how-do-i-connect-a-snap-to-dbus/1533](https://forum.snapcraft.io/t/how-do-i-connect-a-snap-to-dbus/1533)

[https://github.com/snapcore/snapd/pull/2592](https://github.com/snapcore/snapd/pull/2592)

[https://blog.csdn.net/tq08g2z/article/details/78685011](https://blog.csdn.net/tq08g2z/article/details/78685011)

[https://www.2cto.com/kf/201607/528337.html](https://www.2cto.com/kf/201607/528337.html)

[https://blog.csdn.net/ubuntutouch/article/details/51886345](https://blog.csdn.net/ubuntutouch/article/details/51886345)

[https://blog.csdn.net/ubuntutouch/article/details/51953272](https://blog.csdn.net/ubuntutouch/article/details/51953272)

[https://www.msweet.org/blog/2018-01-23-snaps-and-gui-apps.html](https://www.msweet.org/blog/2018-01-23-snaps-and-gui-apps.html)

[https://github.com/ubuntu/snappy-playpen](https://github.com/ubuntu/snappy-playpen)

[https://code.launchpad.net/~dpm/ubuntu-calendar-app/snap-all-things](https://code.launchpad.net/~dpm/ubuntu-calendar-app/snap-all-things)

[1]: https://github.com/eightplus/system-tool
[2]: https://dashboard.snapcraft.io/openid/login
