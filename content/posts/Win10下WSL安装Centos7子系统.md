---
title: "Win10下WSL安装Centos7子系统"
date: 2023-08-09T22:01:31+08:00
draft: false
---

# 1、控制面板设置

控制面板-程序和功能-启用或关闭Windows功能，勾选如下选项并重启

![](https://i.imgur.com/Y1WeXt9.png)

# 2、下载CentOS系统

下载地址：[https://github.com/wsldl-pg/CentWSL/releases](https://github.com/wsldl-pg/CentWSL/releases)

选择自己对应的系统版本就好，我选择的是如下版本

![](https://i.imgur.com/eWBBckm.png)

# 3、解压

1、以管理员身份运行CentOS.exe，过程有点慢，要稍等一下，执行完窗口会自动关闭
![](https://i.imgur.com/nSxLoaZ.png)

2、目录下会自动创建两个文件夹rootfs和temp
![](https://i.imgur.com/6lnsRw3.png)
3、验证是否安装成功，cmd命令行输入 `wsl`，出现如下所示，说明安装成功，大功告成，一台Linux虚拟主机创建完成
![](https://i.imgur.com/Xbh1nKA.png)

# 4、扩展

资源管理器输入`\\wsl$`，出现如下页面，可以清晰查看该虚拟主机下所有系统文件

![](https://i.imgur.com/StjcbxA.png)


![](https://i.imgur.com/DgruaU9.png)
