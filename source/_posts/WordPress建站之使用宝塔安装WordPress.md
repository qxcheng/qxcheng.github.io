---
categories:
- - WordPress建站
date: 2024-05-27 11:55:14
description: 在域名的ICP备案完成后，我们可以开始wordpress的搭建工作了，这里借助宝塔面板工具来协助完成。
id: '278'
tags:
- wordpress
title: WordPress建站之使用宝塔安装WordPress
---


在域名的ICP备案完成后，我们可以开始wordpress的搭建工作了，这里借助宝塔面板工具来协助完成。

## 1.域名解析

首先我们需要将域名解析到服务器上，打开阿里云的域名解析页面，点击解析设置： ![](https://img.bplan.top/2024-05-19.webp) 点击添加记录，如图填写完善后，点击确认，可以再添加一条值为@的主机记录：![](https://img.bplan.top/2024-12-20.webp)

## 2.安装宝塔

首先我们在宝塔官网，根据服务器的系统类型复制相应的安装脚本： ![](https://img.bplan.top/2024-05-21.webp) 然后ssh到服务器上粘贴安装脚本并回车执行： ![](https://img.bplan.top/2024-05-22.webp) 继续输入y并回车： ![](https://img.bplan.top/2024-05-23.webp) 继续输入yes并回车： ![](https://img.bplan.top/2024-05-24.webp) 安装完成后会出现登录地址、用户名和密码，请妥善保存，此时我们还需要到阿里云控制台为安全组开放16525端口，进入安全组管理规则页面： ![](https://img.bplan.top/2024-05-25.webp) 手动添加一条入方向记录： ![](https://img.bplan.top/2024-05-26.webp) 添加好后我们便可以使用账号密码登录宝塔的管理面板，进入面板，安装好推荐的套件后就结束了宝塔的安装流程。 ![](https://img.bplan.top/2024-05-27.webp)

## 3.安装wordpress

首先我们在宝塔面板中添加站点：![](https://img.bplan.top/2024-12-28.webp) 然后到wordpress官网下载wordpress中文压缩包： ![](https://img.bplan.top/2024-05-29.webp) 接着将压缩包上传到如图所示的网站根目录下面：![](https://img.bplan.top/2024-12-30.webp) 上传完成后，点击压缩包右侧的解压按钮解压文件： ![](https://img.bplan.top/2024-05-31.webp) 将解压后得到的wordpress文件夹中的所有文件剪切到网站根目录：![](https://img.bplan.top/2024-12-32.webp) ![](https://img.bplan.top/2024-12-33.webp) 现在我们登录wordpress的后台管理页面： ![](https://img.bplan.top/2024-05-34.webp) 填写添加站点时生成的数据库账号密码信息后点击提交： ![](https://img.bplan.top/2024-05-35.webp) 接着点击运行安装程序： ![](https://img.bplan.top/2024-05-36.webp) 完善信息后点击安装，最后会提示安装成功，使用用户名密码登录即可进入到wordpress的后台管理页面，至此安装流程结束。 ![](https://img.bplan.top/2024-05-37.webp)

## 4.结语

wordpress安装成功后，我们便可以方便的在后台进行配置主题、发布文章等操作了。