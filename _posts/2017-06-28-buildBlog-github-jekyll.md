---
layout: post
title: github.io+jekyll搭建博客
categories: Blog
description: 参考教程搭建博客
keywords: github.io,jekyll,搭建博客
---

## gitHub.io+jekyll搭建静态博客

### 环境准备
> macos10.12(Linux类似)
> rvm
> ruby 2.3.0
> jekyll 3.2.1

本文基于macOs的,Linux系统类似

- - -

### 一.安装rvm
因为国内网络原因,rvm安装需要翻墙才能访问.至于翻墙,自行想办法了.
install rvm 参考地址:[https://rvm.io/rvm/install](https://rvm.io/rvm/install)进行安装
中间可能需要等待有点长时间,时间视网络情况,你懂的;
安装成功输入 rvm -v 查看

```bash
rvm -v
rvm 1.29.2 (latest) by Michal Papis, Piotr Kuczynski, Wayne E. Seguin [https://rvm.io/]

```
---
### 二.安装ruby
如果你成功安装rvm了.安装ruby就轻松了.参考
[https://ruby-china.org/wiki/install_ruby_guide](https://ruby-china.org/wiki/install_ruby_guide)
,soeasy
``` bash
rvm install 2.3.0
# 设置默认使用的 ruby 版本
rvm use 2.3.0 --default
# 安装 bundler
gem install bundler

```
---
### 三.安装jekyll
克隆jekyll模板
git clone https://github.com/mzlogin/mzlogin.github.io.git 到本地目录,
进入刚刚克隆的目录
``` bash
# 进入主题目录
cd mzlogin.github.io
# 安装 jekyll 等
bundle install
# 启动
jekyll -H 0.0.0.0 -P 4444

```
打开浏览器可看到,删除作者自己写的_posts中的内容,替换成自己的博客markdown文件,根据博客主题作者提示改配置和文件,替换为自己的即可;


- - -


### 四.推送到自己gitHub上
首先在 Github 上创建一个自己用户名的github.io，如: 用户名.github.io，之后讲刚刚修改的主题目录下的 .git 删除 ,git init 初始化,设置 git remote set-url 指向自己的github.io,最后推送到github上完成；Github 本身也是使用 jekyll 进行生成，所以会自动识别并生成博客；最后访问 http://用户名.github.io 即可.

---

### 参考
本搭建教程参考
[https://mritd.me/2016/10/09/jekyll-create-a-static-blog/](https://mritd.me/2016/10/09/jekyll-create-a-static-blog/)


