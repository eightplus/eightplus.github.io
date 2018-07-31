---
title:      "使用Jekyll + Github搭建博客"
date:       2018-06-27 15:33:17 +0800
categories: 工具
tags:
    - Jekyll
---

> 一直想把自己学习linux的过程记录下来，又不想总是依托那些现成的博客网站，在WordPress和Github中，我选择了Github，原因只有一个，穷，哈哈哈。<br><br>
> 这里先简要接收博客搭建的初期准备工作！

---

## Github

- github帐号注册和创建页面仓库

    - 注册地址：[https://github.com/](https://github.com/)

    - 仓库的名字需要和你的账号对应，格式：yourname.github.io，比如我这里演示的页面仓库名为eightplus.github.io

- 生成ssh密钥，并将密钥添加到github上

    - 生成密钥：`ssh-keygen -t rsa -C` 注册帐号时的邮箱地址"

    - 密钥生成后，在 ~/.ssh目录下会生成一些文件，包括id_rsa和id_rsa.pub

    - 打开id_rsa.pub，选中所有内容复制，网页进入 [https://github.com/settings/ssh](https://github.com/settings/ssh) ，Add SSH key，粘贴之前复制的内容。

- Github安装和帐号配置

    - `sudo apt-get install git`

    - `git config --global user.name "yourname"`

    - `git config --global user.email "注册帐号时的邮箱地址"`


## 安装ruby环境

`sudo apt-get install ruby-all-dev`
![](2018-06-27-welcome-to-jekyll/01.png)
我的系统为Ubuntu 16.04，使用的软件源里面存在ruby2.3-dev，则此种方式会将ruby2.3-dev一并安装，当然，上述操作也可以替换成安装ruby2.3-dev，(`sudo apt-get install ruby2.3-dev`)

安装完成后，在终端中输入`ruby -v`，查看版本信息，如下图：
![](2018-06-27-welcome-to-jekyll/02.png)

完成ruby环境后，在终端中输入`gem -v`，出现如下结果，则说明ruby环境已经完全配置成功了，如果出现了报错信息，可能需要安装nodejs (sudo apt-get install nodejs)。
![](2018-06-27-welcome-to-jekyll/03.png)


## 安装JeKyll环境

- 先安装依赖包bundler：
    - `sudo gem install bundler`
- 再安装安装jekyll：
    - `sudo gem install jekyll`
- 安装完成后，在终端中输入`jekyll --version`，出现如下结果，则说明安装成功
![](2018-06-27-welcome-to-jekyll/04.png)


## 工程创建

[是否迫不及待的想看下第一个博客的具体内容 👉 ](#build)

在你打算存放工程代码的目录下打开一个终端后使用jekyll创建一个项目，这里我的目录为：~/work/git/：
`jekyll new blog`
操作之后会生成很多文件/文件夹，详细说明如下：
![](2018-06-27-welcome-to-jekyll/05.png)
- _config.yml：Jekyll配置文件，存储配置数据
- _drafts：草稿目录，可手动创建
- _includes：包含一些模板，可以重复利用
- _layouts：存放页面模板的地方
- _posts：存放文章的目录，文章格式为 mardown 格式（year-month-title.markdown）或.md，文件名确定了发表的日期和标记语言
- _data：存放yaml格式的数据文件
- _site：使用Jekyll编译后的静态站点将存放于这个目录下，即jekyll生成的网站会放在该文件夹下，该目录不需要push到github，可在.gitignore文件中加入这个目录
- index.html：该文件带有 yaml 头信息，大概如下：
```
---
layout: post
title:  "Welcome to Jekyll!"
date:   2018-06-27 15:33:17 +0800
categories: jekyll update
---
```
上述操作会生成个默认文章，位于_posts目录下，名字类似为：`2018-06-27-welcome-to-jekyll.markdown`

可以复制2018-06-27-welcome-to-jekyll.markdown后进行修改来进行新的博客编写，这里推荐使用 [git的atom编辑器][1] 来编辑.markdown文件，可以在atom官网进行deb包的下载，新页面生成和编辑完成后，重启jekyll内置服务器（终端执行：jekyll serve），打开或刷新页面：[http://localhost:4000](http://localhost:4000)，这样就可以在页面看到自己添加的博文了。
![](2018-06-27-welcome-to-jekyll/06.png)
![](2018-06-27-welcome-to-jekyll/07.png)


## Git同步

将前面创建的仓库克隆到本地，然后将blog目录中生成的文件复制到github项目目录下，我这里项目名为 `eightplus.github.io`。

- `git clone https://github.com/yourname/yourname.github.io.git`
- `git add .`
- `git commit -m "init"`
- `git push -u origin master`

至此，在浏览器中输入https://yourname.github.io，比如：https://eightplus.github.io/，即可看到下图，博客搭建完成
<p id = "build"></p>
![博客效果图](2018-06-27-welcome-to-jekyll/08.png)
*网页浏览效果图*

[1]: https://atom.io/
