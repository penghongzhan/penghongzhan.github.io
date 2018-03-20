---
title: 通过GitHub和hexo部署自己的博客
date: 2018-03-17 11:47:22
categories:
- Hexo
tags:
- Hexo
- github
- 博客部署
---

---
# 参照博客
---

[搭建个人博客-hexo+github详细完整步骤--简书](https://www.jianshu.com/p/189fd945f38f)

[搭建hexo博客并简单的实现多终端同步](https://righere.github.io/2016/10/10/install-hexo/)

[如何解决github+Hexo的博客多终端同步问题](http://blog.csdn.net/Monkey_LZL/article/details/60870891)

---
# 注意问题
---

- 最好在git-bash中执行命令，不要在window的控制台中执行
- 创建GitHub的远程仓库的时候，名字需要使用自己的GitHub用户名.github.io
- hexo的_config配置文件中，需要修改url为`https://xxx.github.io`