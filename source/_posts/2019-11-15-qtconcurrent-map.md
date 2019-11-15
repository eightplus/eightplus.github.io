---
title:      "Qt多线程之QtConcurrent::map()"
date:       2019-11-15 15:53:10
author:     "lixiang"
categories: Linux 编程
tags:
    - Qt
---

> QtConcurrent::map()、QtConcurrent::mapped()和QtConcurrent::mappedReduced()函数可以对一个序列中（如QList、QVector等）的项目并行地进行计算。

---

## 示例源码
- [qtconcurrent-map](https://github.com/eightplus/examples/tree/master/code/Qt/thread/qtconcurrent-map-demo)

## 开发库的安装
`$ sudo apt install qtbase5-dev qt5-qmake qtchooser qtscript5-dev qttools5-dev-tools qtbase5-dev-tools`

## 示例源码的编译和运行
```
$ qmake
$ make
$ ./qtconcurrent-map-demo
```

## 示例代码分析要点
### QtConcurrent::map()
  如果想直接修改一个序列，可以使用QtConcurrent::map()。这个函数会将作为参数传入的函数应用到容器中的每一项，对这些项进行就地修改，即QtConcurrent::map()的序列被直接修改。虽然QtConcurrent::map()不通过QFuture返回任何结果（map函数的返回值、返回类型没有被使用）但仍然可以使用QFuture和QFutureWatcher监控map的状态。map函数有两个参数，第一个是序列(如QList)，第二个参数是一个函数。它的作用就是同时用第二个参数来计算第一个参数中的每一个元素，且结果直接覆盖到元素中。示例代码中m_list的值由原来的(1, 3, 5, 7, 9)变成了(2, 4, 6, 8, 10)，函数my_fun1被调用了5次，且根据打印处理的线程id可以判断这5次调用都是在不同的线程中执行的，而且都和主进程的线程id不同：
  ```
  void my_fun1(int &value)
  {
    qDebug() << "run my_fun1 thread id:" << QThread::currentThreadId();
    value += 1;
  }

  void MainWindow::onMapBtnClicked()
  {
    QList<int> m_list;
    m_list << 1 << 3 << 5 << 7 << 9;

    QFuture<void> future = QtConcurrent::map(m_list, my_fun1);
    future.waitForFinished();
    qDebug() << "m_list: "<< m_list;
  }
  ```

### QtConcurrent::mapped()
  QtConcurrent::mapped()函数与QtConcurrent::map()函数类似，区别在于QtConcurrent::mapped()函数接受一个输入序列和一个map函数，返回一个包含修改内容的新序列（即把计算结果放到了新的容器中），适用于进行数据处理后需要整理为新的数据结构的情况。
  * 从如下的示例代码的运行结果中我们可以看到原容器的值没有发生改变，原容器的值为 (2, 4, 6, 8, 10)，修改后的新容器的值为(1, 3, 5, 7, 9)，函数my_fun2被调用了5次，且根据打印处理的线程id可以判断这5次调用都是在不同的线程中执行的，而且都和主进程的线程id不同：
  ```
  int my_func2(int value)
  {
    qDebug() << "run my_func2 thread id:" << QThread::currentThreadId();
    return value - 1;
  }

  void MainWindow::onMappedBtnClicked()
  {
    QList<int> m_list;
    m_list << 2 << 4 << 6 << 8 << 10;

    QFuture<int> future = QtConcurrent::mapped(m_list, my_func2);
    future.waitForFinished();
    qDebug()<<"m_list: " << m_list;
    qDebug()<<"new list: " << future.results();
  }
  ```

  * 下面再介绍一段稍微复杂点的关于QtConcurrent::mapped()的代码片段，在加载多张照片的过程中，体现如何进行线程执行的进度、取消/暂停/恢复等。点击示例代码中的“打开”按钮时，调用槽函数onOpenBtnClicked()，默认打开图片目录进行图片选择。然后根据图片创建对应数量QLabel去显示缩放的图片，为了体现出一个耗时处理进度，我在代码图片缩放处理的过程中特意添加了sleep()，并添加了一个进度条。图片选择完成后，使用mapped()进行并行计算，并添加至QFutureWatcher中，让其使用信号和槽监视QFuture。当QFutureWatcher的progressRangeChanged()的信号触发时，进度条的范围会发生改变，而progressValueChanged()信号触发时，会更新进度条的值。如果加载的图片较多，可以通过点击“取消”按钮，这时会调用QFutureWatcher的cancel()槽函数来取消计算。“暂停/恢复”则调用togglePaused()槽函数，用于切换异步计算的暂停状态（如果计算当前已暂停，调用此函数将进行恢复；如果计算正在运行，则会暂停）。
  ```
  void MainWindow::onOpenBtnClicked()
  {
    if (m_watcher->isRunning()) {
        m_watcher->cancel();
        m_watcher->waitForFinished();
    }

    QStringList files = QFileDialog::getOpenFileNames(this, tr("Select Images"),
                            QStandardPaths::writableLocation(QStandardPaths::PicturesLocation), "*.jpg *.png");

    if (files.isEmpty())
        return;

    qDeleteAll(m_labels);
    m_labels.clear();

    int dim = qSqrt(qreal(files.count())) + 1;
    for (int i = 0; i < dim; ++i) {
        for (int j = 0; j < dim; ++j) {
            QLabel *imageLabel = new QLabel;
            imageLabel->setFixedSize(imageSize,imageSize);
            m_imagesLayout->addWidget(imageLabel,i,j);
            m_labels.append(imageLabel);
        }
    }

    std::function<QImage(const QString&)> image_scale = [imageSize](const QString &imageName) {
        //在线程中执行，所以线程id与主进程的不一样
        static int count = 0;
        qDebug() << "run image_scale thread id:" << QThread::currentThreadId();
        QImage image(imageName);
        count += 3;
        sleep(count);
        return image.scaled(QSize(imageSize, imageSize), Qt::IgnoreAspectRatio, Qt::SmoothTransformation);
    };

    auto future = QtConcurrent::mapped(files, image_scale);
    m_watcher->setFuture(future);

    m_openBtn->setEnabled(false);
    m_cancelBtn->setEnabled(true);
    m_pauseBtn->setEnabled(true);
  }
  ```
  当然，上面出现的image_scale()函数也可以写成如下格式：
  ```
  QImage image_scale(const QString &imageName)
  {
    //在线程中执行，所以线程id与主进程的不一样
    static int count = 0;
    qDebug() << "run image_scale thread id:" << QThread::currentThreadId();
    QImage image(imageName);
    count += 3;
    sleep(count);

    return image.scaled(QSize(imageSize, imageSize), Qt::IgnoreAspectRatio, Qt::SmoothTransformation);
  }
  ```

### QtConcurrent::mappedReduced()
  QtConcurrent::mappedReduced()功能类似于mapped()，但它不是返回具有新结果的序列，它比mapped()多一个参数，这个参数也是个函数，它会将修改过的每一项再传入这个比mapped()多的参数函数中进行简化，将多个结果按某种要求简化成一个。mappedReduced函数遵循如下格式： V function(T &result, const U &intermediate)， 其中result就是最后的结果，intermediate就是mapped出来的结果。
  * 从如下示例代码的运行结果中我们可以看到result的值为30,即将每个新值加起来的总和，函数my_sum(即称为reduce的函数)将由map函数返回的每个结果调用一次，并且应该合并中间体到结果变量。QtConcurrent::mappedReduced()可以保证保证一次只有一个线程调用reduce，所以没有必要用一个mutex锁定结果变量。QtConcurrent::ReduceOptions枚举提供了一种方法来控制reduction完成的顺序。如果使用了QtConcurrent::UnorderedReduce（默认），顺序是不确定的；而QtConcurrent::OrderedReduce确保reduction按照原始序列的顺序完成。示例中函数my_fun3()被调用了5次，且根据打印处理的线程id可以判断这5次调用都是在不同的线程中执行的，而且都和主进程的线程id不同，另外函数my_sum()也被调用了5次，但这五次的线程id一样，且和上面第一次执行函数my_func3的线程id一致。

  ```
  int my_func3(int value)
  {
    qDebug() << "run my_func3 thread id:" << QThread::currentThreadId();
    return value * 2;
  }

  void my_sum(int &result, const int &intermediate)
  {
    qDebug() << "run my_sum thread id:" << QThread::currentThreadId();
    result += intermediate;
  }

  void MainWindow::onMappedReducedBtnClicked()
  {
    QList<int> m_list;
    m_list<< 1 << 2 << 3 << 4 << 5;

    QFuture<int> future = QtConcurrent::mappedReduced(m_list, my_func3, my_sum);
    future.waitForFinished();

    qDebug() << "result:" << future.result();//30
  }
  ```

  * 下面接着使用QtConcurrent::mappedReduced()来比较使用多线程和单线程处理的效率，示例代码片段的作用是统计用户主目录下所有的.cpp和.h文件中的单词数目，比较使用主线程计算和使用多线程并行计算的耗时，如果你的.cpp和.h文件数目够多，可以通过打印看出两者耗时的明显差距，并且在计算很多文件的过程中，使用主线程计算将会导致界面卡顿，而使用多线程将不会影响界面的event事件。
   - 使用单线程计算

   ```
    WordCount countWordsWithSingleThread(const QStringList &files)
    {
      WordCount wordCount;
      for (const QString &file : files) {
          QFile f(file);
          f.open(QIODevice::ReadOnly);
          QTextStream textStream(&f);
          while (!textStream.atEnd()) {
              const auto words =  textStream.readLine().split(' ');
              for (const QString &word : words)
                  wordCount[word] += 1;
          }
      }

      return wordCount;
    }
    ```

    - 使用多线程并行计算

    ```
    WordCount countWordsWithMultiThreads(const QString &file)
    {
      WordCount wordCount;

      QFile f(file);
      f.open(QIODevice::ReadOnly);
      QTextStream textStream(&f);

      while (!textStream.atEnd()) {
          const auto words =  textStream.readLine().split(' ');
          for (const QString &word : words)
              wordCount[word] += 1;
      }

      return wordCount;
    }

    void reduceWordCounts(WordCount &result, const WordCount &w)
    {
      QMapIterator<QString, int> i(w);
      while (i.hasNext()) {
          i.next();
          result[i.key()] += i.value();
      }
    }

    void MainWindow::onMappedReducedBtn2Clicked()
    {
      QStringList files = findAllFiles("/home/lixiang/work", QStringList() << "*.cpp" << "*.h");

      // Single thread
      QTime time1;
      time1.start();
      WordCount total1 = countWordsWithSingleThread(files);
      qDebug() << "Single thread find" << total1.size() << "words, Using " << time1.elapsed() / 1000.0 << "seconds";

      // Multi thread
      QTime time2;
      time2.start();
      WordCount total2 = QtConcurrent::mappedReduced(files, countWordsWithMultiThreads, reduceWordCounts);
      qDebug() << "Multi thread find " << total2.size() << "words, Using " << time2.elapsed() / 1000.0 << "seconds";
    }
    ```
