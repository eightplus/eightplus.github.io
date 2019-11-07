---
title:      "Qt配置文件之QGSettings"
date:       2019-11-07 16:05:18
author:     "lixiang"
categories: Linux 编程
tags:
    - Qt
---

> 本文介绍使用dconf作为后端的GSetting用法，dconf是一个简单的底层配置存储管理系统，可以使用图形化的dconf-editor根据path来检索GSettings并管理key，而且支持在key发生改变时发出通知(changed信号)。命令行工具gsettings提供了对GSetings的命令行操作，可以查询、读取和设置键值。如果你觉得glib、gio太复杂(查看GSettings的API文档可以使用工具devhelp)，那么使用Qt的QGSettings来操作gsetting将会让你备很轻松，而且代码的可读性将提升，本文提供的demo就是使用QGSettings来操作gsetting。

---

## 示例源码
  示例代码简要展示了如何使用QGSettings进行配置文件的读和写，以及监控配置文件中key的变化，并将其他途径导致的key的变化的value更新到代码中。
- [qgsetting-demo](https://github.com/eightplus/examples/tree/master/code/Qt/setttings/qgsetting-demo)

## 开发库的安装
`$ sudo apt install qtbase5-dev qt5-qmake qtchooser qtscript5-dev qttools5-dev-tools qtbase5-dev-tools libgsettings-qt-dev`

## 示例源码的编译和运行
注意：在Qt的工程文件.pro中，需要加入如下两行：
```
CONFIG += link_pkgconfig
PKGCONFIG += gsettings-qt
```

```
$ sudo install -m 0644 qgsettings-demo.gschema.xml /usr/share/glib-2.0/schemas
$ sudo glib-compile-schemas /usr/share/glib-2.0/schemas
$ qmake
$ make
$ ./qsettings-demo
```

## 配置文件
	GSettings的配置文件是xml格式的，文件需以.gschema.xml结尾，一个配置文件里面可以包含多个schema，每个schema可由多个key组成（key是无法通过代码动态创建的）。详细说明如下：

字段 | 说明
-------- | ---------------
id | schema中的id在整个配置系统中是唯一的，否则在执行glib-compile-schemas时会忽略重复的id的开头通常使用与应用相关的域名。
path | schema中的path必须是以/开头并且以/结尾，不能包含连续的/，path用于指定在storage中存储路径，可以与id不一致。
name | key的名称，需要在此schema中唯一，name的值由小写字母、数字和-组成，并且开头必须是小写字母，不能以-结尾，也不能出现连续的-。
type | key的类型，需要是GVariant支持的类型，除了可以使用基本的类型外，也可按照GVariant的方式组合类型。
default | key的默认值
summary | key的简单描述
description | key的详细描述

  在schema文件中，通常一个id对应一个固定的path，但也可不设置path，这样就是一个可重定位的schema，path两头必须都有/，否则会验证失败。但是需要注意如果没有指定path属性，则gsettings无法被dconf-editor读取。要想schema文件被GSettings使用，还需要借助编译器`glib-compile-schemas`将schema文件编译为二进制文件。GSettings读取XDG_DATA_DIRS下的glib-2.0/schemas路径，需要将schema文件放到环境变量XDG_DATA_DIRS/glib-2.0/schemas/路径下，一般为/usr/share/glib-2.0/schemas和/usr/local/share/glib-2.0/schemas。这里我们将qgsettings-demo.gschema.xml文件拷贝到/usr/share/glib-2.0/schemas路径下，然后执行如下一条命令将其编译刷新系统的gsettings中，否则使用的时候将报错并导致程序退出（Settings schema 'apps.eightplus.qgsettings-demo' is not installed）：
  ```
  $ sudo glib-compile-schemas /usr/share/glib-2.0/schemas
  ```

## gsettings命令
命令 | 说明
-------- | ---------------
gsettings list-schemas  | 显示系统已安装的不可重定位的schema(已安装并已安装并有固定path的 schema)。
gsettings list-relocatable-schemas  | 显示已安装的可重定位的schema(已安装却没有固定path的 schema)。
gsettings list-children SCHEMA      | 显示指定schema的children，其中SCHEMA为xml文件中schema的id属性值，如示例代码中daemon的"apps.eightplus.qgsettings-demo"。
gsettings list-keys SCHEMA          | 显示指定schema的所有项(key)。
gsettings range SCHEMA KEY          | 查询指定schema的指定项KEY的有效取值范围。
gsettings get SCHEMA KEY            | 显示指定schema的指定项KEY的值。
gsettings set SCHEMA KEY VALUE      | 设置指定schema的指定项KEY的值为VALUE。
gsettings reset SCHEMA KEY          | 恢复指定schema的指定项KEY的值为默认值。
gsettings reset-recursively SCHEMA  | 恢复指定schema的所有key的值为默认值。
gsettings list-recursively [SCHEMA] | 如果有SCHEMA参数，则递归显示指定schema的所有项(key)和值(value)，如果没有SCHEMA参数，则递归显示所有schema的所有项(key)和值(value)。

 * 举例
  - 查看apps.eightplus.qgsettings-demo是否已经安装
  ```
  lixiang@lixiang-TM1701:~$ gsettings list-schemas |grep apps.eightplus.qgsettings-demo
  apps.eightplus.qgsettings-demo
  ```
  - 查看apps.eightplus.qgsettings-demo的schema下所有项的取值
  ```
  lixiang@lixiang-TM1701:~$ gsettings list-recursively  apps.eightplus.qgsettings-demo
  apps.eightplus.qgsettings-demo male false
  apps.eightplus.qgsettings-demo age 22
  apps.eightplus.qgsettings-demo name 'lixiang'
  ```

  - 获取name的值
  ```
  gsettings get apps.eightplus.qgsettings-demo name
  ```

  - 设置name的值
  ```
  gsettings set apps.eightplus.qgsettings-demo name kobe
  ```

  - 恢复name的默认值
  ```
  gsettings reset apps.eightplus.qgsettings-demo name
  ```

  - 恢复所有key的默认值
  ```
  gsettings reset-recursively apps.eightplus.qgsettings-demo
  ```

## 参考文档

[https://developer.gnome.org/GSettings/](https://developer.gnome.org/GSettings/)
