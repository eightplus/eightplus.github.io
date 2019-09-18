---
title:      "Boost 序列化"
date:       2019-09-18 10:58:18
author:     "lixiang"
categories: Linux 编程
tags:
    - Boost
---

> 上一篇博客讲解过 protobuf 的序列化（将对象变成字节流的形式传出去）和反序列化（从字节流恢复成原来的对象），这篇博客将介绍另外一种序列化和反序列化的方案：Boost.Serialization。Boost.Serialization可以创建或重建程序中的等效结构，并保存为二进制数据、文本数据、XML或者有用户自定义的其他文件。

---

## 示例源码
- [boost-serialization](https://github.com/eightplus/examples/tree/master/code/C/boost)

## protobuf 与 Boost.Serialization 的比较

 1. protobuf:
 轻量级的，支持的数据类型有限，且数据对象必须预先定义，使用 protoc 编译，但其效率较高，适合要求效率，允许自定义类型的内部场合使用。

 2. Boost.Serialization
 Boost 库非常庞大，功能丰富，Boost.Serialization序列化只是其中的一个小分支，但就算只使用序列化，也需要安装整个Boost库，其支持的序列化功能强大，既支持二维数组（指针），也支持STL容器，序列化使用灵活简。

## Boost.Serialization 的两种模式介绍

  Boost序列化可以分为两种模式：侵入式（intrusive）和非侵入式 （non-intrusive）

  1. 侵入式（intrusive）

  侵入式序列化时，需要在class里面加入序列化的代码，序列化的步骤大致如下：

    - 先引用 boost 头文件
    - 在类的声明中, 编写序列化函数，该函数的格式如下:
      ```
      template<class Archive>
      void serialize(Archive & ar, const unsigned int version)//version是版本号
      {
        ar& m_str;
      }
      ```
     - 类的实例化和赋值
     - 定义一个序列化的对象和数据流的写入(具体序列化和反序列化的代码见示例源码)
      ```
      boost::archive::text_oarchive text_oa(text_sstream);//文本方式
      boost::archive::binary_oarchive binary_oa(binary_sstream);//二进制方式
      binary_oa << info;//将对象info的序列化数据以二进制存储形式写入内存
      ```

  2. 非侵入式（non-intrusive）

  如果class是早已存在的，且我们不想再改变class里面的代码时，这个时候，我们可以使用非侵入式的序列化。非侵入式序列化时，序列化函数需要访问数据成员，这就要求将class的数据成员暴露出来，即public，而不是private。其序列化的步骤和上面的侵入式序列化步骤一致。

## Boost 开发库的安装
`$ sudo apt install libboost-dev libboost1.58-dev libboost-serialization1.58-dev`
