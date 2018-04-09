---
layout: post
title: 本地jekyll-server启动证书错误问题
categories: [Blog]
description: jekyll-server启动证书错误
keywords: jekyll-server, 启动证书
---

--- 

### 问题描述:
> 本地启动jekyll server 博客时显示如下错误:
``` bash
Liquid Exception: SSL_connect returned=1 errno=0 state=error: certificate verify failed in /_layouts/page.html
jekyll 3.6.2 | Error:  SSL_connect returned=1 errno=0 state=error: certificate verify failed    
```

### 解决方案:
> 1.下载一个证书 名称为: cacert.pem

> 2.将证书路径设置到系统变量中 

> window下 可使用命令 set SSL_CERT_FILE=d:/XmacZone/down/cacert.pem

> linux 或macos 编制.brash 或.zshrc 

> 最后一行添加 export  SSL_CERT_FILE=/Users/XmacZone/down/cacert.pem   ;执行命令 source ~/.zshrc 即可.