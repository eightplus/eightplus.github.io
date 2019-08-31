Welcome to GitHub Pages
===========

- 创建分支：git checkout -b jekyll-1.0
- 切换分支：git checkout master
- 查看分支：git branch
- 合并jekyll-1.0分支到master分支：git merge jekyll-1.0
- 删除分支：git branch -d jekyll-1.0
- 克隆分支：git clone -b jekyll-1.0 https://github.com/eightplus/eightplus.github.io.git


Test
============
```
npm install
npm install hexo --save
npm install hexo-cli -g
npm install hexo-deployer-git
hexo clean    #清空
hexo generate #生成静态网页
hexo deploy   #部署
hexo server  #本地预览，默认端口号是4000，可指定断开，如hexo server -p 5000


百度验证:https://ziyuan.baidu.com/https/index
```


Create Page
============

$ cd eightplus.github.io
$ hexo new page categories
$ vim source/categories/index.md
```
---
title: 分类
date: 2018-07-30 18:46:20
type: "categories"
---
```

sitemap
============

- 确认博客是否被收录，百度上输入 site:eightplus.github.io

$ cd eightplus.github.io
$ npm install hexo-generator-sitemap --save
$ npm install hexo-generator-baidu-sitemap --save
$ npm install hexo-generator-seo-friendly-sitemap --save
$ vim _config.yml
```
sitemap:
  path: uploads/sitemap.xml
baidusitemap:
  path: uploads/baidusitemap.xml
```
- 在本地访问 http://127.0.0.1:4000/sitemap.xml 和 http://127.0.0.1:4000/baidusitemap.xml 查看效果


Search
============

$ npm install hexo-generator-search --save
$ npm install hexo-generator-searchdb --save
$ vim _config.yml
```
search:
     path: search.xml
     field: post
     format: html
     limit: 10000
```
$ vim themes/next/_config.yml
```
local_search:
  enable: true
```


字数统计
============

$ npm install hexo-wordcount --save
$ vim themes/next/_config.yml
```
post_wordcount:
  item_text: true
  wordcount: true
```


RSS
============

$ npm install hexo-generator-feed --save
$ vim _config.yml
```
feed:
  type: atom
  path: atom.xml
  limit: 20
  hub:
  content:
```


Hexo源文件
============

- _config.yml站点的配置文件，需要拷贝
- themes/主题文件夹，需要拷贝
- source博客文章的.md文件，需要拷贝
- scaffolds/文章的模板，需要拷贝
- package.json安装包的名称，需要拷贝
- .gitignore限定在push时哪些文件可以忽略，需要拷贝
- .git/主题和站点都有，标志这是一个git项目，不需要拷贝
- node_modules/是安装包的目录，在执行npm install的时候会重新生成，不需要拷贝
- public是hexo g生成的静态网页，不需要拷贝
- .deploy_git同上，hexo g也会生成，不需要拷贝
- db.json文件，不需要拷贝


Create md file
============

$ npm install hexo-asset-image --save

$ hexo n "2018-07-31-test" (在/source/_posts文件夹内除了生成2018-07-31-test.md文件外，还有一个同名的文件夹存放图片)

- markdown规则
 https://studygolang.com/markdown

- Markdown编辑器atom

    - `$ atom`

    - 预览实时渲染(Ctrl + Shift + M)

Git
============

- 下载源码
    - git clone https://github.com/eightplus/eightplus.github.io.git
- 查看某次commit提交的内容
    - git show commit_id
    - (git show 5588d251dd77ade98e0d5a71a97587b7cf681535)
- 打包某次提交的所有文件
    - git diff --name-only last_commit_id need_commit_id | xargs tar -jcvf need_commit_id.tar.bz2
    - (git diff --name-only 9d169c2681e6c19667cd28dd60f6f5458da211de ba8ce7176ef65dc4167c4524f0eb3ab5bfc1cfb6 | xargs tar -jcvf ba8ce7176ef65dc4167c4524f0eb3ab5bfc1cfb6.tar.bz2)
    - (例子为打包第二次提交的所有文件，其中9d169c2681e6c19667cd28dd60f6f5458da211de是第二次提交之前的commit_id，即第一次提交的commit_id, 而ba8ce7176ef65dc4167c4524f0eb3ab5bfc1cfb6是第二次次提交的commit_id, 则ba8ce7176ef65dc4167c4524f0eb3ab5bfc1cfb6.tar.bz2是第二次提交所有文件的打包)

git恢复(可以使用git reflog show或git log -g命令来看到所有的操作日志)：
- 通过git log -g命令来找到需要恢复的信息对应的commitid，可以通过提交的时间和日期来辨别,找到执行reset --hard之前的那个commit对应的commitid
- 通过git branch recover_branch commitid 来建立一个新的分支，就把到commitid为止的代码、各种提交记录等信息都恢复到了recover_branch分支上了

