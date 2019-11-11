---
title:      "Qt多线程之子类化QObject"
date:       2019-11-11 16:43:21
author:     "lixiang"
categories: Linux 编程
tags:
    - Qt
---

> 上一篇文章讲述了子类化QThread，并重新实现run()函数，这篇文章将讲述多线程之定义工作对象继承自QObject，然后把这个工作对象move到QThread的一个对象中（moveToThread(QThread * thread)函数将工作类对象移到所创建的QThread对象中去执行），本方法巧妙的利用了Qt信号和槽机制，比如在UI主进程中想让线程执行一个耗时操作，这时候不是通过线程对象去直接调用继承于QObject的那个类的方法，而是通过信号槽绑定，UI主进程发送一个信号signal，继承于QObject的那个类的槽函数则是在新线程中执行了。同样，在示例代码中，通过打印的线程ID(QThread::currentThreadId())可以看出哪些代码在原线程中执行，哪些代码在新线程中执行。

---

## 示例源码
- [qthread-object](https://github.com/eightplus/examples/tree/master/code/Qt/thread/qthread-object)

## 开发库的安装
`$ sudo apt install qtbase5-dev qt5-qmake qtchooser qtscript5-dev qttools5-dev-tools qtbase5-dev-tools`

## 示例源码的编译和运行
```
$ qmake
$ make
$ ./thread-object-demo
```

## 大概步骤
	1. 在主线程中，创建一个QThread实例
	2. 子类化QObject，把耗时操作封装到子类(示例代码的Worker类)槽函数中
	3. 创建Worker实例，建立QThread实例与Worker实例之间的信号和槽关系
	4. 调用Worker实例的moveToThread(QThread * thread)函数，将它移动到创建的QThread线程中去
	5. 执行QThread线程的start()方法

## 示例代码分析要点
  示例代码和上一篇文章的示例代码区别不大，代码很短，容易看懂，按钮点击后，发送了一个名为requestDoWork的信号，这个信号绑定到了Worker类的槽函数doLongTimeWork，那么doLongTimeWork()函数就是在新线程中执行了。示例代码中体现线程创建和move的代码如下：
  ```
  m_worker = new Worker();
  m_thread = new QThread();
  m_worker->moveToThread(m_thread);//移动worker对象实例到线程中
  connect(m_worker, SIGNAL(destroyed(QObject*)), m_thread, SLOT(quit()));//当worker对象实例销毁时，退出线程
  connect(m_thread, SIGNAL(finished()), m_thread, SLOT(deleteLater()));//当线程结束时，销毁线程对象实例
  connect(m_worker, &Worker::sendCount, this, &MainWindow::onResponseCount);
  connect(this, SIGNAL(requestDoWork()), m_worker, SLOT(doLongTimeWork()));
  m_thread->start();//启动线程
  ```
