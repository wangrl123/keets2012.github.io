---
title: mac下快速进入当前目录iterm2
date: 2017-08-28
categories: Utils
tags:
- mac
- utils
---
win环境下，有直接在文件浏览的地址上，直接输入cmd，即可打开cmd命令框。
笔者在macOS下，也想实现这样的功能，网上查了一下，可以成功实践。

### 1. 添加服务
```
git clone https://github.com/peterldowns/iterm2-finder-tools.git
```
进入 iterm2-finder-tools文件夹，运行iTerm.workflow。安装服务栏。

### 2. 使用服务
在工作文件夹上右键，弹出窗口中找到服务一栏，将鼠标放置其上，在弹出窗口中找到 Open iTerm一栏，单击即可。

### 3. 添加快捷键
服务->偏好设置->快捷键


![快捷键][shift]
[shift]:http://ovci9bs39.bkt.clouddn.com/finder.png  "快捷键设置"

笔者设置了control+command+L。大家可以根据自己的喜好进行设置快捷键。

![效果][real]
[real]:http://ovci9bs39.bkt.clouddn.com/real-pic.png  "实现展示"

---
参考：   
[ Mac在Finder中当前目录下打开iTerm2](http://blog.csdn.net/u014632633/article/details/72769832)


