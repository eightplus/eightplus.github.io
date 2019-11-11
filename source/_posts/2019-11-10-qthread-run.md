---
title:      "Qt多线程之子类化QThread"
date:       2019-11-10 15:28:40
author:     "lixiang"
categories: Linux 编程
tags:
    - Qt
---

> Qt中提供了对于线程的支持，它提供了一些独立于平台的线程类QThread，要使用QThread进行多线程编程，有两种方式。一是子类化QThread，并重新实现run()函数；二是定义工作对象继承自QObject，然后把这个工作对象move到QThread的一个对象中。本文介绍第一种方法，在使用这种方法时，构造函数还是在原来的线程中执行，run()函数才在新的线程中执行，在示例代码中，通过打印的线程ID(QThread::currentThreadId())可以看出哪些代码在原线程中执行，哪些代码在新线程中执行。

---

## 示例源码
- [qthread-run](https://github.com/eightplus/examples/tree/master/code/Qt/thread/qthread-run)

## 开发库的安装
`$ sudo apt install qtbase5-dev qt5-qmake qtchooser qtscript5-dev qttools5-dev-tools qtbase5-dev-tools`

## 示例源码的编译和运行
```
$ qmake
$ make
$ ./qthread-demo
```

## 大概步骤
	1. 子类化QThread
	2. 重载run函数，run函数内有一个forever或while或for的死循环（模拟耗时操作）
	3. 设置一个标记为来控制死循环的退出

## 设置线程优先级
	- 通过start()设置，如：start(QThread::LowPriority)；
	- 通过setPriority(Priority priority)函数设置，这个函数设置正在运行线程的优先级，如果线程没有运行，此功能不执行任何操作并立即返回。

  | QThread::Priority枚举          |    值    |     描述
  | ----------------------------- | -------- | ---------------------
  | QThread::IdlePriority         |     0    | 没有其它线程运行时才调度
  | QThread::LowestPriority       |     1    | 比LowPriority调度频率低
  | QThread::LowPriority          |     2    | 比NormalPriority调度频率低
  | QThread::NormalPriority       |     3    | 操作系统默认的默认优先级
  | QThread::HighPriority         |     4    | 比NormalPriority调度频繁
  | QThread::HighestPriority      |     5    | 比HighPriority调度频繁
  | QThread::TimeCriticalPriority |     6    | 尽可能频繁的调度
  | QThread::InheritPriority      |     7    | 使用和创建线程同样的优先级（默认值）

## 示例代码分析要点
  示例代码主要为演示在run中执行耗时操作，如果想要耗时函数执行完成后发出通知，可以通过信号槽机制通知线程结束了，即发送一个信号。代码用到了QMutex和QWaitCondition。QMutex是强制执行互斥的基本类，一个线程锁定一个互斥量（mutex），以获得共享资源的访问权限，当互斥量已被锁定时，此时如果另一个线程试图锁定互斥量，则它将进入睡眠状态，直到第一个线程完成其任务并解锁互斥量。QWaitCondition同步线程，不通过执行互斥，而通过提供一个条件变量，QWaitCondition 使线程等待，直到满足一个特定的条件才让等待中的线程继续进行（其中QWaitCondition::wakeAll()唤醒所有在等待waitCondition的线程，这些线程被唤醒的顺序依赖于操组系统的调度策略，并且不能被控制或预知；QWaitCondition::wakeOne()唤醒一个随机的等待waitCondition的线程，线程的唤醒取决于操作系统的政策，不能被控制。）

  如下函数doLongTimeWork()，里面构造了一个局部的QMutexLocker对象并锁住互斥量，当QMutexLocker被销毁的时候，互斥量将被自动解锁(因为QMutexLocker是个局部变量，当函数返回时它就被销毁)，当执行函数doLongTimeWork()时，如果线程未运行，则调用start()开启线程，如果线程已经在运行了，则调用condition.wakeOne()唤醒线程。
  ```
  void WorkerThread::doLongTimeWork()
  {
      QMutexLocker locker(&mutex);

      if (!isRunning()) {
          start(QThread::LowPriority);
      }
      else {
          condition.wakeOne();
      }
  }
  ```

  run()中的condition.wait(&mutex)会让mutex会被暂时释放(会自动调用unlock解锁之前锁住的资源，避免造成死锁)，并阻塞在这个地方，当线程被cond.wakeOne()等唤醒时，mutex又会被重新锁定，并继续运行。
  ```
  void WorkerThread::run()
  {
      static int count = 0;
      forever {
          if (abort)
              return;
          mutex.lock();
          //当处于wait状态时mutex会被暂时释放(会自动调用unlock解锁之前锁住的资源，避免造成死锁)，并阻塞在这个地方；当线程被cond.wakeOne()等唤醒时，mutex又会被重新锁定，并继续运行
          this->loadDataFromFile();
          emit sendCount(count++);
          condition.wait(&mutex);//QWaitCondition 用于多线程的同步，一个线程调用QWaitCondition::wait() 阻塞等待，直到另一个线程调用QWaitCondition::wake() 唤醒才继续往下执行
          mutex.unlock();
      }
  }
  ```

  注意QMutex与QWaitCondition区别：
  - QMutex::lock相当于临界区锁，让处于锁中的资源不能被别的线程访问，即自己霸着不让别人用。
  - QWaitCondition::wait相当于互斥量锁，让线程等着，当别的线程访问完了，自己才能访问，即自己排队等着别人用完自己再用。
