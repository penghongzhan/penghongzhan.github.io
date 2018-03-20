---
title: GitHub访问慢解决方案
categories:
- github
tags:
- github
- 博客部署
---

---
# GitHub访问慢
---

windows系统中：

编辑host文件，将GitHub的域名和实际的ip配置上。查询域名对应的ip通过下文章提到的网站进行查询。

## 编辑host文件

C:\Windows\System32\drivers\etc\hosts

## 查询域名对应的ip

https://www.ipaddress.com/ip-lookup/

## github对应的ip

### github.com

192.30.253.112
192.30.253.113

192.30.253.113 github.com

### github.global.ssl.fastly.net

151.101.45.194 github.global.ssl.fastly.net