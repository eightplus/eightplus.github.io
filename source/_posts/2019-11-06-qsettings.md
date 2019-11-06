---
title:      "Qt配置文件之QSettings"
date:       2019-11-06 09:30:05
author:     "lixiang"
categories: Linux 编程
tags:
    - Qt
---

> 在用Qt编程时，很多时候需要在本地保留用户的配置，方便下次启动程序的时候使用上次的配置数据，这里推荐使用QSettings读写配置文件（QSettings可重入，即可以同时在不同的线程中使用不同的QSettings对象），而不是去用数据库去记录和读取这些数据（如轻量级数据库sqlite）。

---

## 示例源码
- [qsetting-demo](https://github.com/eightplus/examples/tree/master/code/Qt/setttings/qsetting-demo)

## 开发库的安装
`$ sudo apt install qtbase5-dev qt5-qmake qtchooser qtscript5-dev qttools5-dev-tools qtbase5-dev-tools`

## 示例源码的编译安装和运行
```
$ qmake
$ make
$ ./qsettings-demo
```

## QSettings对象的创建
  QSettings对象既可以创建在栈上，也可以创建在堆上，构建和销毁也非常快。QSettings类的构造函数的重载允许我们用各种方法来初始化QSettings对象，比如可以在当创建QSettings对象时，通过指定公司或组织名称以及产品名称，也可以通过指定文件路径和Format类型，这里我们介绍用指定文件路径和Format类型的方式构造QSettings对象。

    - 栈上的对象
    ```
    QString filename = QDir::homePath() + "/.config/eightplus/qsettings-demo.ini";
    QSettings m_settings(filename, QSettings::IniFormat);
    m_settings.setIniCodec("UTF-8");
    ```

    - 堆上的对象
    ```
    QString filename = QDir::homePath() + "/.config/eightplus/qsettings-demo.ini";
    QSettings *m_settings = new QSettings(filename, QSettings::IniFormat);
    m_settings->setIniCodec("UTF-8");
    ```

    其中，枚举类型Format的定义如下：
    ```
    enum Format {
        NativeFormat,
        IniFormat,

        InvalidFormat = 16,
        CustomFormat1,
        CustomFormat2,
        CustomFormat3,
        CustomFormat4,
        CustomFormat5,
        CustomFormat6,
        CustomFormat7,
        CustomFormat8,
        CustomFormat9,
        CustomFormat10,
        CustomFormat11,
        CustomFormat12,
        CustomFormat13,
        CustomFormat14,
        CustomFormat15,
        CustomFormat16
    };
    ```
    在Linux上编程，我们介绍NativeFormat和IniFormat，对于我个人而已，我一直使用的是IniFormat。
  常量      | 值              | 描述
  -------- | --------------- | ---------------
  QSettings::NativeFormat | 0 | 使用平台最合适的存储格式设置。在Unix中，使用的是INI格式的文本配置文件。
  QSettings::IniFormat | 1 | 存储在INI文件中的设置。

## QSettings存储
  QSettings使用setValue()函数存储一系列设置，每个设置包括key（字符串）和一个与该key关联的value（QVariant），如：
  ```
  m_settings->beginGroup("General");
  m_settings->setValue("name", m_name);
  m_settings->setValue("male", m_male);
  m_settings->setValue("age", m_age);
  m_settings->endGroup();
  m_settings->sync();
  ```
  sync()：如果存在相同的key，现有的值将被新值覆盖。为了提高效率，这些变化可能不会被立即保存到永久存储（可以随时调用sync()来提交更改）。

## QSettings读取
  QSettings使用使用value()函数得到一个key的value，如：
  ```
  m_settings->beginGroup("General");
  m_name = m_settings->value("name", m_name).toString();
  if (m_name.isEmpty()) {
      m_name = "lixiang";
  }
  m_male = m_settings->value("male", m_male).toBool();
  m_age = m_settings->value("age", m_age).toInt();
  m_settings->endGroup();
  ```

## QVariant类型
  QSettings的value为QVariant类型，Qvariant是一种数据类型的集合，是Qt Core模块的一部分，它支持大部分通常的Qt数据类型转换，如：toInt()，toString(), toBool()，toPoint()，toSize()等，但它不能提供Qt GUI那一部分的Qt数据类型转换，如：QColor、QImage、 QPixmap，即QVariant中没有toColor()、toImage()、toPixmap()等接口，此时可以使用QVariant::value()或qVariantValue()模板函数，如下所示：
  ```
  QColor color = m_settings->value("color").value<QColor>();
  ```
  既然上面说到QVariant中没有toColor()等接口，那如果是QColor等类型，是否可以直接用setValue()去存储呢？答案是可以的，包括Qt Core和Qt GUI在类的相关类型，QVariant都支持，如下所示：
  ```
  QColor color = palette().background().color();
  m_settings->setValue("color", color);
  ```

  另外，使用qRegisterMetaType()和qRegisterMetaTypeStreamOperators()注册的自定义类型，也可以使用QSettings进行存储。

## ini配置文件示范
  ```
  [%General]
  age=18
  male=true
  name=lixiang
  ```
