---
title: Hexo的使用技巧
date: 2018-03-17 14:47:22
categories:
- Hexo
tags:
- Hexo
- 博客部署
---

---
# 添加分类和标签
---

参考博客：

[Hexo使用攻略-添加分类及标签](https://linlif.github.io/2017/05/27/Hexo%E4%BD%BF%E7%94%A8%E6%94%BB%E7%95%A5-%E6%B7%BB%E5%8A%A0%E5%88%86%E7%B1%BB%E5%8F%8A%E6%A0%87%E7%AD%BE/)

---
# 主页显示文章摘要
---

针对next主题进行设置

- 进入hexo博客项目的themes/next目录
- 用文本编辑器打开_config.yml文件

```
# Automatically Excerpt. Not recommand.
# Please use <!-- more --> in the post to control excerpt accurately.
auto_excerpt:
  enable: false
  length: 150
```

---
# 修改博客基本信息
---

修改`_config`文件

```
title: Hong的博客
subtitle:
description: 一个热爱生活的程序猿
author: penghongzhan
language:
timezone:
```