---
categories:
- - flet
date: 2024-12-06 22:38:35
description: '在这个系列教程中，我们将使用Python的Flet框架来构建一个现代化的笔记应用。Flet是一个简单但功能强大的GUI框架，它允许我们使用Python创建美观的跨平台应用程序。 本系列将通过构建一个实际的笔记应用来学习Flet的核心概念和最佳实践。'
id: '377'
tags:
- GUI
title: Python Flet教程（一）：构建现代化笔记应用的基础界面
---


## 1\. 开篇陈述

在这个系列教程中，我们将使用Python的Flet框架来构建一个现代化的笔记应用。Flet是一个简单但功能强大的GUI框架，它允许我们使用Python创建美观的跨平台应用程序。 本系列将通过构建一个实际的笔记应用来学习Flet的核心概念和最佳实践。在第一篇教程中，我们将搭建应用的基础界面，包括：

*   创建主窗口
*   实现基本布局
*   设置笔记列表和编辑区

## 2\. 项目结构

```python
noter/
│
├── main.py
├── requirements.txt
└── assets/
    └── icon.png
```

requirements.txt内容：

```python
flet>=0.9.0
```

## 3\. 代码分步讲解

### 3.1 基础设置和导入

```python
import flet as ft

def main(page: ft.Page):
    # 配置页面基本属性
    page.title = "Noter - 现代化的笔记应用"
    page.window_width = 1100
    page.window_height = 800
    page.padding = 0
    page.theme_mode = "light"
    
    # 设置窗口最小尺寸
    page.window_min_width = 800
    page.window_min_height = 600
```

这段代码设置了应用窗口的基本属性，包括标题、尺寸和主题模式。我们将padding设为0以获得更现代的外观。

### 3.2 创建侧边栏

```python
    # 侧边栏
    sidebar = ft.Container(
        content=ft.Column(
            controls=[
                ft.Container(
                    content=ft.Text("Noter", size=32, weight=ft.FontWeight.BOLD),
                    padding=ft.padding.only(left=20, top=20, bottom=20)
                ),
                ft.Container(
                    content=ft.ElevatedButton(
                        "新建笔记",
                        icon=ft.icons.ADD,
                        width=200,
                    ),
                    padding=ft.padding.only(left=20)
                ),
                ft.Container(
                    content=ft.TextField(
                        hint_text="搜索笔记...",
                        prefix_icon=ft.icons.SEARCH,
                        width=200,
                    ),
                    padding=ft.padding.only(left=20, top=20)
                ),
            ],
        ),
        width=250,
        bgcolor=ft.colors.BLUE_GREY_50,
        border=ft.border.only(right=ft.BorderSide(1, ft.colors.BLACK12)),
    )
```

侧边栏包含应用标题、新建笔记按钮和搜索框。我们使用Container来管理布局和样式，确保组件之间有适当的间距。

### 3.3 创建笔记列表

```python
    # 笔记列表
    notes_list = ft.ListView(
        spacing=2,
        padding=10,
        controls=[
            ft.Container(
                content=ft.Column(
                    controls=[
                        ft.Text("示例笔记标题", size=16, weight=ft.FontWeight.BOLD),
                        ft.Text("这是笔记的预览内容...", size=14, color=ft.colors.GREY_700),
                    ],
                ),
                padding=10,
                bgcolor=ft.colors.WHITE,
                border_radius=8,
                ink=True,
            ) for _ in range(5)  # 创建5个示例笔记
        ],
    )
```

笔记列表使用ListView组件，每个笔记项都是一个可点击的Container。我们添加了一些示例笔记来展示布局效果。

### 3.4 创建编辑区

```python
    # 编辑区
    editor = ft.Container(
        content=ft.Column(
            controls=[
                ft.TextField(
                    label="标题",
                    border=ft.InputBorder.NONE,
                    text_style=ft.TextStyle(size=24, weight=ft.FontWeight.BOLD),
                ),
                ft.TextField(
                    label="开始输入笔记内容...",
                    border=ft.InputBorder.NONE,
                    multiline=True,
                    min_lines=20,
                ),
            ],
            spacing=20,
        ),
        padding=30,
        expand=True,
    )
```

编辑区包含标题和内容两个输入框，使用Column布局管理。我们移除了输入框的边框以获得更现代的外观。

### 3.5 组合整体布局

```python
    # 主布局
    page.add(
        ft.Row(
            controls=[
                sidebar,
                ft.Container(
                    content=notes_list,
                    width=300,
                    bgcolor=ft.colors.BLUE_GREY_50,
                ),
                editor,
            ],
            expand=True,
        )
    )

ft.app(target=main)
```

使用Row组件将侧边栏、笔记列表和编辑区组合在一起，创建三栏布局。

## 4.界面展示

如下是本篇完成后的笔记应用界面：

![](https://img.bplan.top/2024-12-2024-12-06_225048.webp)

## 5\. 本篇小结

在这一篇教程中，我们：

*    搭建了笔记应用的基础界面
*    学习了Flet的基本布局组件（Container、Row、Column）
*    实现了三栏式布局（侧边栏、笔记列表、编辑区）
*    使用了多种Flet控件（TextField、Button、ListView等）
*    设置了基本的样式（颜色、边框、内边距等）

下一篇教程中，我们将为这些界面元素添加实际的功能，包括新建笔记、选择笔记以及实时保存等功能。