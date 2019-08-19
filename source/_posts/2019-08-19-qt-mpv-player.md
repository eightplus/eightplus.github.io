---
title:      "使用mpv编写自己的播放器"
date:       2019-08-19 15:32:08
author:     "lixiang"
categories: Linux Qt 开发
tags:
    - Qt
---

> MPV 是一个基于 MPlayer 和 MPlayer2 的多平台开源播放器，其在Linux上拥有广泛的输出设备支持，内置ffmpeg解码器，支持绝大部分的视频和音频格式，支持本地播放和网络播放，支持ass特效字幕，GPU 解码能力十分出色。虽然MPV功能强大，但默认情况下，MPV无GUI图形界面，用户需要通过命令行或者手动修改其配置文件达到配置MPV的目的，这样就给普通用户带来了诸多不便。为此，本文将介绍如何在Ubuntu系统上使用Qt编写带UI图形的MPV播放器，使用户对快速定制具有图形界面的MPV播放器有一个大致的了解。<br>

---

## 示例源码
- [mympvplayer](https://github.com/eightplus/examples/tree/master/code/Qt/mympvplayer)

## 开发前准备工作

- Ubuntu上MPV的安装：
```
$ sudo apt-get update
$ sudo install mpv
```

- 编程开发依赖包安装：
```
$ sudo install qtbase5-dev qt5-qmake qtscript5-dev qttools5-dev-tools
```

## 要点分析

  为了实现一个既可以在Qt程序中控制MPV，又可以让Qt程序得到MPV的输出信息的播放器，这里重点介绍Qt的QProcess类。QProcess类可用来调用外部程序，并与外部程序进行通信。其把外部程序的进程当作一个有序的I/O设备，通过对通过I/O设备的读写来完成进程间的通信，即：write()函数实现对进程标准输入的写操作，通过read()，readLine()和getChar()函数实现对标准输出的读操作。在正常渠道模式下，QProcess的无名管道stdinChannelpipe，stdoutChannelpipe和stderrChannelpipe分别与标准输入、标准输出和标准容错进行绑定，实现与外部程序的通信；而在融合模式下，没有容错管道，此时，标准容错端和标准输出端将共同挂接到子进程的stdoutChannelpipe的写端来实现内外进程的通信，即标准输出和标准容错绑定到同一个管道的写端。本文介绍通过QProcess类调用MPV，并设置一系列播放参数，如视频驱动、音频驱动、软解/硬解、缓存等。至于Qt图形和MPV视频窗口的关联，则是使用“--wid widget->winId()”进行绑定，通过winId()可以获得一个数字，其中widget是一个QWidget对象，这样将界面上一个窗口的句柄给了MPV，即视频输出定位到了widget窗体部件中（wid为MPV指定了输入窗口，-wid参数只在X11、directX和OpenGL中适用）。本文推荐使用融合模式，代码为：setProcessChannelMode(QProcess::MergedChannels)。

- 配置介绍

  查看MPV的帮助信息可在终端执行"mpv --help"， 查看MPV可配置信息可在终端执行"mpv --list-options"，查看快捷键列表可在终端执行"mpv ---input-keylist"。MPV参数调用需要加"--"，如果参数是使用配置文件中的参数，则配置文件中无需在参数前加"--"。MPV的配置文件目录为：~/.config/mpv/，本文介绍的播放器定制将不使用配置文件，这里只简要介绍下mpv.conf和input.conf这两个配置文件的格式，mpv.conf 是主配置文件，里面包含一些基本的配置，input.conf 按键配置文件，包含播放过程中一些操作快捷键的设置。

  mpv.conf
  ```
  # Disable the On Screen Controller (OSC).
  osc=no
  # Keep the player window on top of all other windows.
  ontop=yes
  # Enable hardware decoding if available. Often, this does not work with all
  # video outputs, but should work well with default settings on most systems.
  # If performance or energy usage is an issue, forcing the vdpau or vaapi VOs
  # may or may not help.
  hwdec=auto
  ```

  input.conf
  ```
  # Mouse wheels, touchpad or other input devices that have axes
  # if the input devices supports precise scrolling it will also scale the
  # numeric value accordingly
  WHEEL_UP      seek 10
  WHEEL_DOWN    seek -10
  WHEEL_LEFT    add volume -2
  WHEEL_RIGHT   add volume 2
  ## Seek units are in seconds, but note that these are limited by keyframes
  RIGHT seek  5
  LEFT  seek -5
  UP    seek  60
  DOWN seek -60
  ```

  下面详细介绍几个比较重要的配置项
    - quiet

    这个参数会阻止状态行信息的显示，即使得控制台消息尽量少输出。使用Qt嵌入MPV时，需要使用noquiet而不是quiet，否则Qt程序无法获得MPV的状态信息，致使Qt程序无法将MPV的状态准确的展示给用户，如播放进度、出错信息等。当然，如果你的机器性能差，那还是建议你直接使用mpv，且参数使用quiet，而不是像本文介绍的这样对MPV进行UI封装。MPV使用noquiet的格式为：mpv --no-quiet。

    - config

    可让Qt程序将一些基本的配置通过从MPV命令获取各参数支持可选值，并设置一个默认值，且可通过图形展示给用户去选择。所以此处使用no-config，即不从MPV的配置文件读取参数。mpv使用no-config的格式为：mpv --no-config。

    - input-file

    这里将不使用MPV的input.conf配置文件，而是通过标准输入stdin给MPV发送命令，命令后面带上换行"\n"写入stdin即可。另外，在直接使用MPV的过程中，--no-input-default-bindings将使得MPV无法响应按键的事件，而--input-default-bindings参数默认为yes，则可以让MPV响应按键事件。MPV使用input-file的格式为：mpv --input-file=/dev/stdin。

    - term-status-msg

    该参数可以让MPV输出一些视频信息，可以通过 --term-status-msg 参数给它一个输出格式，如："--term-status-msg=STATUS: ${=time-pos} / ${=duration:${=length:0}} P: ${=pause} B: ${=paused-for-cache} I: ${=core-idle} VB: ${=video-bitrate:0} AB: ${=audio-bitrate:0}"

    - vo

    通过命令“mpv --vo help”可查看MPV支持的视频驱动列表，Qt图形程序可以将列表展示出来供用户选择，并将选择的vo加入MPV的参数列表中，加入方式为：mpv --vo xxx，如：mpv --vo=xv。

    - ao

    通过命令“mpv --ao help”可查看mpv支持的音频驱动列表，Qt图形程序可以将列表展示出来供用户选择，并将选择的ao加入mpv的参数列表中，加入方式为：mpv --ao xxx，如：mpv --ao=pulse。

    - hwdec

    hwdec为硬件解码配置，其可用配置列表和GPU有关，这里暂分析其中5种配置：no（软解），auto（自动尝试使用第一种可用的硬解方式），vdpau（用于vdpau和opengl的显示输出，即此时需要保证vo参数为gpu或者vdpau），vaapi（用于vaapi和opengl的视频输出，即此时需要保证vo参数为gpu或者vdapi，仅支持Intel GPU）和vaapi-copy（将视频拷贝回系统内存中，仅支持Intel GPU）。参数使用格式为：--hwdec=vaapi-copy。
    hwdec具体参数见文档：https://mpv.io/manual/stable/。

    - vd-lavc-threads

    硬件解码线程数目，仅适用于MPEG-1/2和H.264，取值范围为0 - any，默认为0。使用格式如下：--vd-lavc-threads=4。


- MPV支持的音/视频、字幕和播放列表格式的大部分列表

  MPV支持的视频格式：
  avi 、vfw、divx、mpg、mpeg、m1v、m2v、mpv、dv、3gp、mov、mp4、m4v、mqv、dat、vcd、ogg、ogm、ogv、ogx、asf、wmv、bin、iso、vob、mkv、nsv、ram、flv、rm、swf、ts、rmvb、dvr-ms、m2t、m2ts、mts、rec、wtv、f4v、hdmov、webm、vp8、bik、smk、m4b、wtv、part

  MPV支持的音频格式：
  mp3、ogg、oga、wav、wma、aac、ac3、dts、ra、ape、flac、thd、mka、m4a、opus

  MPV支持的字幕格式：
  srt、sub、ssa、ass、idx、txt、smi、rt、utf、aqt、vtt

  MPV支持的列表格式：
  m3u、m3u8、pls、xspf

- Qt对MPV的控制

  前面提及过Qt的QProcess类，这里描述下该类是如果设置MPV的参数和启动MPV的，即如何使用给标准输入控制MPV。MPV自动从标准输入中读取信息并执行，该类指令信息都需要以“\n”结尾。管道的读端描述符stdinChannelpipe[0]复制给了标准输入，即标准输入的描述符也为stdinChannelpipe[0]，隐藏按照标准输入的描述符去读信息就是到stdinChannelpipe所对应的管道中读取信息。QProcess的start()函数将开启进程，第一个参数即为mpv二进制，第二个参数为给mpv的参数列表，执行start()函数后，将完成内核中管道以及通信环境的建立。QProcess的成员函数write()向stdinChannelpipe[1]端写入信息，比如想让mpv退出，则通过write函数写入“quit \n”即可（write()函数将向stdinChannelpipe[1]端写入命令）。参考代码如下：
  ```
  QString mpvCmd = "/usr/bin/mpv";		
  QStringList args;
  args << "-- no-quiet";
  args << "--wid=100663330";
  args << xxx.avi;
  process->start(mpvCmd, args);//启动播放
  waitForStarted();
  QString cmd = "set pause yes";
  process->write(cmd.toLocal8Bit() + "\n");//设置暂停
  ```

- Qt获取MPV的信息

  使用QProcess的融合模式，从标准输出和标准容错得到mpv 的信息。绑定QProcess的信号readyReadStandardOutput，当该信号出发时，再通过QProcess的readAllStandardOutput()函数获取信息（当然，这里也可以while循环来判断是否可以读取一行数据：while(process->canReadLine())，如果可以，则读取一行：QString line(process->readLine());），readAllStandardOutput获取的数据类型为QByteArray，对数据进行解析时通过“\n” 和“\r” 切分后一行一行解析，通过对每行的QByteArray数据进行详细解析可以得出MPV的状态信息，并将一些状态在Qt的图形程序上动态展示处理，比如MPV播放成功，则根据播放成功的信息将Qt图形的播放按钮设置为可暂停状态，此时点击该按钮给MPV发送的指令应该是暂停而非启动。另外，可以绑定QProcess的信号finished(int, QProcess::ExitStatus)，当该信号被触发时，通过判断QProcess的bytesAvailable()值，如果>0，则通过QProcess的readAllStandardOutput()函数获取信息。

  获取MPV的信息进行解析的难点是对繁杂冗余的信息进行过滤，针对关键字定位信息表示的意思。这里可以使用Qt的对获取的QRegExp来进行匹配。

  下述正则表达式可用于分析出正在启动中：
  ```
  QRegExp rx_playing;
  rx_playing.setPattern("^Playing:.*|^\\[ytdl_hook\\].*");
  ```
  下述正则表达式可可解析出暂停、缓存中等各种状态信息：
  ````
  QRegExp rx_av;
  rx_av.setPattern("^STATUS: ([0-9\\.-]+) / ([0-9\\.-]+) P: (yes|no) B: (yes|no) I: (yes|no) VB: ([0-9\\.-]+) AB: ([0-9\\.-]+)");
  ```

## 参考文档

[麒麟影音](https://github.com/ukui/kylin-video/)

[mpv配置](https://mpv.io/manual/master/#configuration-files	)

[mpv主页](https://mpv.io/manual/stable/)

[mpv.conf](https://github.com/mpv-player/mpv/blob/master/etc/mpv.conf)

[input.conf](https://github.com/mpv-player/mpv/blob/master/etc/input.conf)
