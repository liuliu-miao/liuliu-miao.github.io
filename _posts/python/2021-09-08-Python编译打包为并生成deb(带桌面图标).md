---
layout: post
title: Python编译打包为并生成deb(带桌面图标)
categories: [Python]
description:  Python编译打包为并生成deb(带桌面图标)
keywords: python,编译,complie,打包,packageUbuntu,deb
---

# Python编译打包为并生成deb(带桌面图标)

## 环境
```
Ubuntu20.04
Python3.8

# 项目未打包前目录结构
tree -L 2
❯ tree -L 2
.
├── application.png
├── debbuild
│   ├── DEBIAN
│   └── usr
├── exec_cmd.py
├── main.py
├── __pycache__
│   └── main.cpython-38.pyc
├── random_wallpaper.py


```

## 1 安装pyinstaller
```
pip3 install pyinstaller
```
## 2 将main编译为二进制可执行文件
```
pyinstaller -F main.py  打包ubuntu下的可执行文件
pyinstaller -F -w main.py  不带控制台的打包
# 带有图标的
pyinstaller -F -w -i application.png main.py  打包指定ubuntu下的可执行文件的图标打包

# 编译完成后目录：
❯ tree -L 2
.
├── application.png
├── build
│   └── main
├── dist
│   └── main
├── exec_cmd.py
├── main.py
├── main.spec
├── __pycache__
│   └── main.cpython-38.pyc
├── random_wallpaper.py
└── venv
    ├── bin
    ├── include
    ├── lib
    ├── lib64 -> lib
    └── pyvenv.cfg

```
**dist/main为当前系统可执行文件**
## 3 将可执行文件打包为deb并且设置桌面图标（Ubuntu环境）
```
mkdir debbuild && cd debbuild
#以 debbuild 根目录，创建如下目录和文件
DEBIAN,usr/lib/RandomWallPaper ,usr/lib/share/applications ,usr/lib/share/icons
❯ tree
.
├── DEBIAN
│   └── control
└── usr
    ├── lib
    │   └── RandomWallpaper
    │       └── main
    └── share
        ├── applications
        │   └── RandomWallpaper.desktop
        └── icons
            └── RandomWallpaper.png

```
**usr/lib/RandomWallPaper/main 这个文件就是可执行二进制文件，来自于dist/main ; RandomWallpaper.png 图标文件为自定义**
### 3.1 control
```
Package: RandomWallpaper
Version: 1.0.0
Architecture: amd64
Maintainer: lius
Description: random set desktop background img and lock img
```
### 3.2 RandomWallpaper.desktop
```
[Desktop Entry]
Name=RandomWallpaper
Comment=An example
Exec=/usr/lib/RandomWallpaper/main
Icon=/usr/share/icons/RandomWallpaper.png
Terminal=false
Type=Application
X-Ubuntu-Touch=true
Categories=Development
```

## 4 执行打包
```
cd ..
#---------------#
❯ tree -L 2
.
├── application.ico
├── application.png
├── build
│   └── main
├── debbuild
│   ├── DEBIAN
│   └── usr
├── dist
│   └── main
├── exec_cmd.py
├── main.py
├── main.spec
├── pk.sh
├── __pycache__
│   └── main.cpython-38.pyc
├── randomWallpaper1.0.0.amd64.deb
├── random_wallpaper.py
└── venv
    ├── bin
    ├── include
    ├── lib
    ├── lib64 -> lib
    └── pyvenv.cfg
#----------------------#

sudo dpkg -b debbuild  randomWallpaper1.0.0.amd64.deb
```

## 5 验证安装
```
sudo dpkg -i randomWallpaper1.0.0.amd64.deb

```

## 备注
翻阅资料发现pyinstaller在某个版本后，不再支持交叉编译，因为兼容性太差。需要对应操作系统的可执行文件，需要到相应的操作系统下进行编译。 

**（But Golang 可以，所以后期有类似的应用程序，尽量都会使用go进行编写）**