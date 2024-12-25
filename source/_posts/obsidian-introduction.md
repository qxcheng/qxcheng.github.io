---
categories:
- - Obsidian
date: 2024-05-10 17:02:31
description: 'Obsidian入门：快速上手与基本配置'
id: '202'
tags:
- Obsidian
title: Obsidian入门：快速上手与基本配置
---


## 1.简介

一直以来使用过不少软件记录笔记，大致经历过印象笔记->为知笔记->typora->zimwiki->我来->Obsidian的迁移历程，目前已将Obsidian作为主力使用，我的主要诉求它都能满足：

1.  笔记离线存储，支持多端同步
2.  代码高亮显示，图片插入
3.  支持大纲
4.  打开大型笔记不卡顿

而且，它还有许多其他优点：

1.  支持多标签页
2.  支持标签模式、双链模式、模板系统、关系图谱
3.  丰富的第三方插件

## 2.编辑器使用

Obsidian的编辑器支持markdown语法，常用的markdown语法如下：

```text
1.标题
# H1
## H2
### H3

2.粗体
**bold text**

3.斜体
*italicized text*

4.引用块
> blockquote

5.有序列表
1. First item
2. Second item
3. Third item

6.无序列表
- First item
- Second item
- Third item

7.分隔线
---

8.代码块（去掉前面的#）
#```python 
#print('Hello World!')
#```

9.链接
[url](https://www.xxx.com)

10.图片（与标准markdown语法不同）
![[xxx.png]]

11.双向链接
[[Python]]
```

## 3.展开大纲

如图所示依次点击打开大纲模式： ![](https://www.bplan.top/2024-05-07.webp)

## 4.标签模式

在编辑器中输入 #python 会生成一个python标签，可以通过标签视图查看： ![](https://www.bplan.top/2024-05-08.webp)

## 5.双链模式

通过 \[\[xxx\]\] 语法来链接到xxx文件，可以在右侧查看当前文件的链接和反向链接： ![](https://www.bplan.top/2024-05-09.webp)

## 6.关系图谱

查看关系图谱，可以通过滚轮放大缩小： ![](https://www.bplan.top/2024-05-10.webp)

## 7.分屏和导出

Obsidian支持分屏模式和导出PDF功能，具体功能位置如下： ![](https://www.bplan.top/2024-05-11.webp)

## 8.图片存储位置配置

复制图片或截图后粘贴，图片将存储到指定的文件夹： ![](https://www.bplan.top/2024-05-06.webp)

## 9.远程坚果云同步配置

1.打开坚果云网页版，点击右上角昵称下面的账户信息： ![](https://www.bplan.top/2024-05-01.webp)

2.切换到安全选项，点击添加应用，输入任意应用名称后点击生成密码： ![](https://www.bplan.top/2024-05-02.webp)

3.打开Obsidian的设置--第三方插件，首次使用需要先关闭安全模式，点击浏览搜索 remotely save，并安装和启用： ![](https://www.bplan.top/2024-05-03.webp)

4.在 remotely save 插件的配置界面一共配置4个地方，其中服务器地址、用户名、密码分别填入第2步中的服务器地址、账户、密码： ![](https://www.bplan.top/2024-05-04.webp)

5.点击同步按钮即可进行远程同步（移动端按相同方式配置即可）： ![](https://www.bplan.top/2024-05-05.webp)

## 10.结语

本文简单介绍了Obsidian的基本使用和配置，但这只是其强大功能的一部分，后续会继续介绍Obsidian的进阶功能，希望能引起你的兴趣。