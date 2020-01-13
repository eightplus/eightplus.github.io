---
title:      "X11进行窗口状态设置"
date:       2020-01-13 21:49:40
author:     "lixiang"
categories: Linux 编程
tags:
    - Qt X11
---

> 在Qt的编程中，我们可以很方便的使用QWidget的一些属性和槽函数来进行窗口状态设置，比如`setWindowFlags(windowFlags() | Qt::WindowStaysOnTopHint)`可以设置窗口置顶，`showMinimized()`设置窗口最小化，`showMaximized()`设置窗口最大化，`showFullScreen()`设置窗口全屏，`showNormal()`恢复窗口的正常大小状态。但是在Linux不同的桌面环境和窗口管理器下，这些属性设置和槽函数的使用，往往会出现一些问题或者设置无效。我曾经遇到Qt4编写的代码在mate桌面环境下，不管窗口管理器是marco还是mutter，程序启动的时候设置置顶有效，但是程序运行过程中动态切换到置顶无效。我的测试代码将窗口隐藏后开启一个定时器想隔段时间后再自动弹出该窗口，并让该窗口处于激活状态，结果不行，而同一段代码使用Qt5编译运行，则正常。后来我用libwnck-dev中x11的接口解决了该问题（`libwnck/window-action-menu.c:wnck_window_make_above (window)-->libwnck/window.c:_wnck_change_state`）。上篇文章介绍了使用x11库函数来设置和取消鼠标点击穿透，本篇文章将介绍如何使用x11的库函数来设置窗口状态。

如下是我当时测试让隐藏的主界面重新显示并处于激活状态的代码：
  ```
    //for marco window manager
    this->setWindowState(Qt::WindowActive);
    this->activateWindow();
    this->show();
    this->raise();

    //for mutter window manager
    this->hide();
    this->setWindowState(Qt::WindowActive);
    this->activateWindow();
    this->show();
    this->raise();
```
---

## 示例源码
- [windowstate](https://github.com/eightplus/examples/tree/master/code/Qt/windowstate)

## 开发库的安装
`$ sudo apt install qtbase5-dev qt5-qmake qtchooser qtscript5-dev qttools5-dev-tools qtbase5-dev-tools libqt5x11extras5-dev libx11-dev libxext-dev`

## 示例源码的编译和运行
```
$ qmake
$ make
$ ./windowstate
```

## 示例代码分析要点

  1. 在Qt的.pro文件中，需要注意如下三行，即链接x11和xext库。
  ```
  QT       += x11extras
  CONFIG += link_pkgconfig
  PKGCONFIG += x11 xext
  ```
  2. 获取当前桌面属性值

  ```
  #include <QtX11Extras/QX11Info>
  #include <X11/extensions/shape.h>
  #include <X11/Xatom.h>
  #include <X11/Xlib.h>

  unsigned char *getWindowProperty(Display *display, Window win, const char *prop)
  {
      Atom reqAtom = XInternAtom(display, prop, true);
      if (reqAtom == None) {
          return NULL;
      }

      int retCheck = None;
      Atom retType = None;
      int retFormat = 0;
      unsigned long retItems = 0UL;
      unsigned long retMoreBytes = 0UL;
      unsigned char* retValue = NULL;

      retCheck = XGetWindowProperty(display, win, reqAtom, 0L, 0L, false, AnyPropertyType,
          &retType, &retFormat, &retItems, &retMoreBytes, &retValue);

      if (retValue != NULL) {
          XFree(retValue);
          retValue = NULL;
      }

      if (retCheck != Success || retType == None || retMoreBytes == 0) {
          return NULL;
      }

      retCheck = None;
      retFormat = 0;
      retItems = 0UL;

      // Convert the byte length into 32bit multiples.
      if (retMoreBytes % 4 != 0)
          retMoreBytes += 4 - retMoreBytes % 4;
      retMoreBytes /= 4;

      // Request the actual property value with correct length and type.
      retCheck = XGetWindowProperty(display, win, reqAtom, 0L, retMoreBytes, false, retType,
          &retType, &retFormat, &retItems, &retMoreBytes, &retValue);

      if (retCheck != Success || retMoreBytes != 0) {
          if (retValue != NULL)
              XFree(retValue);
          return NULL;
      }

      return retValue;
  }
  ```

  3. 设置窗口置顶/取消窗口置顶
  ```
  void Utils::setStayOnTop(int wid, bool top)
  {
      const auto display = QX11Info::display();//Display *display = QX11Info::display();

      const auto wmState = XInternAtom(display, "_NET_WM_STATE", false);
      const auto wmStateAbove = XInternAtom(display, "_NET_WM_STATE_ABOVE", false);
      const auto wmStateStaysOnTop = XInternAtom(display, "_NET_WM_STATE_STAYS_ON_TOP", false);

      XEvent xev;
      memset(&xev, 0, sizeof(xev));
      xev.xclient.type = ClientMessage;
      xev.xclient.display = display;
      xev.xclient.window = wid;
      xev.xclient.message_type = wmState;
      xev.xclient.format = 32;
      if (top)
          xev.xclient.data.l[0] = 1;// add/set property
      else
          xev.xclient.data.l[0] = 0;// remove/unset property
      xev.xclient.data.l[1] = wmStateAbove;
      xev.xclient.data.l[2] = wmStateStaysOnTop;
      xev.xclient.data.l[3] = 1;

      XSendEvent(display,
          QX11Info::appRootWindow(QX11Info::appScreen()),
          false,
          SubstructureRedirectMask | SubstructureNotifyMask,
          &xev);

      XFlush(display);
  }
  ```

  4. 设置/取消窗口全屏
  ```
  void Utils::showFullscreenWindow(int wid, bool fullscreen)
  {
      const auto display = QX11Info::display();//Display *display = QX11Info::display();
      const auto screen = QX11Info::appScreen();

      const Atom wmState = XInternAtom(display, "_NET_WM_STATE", false);
      const Atom wmfullscreenState = XInternAtom(display, "_NET_WM_STATE_FULLSCREEN", false);

      XEvent xev;
      memset(&xev, 0, sizeof(xev));
      xev.xclient.type = ClientMessage;
      xev.xclient.display = display;
      xev.xclient.window = wid;
      xev.xclient.message_type = wmState;
      xev.xclient.format = 32;
      if (fullscreen) {
          xev.xclient.data.l[0] = 1;//add/set property
      } else {
          xev.xclient.data.l[0] = 0;//remove/unset property
      }
      xev.xclient.data.l[1] = wmfullscreenState;
      xev.xclient.data.l[2] = 0;
      xev.xclient.data.l[3] = 1;

      XSendEvent(display,
                 QX11Info::appRootWindow(screen),
                 false,
                 SubstructureRedirectMask | SubstructureNotifyMask,
                 &xev
                );
      XFlush(display);
  }
  ```

  5. 设置/取消窗口最大化
  ```
  void Utils::showMaximizedWindow(int wid, int state)
  {
    // state:
    //      0: remove/unset property
    //      1: add/set property
    //      2: toggle property
      const auto display = QX11Info::display();//Display *display = QX11Info::display();
      const auto screen = QX11Info::appScreen();

      const Atom wmState = XInternAtom(display, "_NET_WM_STATE", false);
      const Atom wmAddState = XInternAtom(display, "_NET_WM_STATE_ADD", false);
      const Atom wmVMaximizedState = XInternAtom(display, "_NET_WM_STATE_MAXIMIZED_VERT", false);
      const Atom wmHMaximizedState = XInternAtom(display, "_NET_WM_STATE_MAXIMIZED_HORZ", false);

      XEvent xev;
      memset(&xev, 0, sizeof(xev));
      xev.xclient.type = ClientMessage;
      xev.xclient.display = display;
      xev.xclient.window = wid;
      xev.xclient.message_type = wmState;
      xev.xclient.format = 32;
      xev.xclient.data.l[0] = state;//xev.xclient.data.l[0] = wmAddState;
      xev.xclient.data.l[1] = wmVMaximizedState;
      xev.xclient.data.l[2] = wmHMaximizedState;
      xev.xclient.data.l[3] = 1;

      XSendEvent(display,
                 QX11Info::appRootWindow(screen),
                 false,
                 SubstructureRedirectMask | SubstructureNotifyMask,
                 &xev);
      XFlush(display);
  }
  ```
  6. 设置/取消窗口最小化
  ```
  void Utils::showMinimizedWindow(int wid, bool minimized)
  {
      const auto display = QX11Info::display();//Display *display = QX11Info::display();
      const auto screen = QX11Info::appScreen();

      const Atom wmState = XInternAtom(display, "_NET_WM_STATE", false);
      const Atom wmHiddenState = XInternAtom(display, "_NET_WM_STATE_HIDDEN", false);

      XEvent xev;
      memset(&xev, 0, sizeof(xev));
      xev.xclient.type = ClientMessage;
      xev.xclient.display = display;
      xev.xclient.window = wid;
      xev.xclient.message_type = wmState;
      xev.xclient.format = 32;
      if (minimized) {
          xev.xclient.data.l[0] = 1;//add/set property
      } else {
          xev.xclient.data.l[0] = 0;//remove/unset property
      }
      xev.xclient.data.l[1] = wmHiddenState;
      xev.xclient.data.l[2] = 0;
      xev.xclient.data.l[3] = 1;

      XSendEvent(display,
                 QX11Info::appRootWindow(screen),
                 false,
                 SubstructureRedirectMask | SubstructureNotifyMask,
                 &xev
                );
      XIconifyWindow(display, wid, screen);
      XFlush(display);
  }
  ```

## 参考文档

  [https://tronche.com/gui/x/xlib/window-information/XInternAtom.html](https://tronche.com/gui/x/xlib/window-information/XInternAtom.html)

  [https://linux.die.net/man/3/xinternatom](https://linux.die.net/man/3/xinternatom)

  [https://tronche.com/gui/x/xlib/event-handling/XSendEvent.html](https://tronche.com/gui/x/xlib/event-handling/XSendEvent.html)

  [https://tronche.com/gui/x/xlib/events/structures.html](https://tronche.com/gui/x/xlib/events/structures.html)

  [https://tronche.com/gui/x/xlib/events/client-communication/client-message.html#XClientMessageEvent](https://tronche.com/gui/x/xlib/events/client-communication/client-message.html#XClientMessageEvent)

  [https://tronche.com/gui/x/xlib/window-information/properties-and-atoms.html](https://tronche.com/gui/x/xlib/window-information/properties-and-atoms.html)
