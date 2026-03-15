---
categories:
- - flet
date: 2024-12-07 10:18:03
description: '在上一篇教程中，我们搭建了笔记应用的基础界面。这一篇教程中，我们将为应用添加核心功能，包括新建、编辑、删除和搜索笔记等功能。我们会将代码重构为更易维护的类结构，并实现完整的交互逻辑。'
id: '395'
tags:
- GUI
title: Python Flet教程（二）：实现笔记应用的核心功能
---


## 1.开篇陈述

在上一篇教程中，我们搭建了笔记应用的基础界面。这一篇教程中，我们将为应用添加核心功能，包括新建、编辑、删除和搜索笔记等功能。我们会将代码重构为更易维护的类结构，并实现完整的交互逻辑。

## 2.项目结构

```python
noter/
│
├── main.py
├── note.py        # 新增：笔记数据模型
├── requirements.txt
└── assets/
    └── icon.png
```

## 3.代码分步讲解

### 3.1 笔记数据模型（note.py）

首先，我们创建笔记的数据模型，用于管理单个笔记的数据和行为：

```python
from dataclasses import dataclass
from datetime import datetime
import uuid

@dataclass
class Note:
    id: str
    title: str
    content: str
    created_at: datetime
    updated_at: datetime

    @staticmethod
    def create(title: str = "", content: str = "") -> "Note":
        now = datetime.now()
        return Note(
            id=str(uuid.uuid4()),
            title=title,
            content=content,
            created_at=now,
            updated_at=now
        )

    def update(self, title: str = None, content: str = None) -> None:
        if title is not None:
            self.title = title
        if content is not None:
            self.content = content
        self.updated_at = datetime.now()

    @property
    def preview(self) -> str:
        return self.content[:100] + "..." if len(self.content) > 100 else self.content
```

这个数据模型提供了：

*   使用`uuid`生成唯一ID
*   记录创建和更新时间
*   便捷的创建和更新方法
*   自动生成预览文本的属性

### 3.2应用类基础结构（main.py 第一部分）

```python
import flet as ft
from note import Note
from datetime import datetime

class NoterApp:
    def __init__(self, page: ft.Page):
        self.page = page
        self.setup_page()
        self.notes: list[Note] = []
        self.current_note: Note = None
        self.setup_controls()
        self.render()

    def setup_page(self):
        self.page.title = "Noter - 现代化的笔记应用"
        self.page.window_width = 1100
        self.page.window_height = 800
        self.page.padding = 0
        self.page.theme_mode = "light"
        self.page.window_min_width = 800
        self.page.window_min_height = 600
```

这部分代码：

*   定义了应用的主类`NoterApp`
*   初始化基本属性和页面设置
*   维护笔记列表和当前选中笔记

### 3\. 3核心功能实现（main.py 第二部分）

```python
    def create_note(self, e=None):
        # 创建新笔记
        note = Note.create("新建笔记", "")
        self.notes.append(note)
        self.select_note(note)
        self.update_notes_list()
        # 聚焦标题输入框
        self.title_field.focus()
        self.page.update()

    def select_note(self, note: Note):
        self.current_note = note
        self.title_field.value = note.title
        self.content_field.value = note.content
        self.page.update()

    def delete_note(self, note: Note):
        if note in self.notes:
            self.notes.remove(note)
            if self.current_note == note:
                self.current_note = None
                self.title_field.value = ""
                self.content_field.value = ""
            self.update_notes_list()
            self.page.update()

    def on_title_change(self, e):
        if self.current_note:
            self.current_note.update(title=e.control.value)
            self.update_notes_list()
            self.page.update()

    def on_content_change(self, e):
        if self.current_note:
            self.current_note.update(content=e.control.value)
            self.update_notes_list()
            self.page.update()

    def search_notes(self, e):
        query = e.control.value.lower()
        for note_container in self.notes_list.controls:
            note = note_container.data
            visible = (
                query in note.title.lower() or
                query in note.content.lower()
            )
            note_container.visible = visible
        self.page.update()
```

这部分实现了所有核心功能：

*    `create_note`: 创建新笔记
*    `select_note`: 选择并显示笔记
*    `delete_note`: 删除笔记
*    `on_title_change`和`on_content_change`: 处理笔记内容更新
*    `search_notes`: 实现笔记搜索功能

### 3.4笔记列表项创建（main.py 第三部分）

```python
    def create_note_item(self, note: Note) -> ft.Container:
        return ft.Container(
            content=ft.Column(
                controls=[
                    ft.Row(
                        controls=[
                            ft.Text(
                                note.title,
                                size=16,
                                weight=ft.FontWeight.BOLD,
                                expand=True
                            ),
                            ft.IconButton(
                                icon=ft.icons.DELETE,
                                icon_size=16,
                                on_click=lambda e, note=note: self.delete_note(note)
                            )
                        ]
                    ),
                    ft.Text(
                        note.preview,
                        size=14,
                        color=ft.colors.GREY_700,
                        overflow=ft.TextOverflow.ELLIPSIS,
                    ),
                ],
            ),
            data=note,  # 存储笔记引用
            padding=10,
            bgcolor=ft.colors.WHITE,
            border_radius=8,
            ink=True,
            on_click=lambda e, note=note: self.select_note(note)
        )

    def update_notes_list(self):
        self.notes_list.controls = [
            self.create_note_item(note) for note in reversed(self.notes)
        ]
        self.page.update()
```

这部分负责：

*   创建单个笔记列表项的UI
*   包含标题、预览内容和删除按钮
*   处理点击选择和删除事件
*   更新整个笔记列表的显示

### 3.5UI控件设置（main.py 第四部分）

```python
    def setup_controls(self):
        # 创建搜索框
        self.search_field = ft.TextField(
            hint_text="搜索笔记...",
            prefix_icon=ft.icons.SEARCH,
            width=200,
            on_change=self.search_notes
        )

        # 创建笔记列表
        self.notes_list = ft.ListView(
            spacing=2,
            padding=10,
        )

        # 创建标题输入框
        self.title_field = ft.TextField(
            label="标题",
            border=ft.InputBorder.NONE,
            text_style=ft.TextStyle(size=24, weight=ft.FontWeight.BOLD),
            on_change=self.on_title_change
        )

        # 创建内容输入框
        self.content_field = ft.TextField(
            label="开始输入笔记内容...",
            border=ft.InputBorder.NONE,
            multiline=True,
            min_lines=20,
            on_change=self.on_content_change
        )

        # 创建侧边栏
        self.sidebar = ft.Container(
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
                            on_click=self.create_note
                        ),
                        padding=ft.padding.only(left=20)
                    ),
                    ft.Container(
                        content=self.search_field,
                        padding=ft.padding.only(left=20, top=20)
                    ),
                ],
            ),
            width=250,
            bgcolor=ft.colors.BLUE_GREY_50,
            border=ft.border.only(right=ft.BorderSide(1, ft.colors.BLACK12)),
        )

        # 创建编辑区
        self.editor = ft.Container(
            content=ft.Column(
                controls=[
                    self.title_field,
                    self.content_field,
                ],
                spacing=20,
            ),
            padding=30,
            expand=True,
        )
```

这部分设置了所有UI控件：

*   搜索框和事件绑定
*   笔记列表视图
*   标题和内容输入框
*   侧边栏和新建按钮
*   编辑区布局

### 3.6主布局渲染（main.py 第五部分）

```python
    def render(self):
        # 主布局
        self.page.add(
            ft.Row(
                controls=[
                    self.sidebar,
                    ft.Container(
                        content=self.notes_list,
                        width=300,
                        bgcolor=ft.colors.with_opacity(0.5, ft.colors.BLUE_GREY_50),
                    ),
                    self.editor,
                ],
                expand=True,
            )
        )

def main(page: ft.Page):
    app = NoterApp(page)

if __name__ == "__main__":
    ft.app(target=main)
```

这部分完成了：

*   组合所有UI组件
*   创建三栏式布局
*   设置应用入口点

## 4.界面展示

如下是本篇完成后的笔记应用界面：

![](https://img.bplan.top/2024-12-2024-12-06_225048.webp)

## 5.本篇小结

在这一篇教程中，我们：

*   创建了笔记数据模型
*   实现了笔记的增删改查功能
*   添加了实时搜索功能
*   优化了用户界面和交互体验
*   使用类的方式重构了代码，提高了可维护性

下一篇教程中，我们将实现数据持久化，把笔记保存到SQLite数据库中，确保应用关闭后数据不会丢失。