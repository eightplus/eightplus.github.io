Welcome to GitHub Pages
===========

- Jekyll官方网站: https://jekyllrb.com/

- Jekyll官方中文网站: http://jekyllcn.com/

- 博客的模板使用的是Hux大神的模板，在此表示感谢，他的博客地址为: https://huangxuan.me/


Reference
===========
```
https://github.com/Huxpro/huxpro.github.io
https://github.com/BlackrockDigital/startbootstrap-clean-blog-jekyll
https://github.com/tsai1993/tsai1993.github.io
https://tsai1993.github.io/
https://github.com/i2000s/i2000s.github.io
https://www.qixiaodong.tk/
https://github.com/nicolewhite/nicolewhite.github.io
https://nicolewhite.github.io/
https://segmentfault.com/a/1190000007709682
https://www.appinn.com/markdown/
http://jekyllcn.com/docs/configuration/
https://github.com/tianshan/tianshan.github.io
https://github.com/galian123/galian123.github.io
https://www.ezlippi.com/index.html
https://github.com/EZLippi/EZLippi.github.io
```

Test
============

rm -rf .sass-cache/ _site/
jekyll serve


Git
============

- 下载源码
        * git clone https://github.com/eightplus/eightplus.github.io.git

- 查看某次commit提交的内容
        * git show commit_id
        * (git show 5588d251dd77ade98e0d5a71a97587b7cf681535)

- 打包某次提交的所有文件
        * git diff --name-only last_commit_id need_commit_id | xargs tar -jcvf need_commit_id.tar.bz2
        * (git diff --name-only 9d169c2681e6c19667cd28dd60f6f5458da211de ba8ce7176ef65dc4167c4524f0eb3ab5bfc1cfb6 | xargs tar -jcvf ba8ce7176ef65dc4167c4524f0eb3ab5bfc1cfb6.tar.bz2)
        * (例子为打包第二次提交的所有文件，其中9d169c2681e6c19667cd28dd60f6f5458da211de是第二次提交之前的commit_id，即第一次提交的commit_id, 而ba8ce7176ef65dc4167c4524f0eb3ab5bfc1cfb6是第二次次提交的commit_id, 则ba8ce7176ef65dc4167c4524f0eb3ab5bfc1cfb6.tar.bz2是第二次提交所有文件的打包)
