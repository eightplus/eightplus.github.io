Welcome to GitHub Pages
===========

- Jekyll官方网站: https://jekyllrb.com/

- Jekyll官方中文网站: http://jekyllcn.com/

- 博客的模板使用的是Hux大神的模板，在此表示感谢，他的博客地址为: https://huangxuan.me/


Reference
===========
https://github.com/Huxpro/huxpro.github.io <br><br>
https://github.com/BlackrockDigital/startbootstrap-clean-blog-jekyll <br><br>
https://github.com/tsai1993/tsai1993.github.io <br><br>
https://tsai1993.github.io/ <br><br>
https://github.com/i2000s/i2000s.github.io <br><br>
https://www.qixiaodong.tk/ <br><br>
https://github.com/nicolewhite/nicolewhite.github.io <br><br>
https://nicolewhite.github.io/ <br><br>
https://segmentfault.com/a/1190000007709682 <br><br>
https://segmentfault.com/markdown <br><br>
https://www.appinn.com/markdown/ <br><br>
http://jekyllcn.com/docs/configuration/ <br><br>
https://github.com/tianshan/tianshan.github.io <br><br>
https://github.com/galian123/galian123.github.io <br><br>
https://www.ezlippi.com/index.html <br><br>
https://github.com/EZLippi/EZLippi.github.io <br><br>
https://streamelody.github.io/ <br><br>
https://github.com/mmoaay/mmoaay.github.io <br><br>
http://mmoaay.github.io/ <br><br>
https://mmoaay.gitbooks.io/boost-asio-cpp-network-programming-chinese/ <br><br>
https://github.com/JerryC8080/BlueSun_V3 <br><br>
http://huang-jerryc.com/ <br><br>
https://github.com/JerryC8080/understand-tcp-udp <br><br>
https://jerryc8080.gitbooks.io/understand-tcp-and-udp/ <br><br>
https://legacy.gitbook.com/@jerryc8080 <br><br>
https://legacy.gitbook.com/@tinylab <br><br>
http://www.chengweiyang.cn/gitbook/gitbook.com/newbook.html <br><br>
https://www.eyrefree.org/ <br><br>
https://vevlins.github.io/ <br><br>
https://waynechu.cn/ <br><br>
https://liuchi.coding.me/ <br><br>
https://github.com/LooEv <br><br>
http://threehao.com/ <br><br>
http://litten.me/ <br><br>
http://zhufanjia.com/ <br><br>
https://wblearn.github.io/ <br><br>

Test
============
```
rm -rf .sass-cache/ _site/
jekyll serve
百度验证:https://ziyuan.baidu.com/https/index
```

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
