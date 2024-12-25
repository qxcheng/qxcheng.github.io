---
categories:
- - Obsidian
date: 2024-05-17 10:15:43
description: 每日日记和任务管理是我们日常生活中比较常用的需求，本篇文章我们介绍如何使用Obsidian来方便地写日记和做任务管理。
id: '268'
tags:
- Obsidian
title: 使用Obsidian记日记和任务管理
---


每日日记和任务管理是我们日常生活中比较常用的需求，本篇文章我们介绍如何使用Obsidian来方便地写日记和做任务管理。

## 1.日记随笔

首先打开设置中日记的配置界面，配置日记存放位置和日记模板的位置，使用模板可以减少重复的输入： ![](https://www.bplan.top/2024-05-38.webp)

模板的内容可以自行输入： ![](https://www.bplan.top/2024-05-39.webp)

完成设置后，点击左侧新建日记按钮，即可生成今日日记（也可为其配置快捷方式）： ![](https://www.bplan.top/2024-05-40.webp)

## 2.任务管理

首先到插件市场安装这几个插件：Day Planner、Calendar、Dataview、Tasks插件，打开设置中Day Planner的配置界面，进行简单的配置： ![](https://www.bplan.top/2024-05-41.webp)

点击右上角的图标，打开Day Planner和Calendar的面板，并调整成上下分屏模式，点击日历中的日期默认会打开今日日记，点击Day Planner时间线中的区域可以生成一个待办任务到今日日记中，待办任务可以拖拽以改变待办时间： ![](https://www.bplan.top/2024-05-42.webp)

日记模板内容参考：

```text
# Day planner

- [ ] 07:00 - 07:20 起床
- [ ] 12:30 - 13:00 午休
- [ ] 23:00 - 07:00 睡觉

# 今日回顾

1.

2.

3.

# 今日反思

1.

2.

3.
```

为了方便快速打开这种布局模式，我们将其保存到工作区，首先在设置中打开工作区按钮： ![](https://www.bplan.top/2024-05-43.webp)

然后点击左侧的管理工作区布局按钮，输入名称后保存，下次直接加载即可： ![](https://www.bplan.top/2024-05-44.webp)

## 3.任务汇总

点击左侧的打开命令面板按钮，搜索tasks，点击创建一个新的任务（可为其设置一个快捷键，也可以在编辑器直接输入`- [ ] xxx任务`来创建待办）： ![](https://www.bplan.top/2024-05-45.webp)

我们输入任务描述，根据情况输入截止日期、计划日期、开始日期，可以输入英文的星期几，也可以直接输入日期，如`2024-5-18`，然后点击apply： ![](https://www.bplan.top/2024-05-46.webp)

我们可以根据如下语法来汇总任务，比如在我的日记目录下，未完成且截止日期为今天的任务，并按照优先级排序： ![](https://www.bplan.top/2024-05-47.webp)

完成效果如下所示： ![](https://www.bplan.top/2024-05-48.webp)

其他语法如下所示：

```python
not done 
done

# 时间索引
due today
due before next monday
no due date

path includes GitHub  # 路径包含GitHub
heading includes OB   # 标题包含OB
exclude sub-items     # 排除子待办事项

hide recurrence rule  # 隐藏循环任务的循环规则
hide task count       # 隐藏任务数量
hide backlink         # 隐藏反向链接（即任务所在的文件链接）

short mode            # 精简模式
limit 2               # 只显示前两条

# 排序
sort by due reverse 
sort by priority
sort by status
sort by description reverse
sort by path
```

## 4.结语

本篇内容我们介绍了Obsidian记日记和做任务管理的方法，得益于其灵活的语法，我们只需要少量的配置就能实现强大的功能，希望对你有所帮助。