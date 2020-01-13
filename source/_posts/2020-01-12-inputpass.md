---
title:      "鼠标点击穿透"
date:       2020-01-12 12:39:05
author:     "lixiang"
categories: Linux 编程
tags:
    - Qt X11
---

> 鼠标点击穿透，即所有鼠标键盘操作全部会穿透窗口到下方窗口。在Linux下，Qt的Qt::WA_TransparentForMouseEvents属性可以对子部件实现鼠标穿透，但是对整个窗口不行，要想在Linux下对Qt的整个窗口设置鼠标穿透，这时候可以用到X11中x11shape的XShapeCombineRectangles()库函数，很多窗口挂件都是这样做的，如osd桌面歌词程序等等。示例代码编写了一个简单的桌面水印，在桌面的右下方显示了一张图片和一段文字描述，在设置了鼠标点击穿透的情况下用鼠标点击该水印，是可以直接穿透该水印的。

---

## 示例源码
- [inputpass](https://github.com/eightplus/examples/tree/master/code/Qt/inputpass)

## 开发库的安装
`$ sudo apt install qtbase5-dev qt5-qmake qtchooser qtscript5-dev qttools5-dev-tools qtbase5-dev-tools libqt5x11extras5-dev libx11-dev libxext-dev`

## 示例源码的编译和运行
```
$ qmake
$ make
$ ./inputpass
```

## XShapeCombineRectangles库函数参数分析
```
void XShapeCombineRectangles (
  Display *dpy,
  XID dest,
  int destKind,
  int xOff,
  int yOff,
  XRectangle *rects,
  int n_rects,
  int op,
  int ordering);
```
在设置鼠标穿透的时候给函数传的第六个参数为NULL（参数为XRectangle*类型），那整个窗口都将被穿透，第七个参数就是控制设置穿透的，为0时表示设置鼠标穿透，为1时表示取消鼠标穿透，当取消设置鼠标穿透的时候，必须设置区域，即第六个参数不再为NULL(当第6个参数XRectangle的x，y，width和height都为0,且第七个参数设置为1时，也表示设置为鼠标穿透)。

## 示例代码分析要点

  1. 在Qt的.pro文件中，需要注意如下三行，即链接x11和xext库。
  ```
  QT       += x11extras
  CONFIG += link_pkgconfig
  PKGCONFIG += x11 xext
  ```
  2. 设置鼠标穿透

  头文件 X11/extensions/shape.h 来自于libxext-dev包。

  ```
  #include <QtX11Extras/QX11Info>
  #include <X11/extensions/shape.h>

  void Utils::passInputEvent(int wid)
  {
      XRectangle *reponseArea = new XRectangle;
      reponseArea->x = 0;
      reponseArea->y = 0;
      reponseArea->width = 0;
      reponseArea->height = 0;

      //XShapeCombineRectangles(QX11Info::display(), wid, ShapeInput, 0, 0, reponseArea ,1 ,ShapeSet, YXBanded);
      XShapeCombineRectangles(QX11Info::display(), wid, ShapeInput, 0, 0, NULL, 0, ShapeSet, YXBanded);

      delete reponseArea;
  }
  ```

  3. 取消鼠标穿透
  ```
  void Utils::unPassInputEvent(int wid, QSize size)
  {
      XRectangle *reponseArea = new XRectangle;
      reponseArea->x = 0;
      reponseArea->y = 0;
      reponseArea->width = size.width();
      reponseArea->height = size.height();

      XShapeCombineRectangles(QX11Info::display(), wid, ShapeInput, 0, 0, reponseArea ,1 ,ShapeSet, YXBanded);

      delete reponseArea;
  }
  ```
