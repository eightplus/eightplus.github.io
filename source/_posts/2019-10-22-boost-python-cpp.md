---
title:      "通过boost.python将c++封装成动态库供python调用"
date:       2019-10-22 23:10:10
author:     "lixiang"
categories: Linux 编程
tags:
    - Boost Python C++
---

> 这里简要介绍如何通过boost.python将c++封装动态库，让python直接调用库中的函数，要做到这一点，我们需要让so中将要在python中直接调用的函数、类、结构体等，通过boost中特定函数暴露给python，而且so的目标名必须和BOOST_PYTHON_MODULE使用的module名一致（即生成的动态库命名为封装模块的名字），这样，在python就可以直接import该模块，并像调用其他模块的函数、类、结构体等一样，直接调用里边的函数、类、结构体......

---

## 示例源码
- [boost-python-cpp](https://github.com/eightplus/examples/tree/master/code/C/boost-python-cpp)

## 开发库的安装
`$ sudo apt install libboost-dev libboost1.58-dev libboost-python-dev gcc g++ pkg-config python-dev build-essential`

## 示例源码的编译安装和运行
```
$ make
$ sudo make install
$ python test.py
```

## C++封装动态库

 如示例代码中daemon.cc中的代码所示，我们实现了一个类Daemon，该类的构造函数有一个string类型的参数，类中有两个成员函数detect_os_info和get_os_info，而且还在Deamon类之外实现了一个普通函数get_username。如下所示的BOOST_PYTHON_MODULE代码，目的是导出类及部分成员，以及普通函数，这些导出的类和函数，将可以在python中调用。
  ```
  BOOST_PYTHON_MODULE(boost_py_cpp_module)
  {
      class_<Daemon, boost::noncopyable > ("Daemon", "This is the test daemon for python and C++", init<std::string>())//构造函数
              .def("detect_os_info", &Daemon::detect_os_info)//成员函数
              .def("get_os_info", &Daemon::get_os_info)//成员函数
              ;
      def("get_username", &get_username);//普通函数
  }
  ```

## python调用C++

  示例中的test.py展示了python是如何调用C++中的类和函数，具体代码如下（其中 import boost_py_cpp_module就是导入C++编写的模块，该模块的名字与C++生成的库的名字一致）：

  ```
  import os
  import sys
  import ctypes
  try:
      libc = ctypes.CDLL('libc.so.6')
      libc.prctl(15, 'ydaemon', 0, 0, 0)
  except: pass
  import boost_py_cpp_module

  def get_os():
      iface = boost_py_cpp_module.Daemon("OS")
      iface.detect_os_info()
      data = iface.get_os_info()
      return data

  def get_username():
      try:
          import boost_py_cpp_module
          username = boost_py_cpp_module.get_username()
          return username
      except:
          return "root"

  if __name__ == '__main__':
      print "OS info: ", get_os()
      print "User name: ", get_username()
  ```
