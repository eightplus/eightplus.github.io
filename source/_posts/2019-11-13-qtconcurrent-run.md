---
title:      "Qt多线程之QtConcurrent::run()"
date:       2019-11-13 20:14:36
author:     "lixiang"
categories: Linux 编程
tags:
    - Qt
---

> 前面两篇文章，我们介绍了QThread多线程的使用方法，今天开始介绍QtConcurrent，QtConcurrent是一个名字空间, 包含了众多的高级API, 方便用户编写多线程程序，且不需要使用低级线程原语(如:互斥、读写锁、等待条件或信号量)。QtConcurrent返回一个QFuture对象（QFuture类没有继承QObject，该类代表了一个异步调用的结果，它介于QtConcurrent和QFutureWatcher之间进行工作），这个对象可以用来查询任务当前的执行进度、暂停、恢复、取消任务以及获得运算结果，但并非所有的QFuture对象都支持暂停或取消的操作，如本篇文章将要介绍的由`QtConcurrent::run()`返回的QFuture对象则不能取消，而由QtConcurrent::mappedReduced()返回的对象则是是可以执行取消的。QFutureWatcher通过信号槽来跟异步调用实现交互，即可以通过信号槽来监视线程的完成情况（QFuture的进度）, 并获取线程的返回值。QFuture让线程可以通过某个后期产生的结果来实现同步，这个结果可以是任何拥有默认构造函数和拷贝构造函数的类型，如果这个结果在调用其result(), resultAt(), 或者results()方法时还没有准备好，QFuture将会一直等，直到结果准备好为止。在编写代码时，可以通过isResultReadyAt()方法来检测结果是否准备好，可以用绑定信号resultReadyAt来获取线程的结果。对于QFuture对象需要准备多个结果的情况，调用resultCount()方法可以返回连续结果的数量即我们可以从0到resultCount()进行遍历操作。

---

## 示例源码
- [qtconcurrent-run](https://github.com/eightplus/examples/tree/master/code/Qt/thread/qtconcurrent-run-demo)

## 开发库的安装
`$ sudo apt install qtbase5-dev qt5-qmake qtchooser qtscript5-dev qttools5-dev-tools qtbase5-dev-tools`

## 示例源码的编译和运行
```
$ qmake
$ make
$ ./qtconcurrent-run-demo
```

## 示例代码分析要点
  示例代码中模拟耗时操作的doLongTimeWork()函数有两种写法，如下面代码所示，QFuture<QList<CityInfo>>表示返回异步调用的结果，类型为QList<CityInfo>，而QFuture<void>比较特殊，它表示不返回异步调用的结果。任何QFuture<T>对象都可以赋值或者拷贝给QFuture<void>对象，QFuture<void>对于结果数据不需要而只需要状态或者进度信息的情况比较有用。

  写法一：
  ```
  void Worker::doLongTimeWork()
  {
      QFutureWatcher<QList<CityInfo>> *watcher = new QFutureWatcher<QList<CityInfo>>(/*this*/);
      QFuture<QList<CityInfo>> future = QtConcurrent::run(this, &Worker::loadDataFromFile);//开始异步调用，QtConcurrent::run()用于在另外的线程运行一个函数
      connect(watcher, &QFutureWatcher<QList<CityInfo>>::finished, this, &Worker::onInitFinished);
      connect(watcher, &QFutureWatcher<QList<CityInfo>>::resultReadyAt, this, &Worker::onResultReadyAt);
      watcher->setFuture(future);
  }
  ```
  写法二：
  ```
  void Worker::doLongTimeWork()
  {
      QFutureWatcher<QList<CityInfo>> *watcher = new QFutureWatcher<QList<CityInfo>>(/*this*/);
      QFuture<QList<CityInfo>> future = QtConcurrent::run(load_data_from_file);//开始异步调用，QtConcurrent::run()用于在另外的线程运行一个函数
      connect(watcher, &QFutureWatcher<QList<CityInfo>>::finished, this, &Worker::onInitFinished);
      connect(watcher, &QFutureWatcher<QList<CityInfo>>::resultReadyAt, this, &Worker::onResultReadyAt);
      watcher->setFuture(future);
  }
  ```

  通过QFutureWatcher的resultReadyAt信号绑定槽函数的onResultReadyAt，我们可以知道线程已经将什么样的结果已经准备好了，而通过QFutureWatcher的finished信号，我们可以知道线程执行完毕了，并可以通过wather来获取线程的结果，如下两个函数：
  ```
  void Worker::onResultReadyAt(int index)
  {
      QFutureWatcher<QList<CityInfo>> *watcher = static_cast<QFutureWatcher<QList<CityInfo>> *>(sender());
      if (!watcher)
          return;

      QList<CityInfo> info_list = watcher->resultAt(index);
      qDebug() << "length: "<< info_list.length() << ", index: " << index << ", region:" << info_list.at(0).region;
  }

  void Worker::onInitFinished()
  {
      static int count = 0;
      qDebug() << "Worker::onInitFinished";
      QFutureWatcher<QList<CityInfo>> *watcher = static_cast<QFutureWatcher<QList<CityInfo>> *>(sender());
      if (!watcher)
          return;

      m_cityInfos = watcher->result();
      for (const auto &info : m_cityInfos)
      {
          //qDebug() << "region:" <<  info.region;
      }

      watcher->deleteLater();

      emit scanFinished(count++);
  }
  ```
