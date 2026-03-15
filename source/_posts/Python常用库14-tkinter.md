---
categories:
- - Python
date: 2024-04-19 09:14:00
description: 'Python常用库14-tkinter'
id: '140'
tags:
- GUI
- python
title: Python常用库14-tkinter
---


## 1.主框架

```Python
import tkinter as tk
root = tk.Tk()
root.title('my window')
root.geometry('600x800')
root.mainloop()
```

## 2.线程封装

```python
import tkinter as tk
import threading

class Application(tk.Tk):
    def __init__(self):
        super().__init__()
        self.createUI()

    # 生成界面
    def createUI(self):
        self.title('my window')
        self.geometry('720x960')

    # 线程封装
    @staticmethod
    def thread_it(func, *args):
        t = threading.Thread(target=func, args=args) 
        t.setDaemon(True)   
        t.start()         

if __name__ == '__main__': 
    app = Application()
    app.mainloop()
```

## 3.标签 变量

```Python
import tkinter as tk
from tkinter import ttk

root = tk.Tk()
root.title('my window')
root.geometry('450x600')

# 定义ttk风格
style = ttk.Style()
style.configure("TLabel", padding=6, relief="raised",
                foreground="red", background="yellow",
                font=('Arial', 12), width=15, anchor='center')
# relief取值：'raised' 'sunken' 'flat' 'ridge' 'groove' 'solid'

# 定义变量 tk.IntVar()
var = tk.StringVar()  
var.set('Label')

# 定义标签
l = ttk.Label(root, textvariable=var, style="TLabel")
l.pack(fill=tk.NONE, expand=True, side='top')  
# fill取值：tk.X tk.Y tk.BOTH tk.NONE 设置填充模式
# expand：设置组件是否展开，当值为True时，side选项无效，组件显示在父容器中心位置
# side取值：bottom, left, right 设置组件对齐方式

root.mainloop()
```

## 4.输入框 文本框 按钮

```Python
import tkinter as tk
from tkinter import ttk

root = tk.Tk()
root.title('my window')
root.geometry('450x600')

# 定义ttk风格
style = ttk.Style()
style.configure("TButton", relief="flat", font=('Arial', 12), 
                width=15, anchor='center')

# 定义按钮的回调函数
def hit_me():
    var = e.get()               # 从输入框获取内容
    t.insert('insert', var)     # 在文本框光标处输入内容
    t.insert('end', '#\n')      # 向文本框末尾处输入内容

# 输入框
e = ttk.Entry(root, show='*')   # 输入内容显示为*
e.pack(fill=tk.X, expand=True)

# 文本框
t = tk.Text(root, width=20, height=20, font=('Arial', 12))
t.pack(fill=tk.X, expand=True)

# 按钮
b = ttk.Button(root, text='hit me', style="TButton", command=hit_me) 
b.pack(expand=True)

root.mainloop()
```

## 5.Listbox

```Python
import tkinter as tk
from tkinter import ttk

root = tk.Tk()
root.title('my window')
root.geometry('450x600')

style = ttk.Style()
style.configure("TButton", padding=6, relief="flat",
                foreground="red", background="yellow",
                font=('Arial', 12), width=10, anchor='center')

def add_item():
    var = e.get() 
    lb.insert('end', var)  # 在末尾插入
    # lb.insert(0, var) 在最上面插入

def del_item():
    cursor = lb.curselection()
    if cursor:
        var = lb.get(cursor)    # 获取光标处的值         
        print("Delete: ", var)
        lb.delete(cursor)       # 删除光标处的值     

var = tk.StringVar()
var.set(('Tom','Mary','Jack'))

# Listbox
lb = tk.Listbox(root, listvariable=var) 
lb.pack(fill=tk.X, side='top', pady=16)

e = ttk.Entry(root) 
e.pack(fill=tk.X, side='top')

b1 = ttk.Button(root, text='增加', command=add_item) 
b1.pack(side='left')
b2 = ttk.Button(root, text='删除', command=del_item) 
b2.pack(side='left')

root.mainloop()
```

## 6.Radiobutton Checkbutton Scale

```Python
import tkinter as tk
from tkinter import ttk
import tkinter.messagebox as msg

root = tk.Tk()
root.title('my window')
root.geometry('450x600')

style = ttk.Style()
style.configure("TRadiobutton", padding=6, foreground="blue", font=('Arial', 10))
style.configure("TCheckbutton", padding=30, foreground="blue", font=('Arial', 10))

def show_item():
    message = f"Radiobutton: {var_r.get()}\n"
    message += f"Checkbutton: {var_c.get()}\n"
    message += f"Scale: {var_s.get()}\n"
    msg.showinfo(title='Hello', message=message)

var_r = tk.StringVar()
var_r.set('A')
var_c = tk.StringVar()
var_c.set(0)
var_s = tk.IntVar()

# Radiobutton 圆形选择框
r1 = ttk.Radiobutton(root, text='Option A', variable=var_r, value='A')
r1.pack(side='top')
r2 = ttk.Radiobutton(root, text='Option B', variable=var_r, value='B')
r2.pack(side='top')

# Checkbutton 勾选项
c = ttk.Checkbutton(root, text='Python', variable=var_c, onvalue=1, offvalue=0)
c.pack(side='top')

# Scale 滑动选择框
s = tk.Scale(root, label='scale', from_=5, to=11, orient=tk.HORIZONTAL,
             length=200, showvalue=1, tickinterval=2, resolution=0.01,
             variable=var_s)
s.pack()

b = ttk.Button(root, text='show', command=show_item) 
b.pack(side='bottom', pady=30)

root.mainloop()  
```

## 7.菜单 弹窗

```Python
import tkinter as tk
from tkinter import ttk
import tkinter.messagebox as msg
import tkinter.filedialog as fd

root = tk.Tk()
root.title('my window')
root.geometry('450x600')

# 弹窗
def msg_func():
    msg.showinfo(title='Hi', message='INFO')       # return 'ok'
    msg.showwarning(title='Hi', message='WARNING') # return 'ok'
    msg.showerror(title='Hi', message='ERROR')     # return 'ok'
    msg.askquestion(title='Hi', message='ask1')    # return 'yes' , 'no'
    msg.askyesno(title='Hi', message='ask2')       # return True, False
    msg.askokcancel(title='Hi', message='ask3')    # return True, False
    msg.askyesnocancel(title="Hi", message="ask4") # return, True, False, None

# 文件对话框
def open_file():
    filename = fd.askopenfilename(title = "Attach file", initialdir = 'd:/')
    f = open(filename, "r", encoding='utf8')
    print(f.read())
    f.close()

def exit_loop():
    root.quit()
    root.destroy()

# 菜单
menubar = tk.Menu(root)                               # 创建菜单栏容器
filemenu = tk.Menu(menubar, tearoff=0)                # 创建空菜单
menubar.add_cascade(label='File', menu=filemenu)      # 将空菜单添加至容器
filemenu.add_command(label='New', command=msg_func)   # 空菜单添加命令
filemenu.add_command(label='Open', command=open_file)
filemenu.add_command(label='Exit', command=exit_loop) # 退出
filemenu.add_separator()                              # 分割线
# 二级菜单
submenu = tk.Menu(filemenu)                           
filemenu.add_cascade(label='Import', menu=submenu, underline=0)
submenu.add_command(label="Image", command=msg_func)

root.config(menu=menubar) 

root.mainloop()
```

## 8.ttk控件

```Python
import tkinter as tk
from tkinter import ttk
import tkinter.messagebox as msg
import tkinter.filedialog as fd

root = tk.Tk()
root.title('my window')
root.geometry('450x600')

def show_combobox(event):
    msg.showinfo(title='Hi', message=cb.get())

def show_spinbox():
    print(s.get())

def show_tree(event):
    for select in tree.selection():
        msg.showinfo(message=tree.item(select, "text"))

# Notebook
notebook = ttk.Notebook(root)
page1 = tk.Frame(notebook, background="orange")
page2 = tk.Frame(notebook,  background="pink")
notebook.add(page1, text="Tab1")
notebook.add(page2, text="Tab2")
notebook.pack(fill=tk.BOTH, expand="yes")

# 输入框下拉选项菜单
cb = ttk.Combobox(page1, values=["Python", "Go", "Java", "C++"])
cb.current(1)
cb.pack(side='top', pady=10)
cb.bind('<<ComboboxSelected>>', show_combobox)  # 事件绑定

# 进度条
pb = ttk.Progressbar(page1, mode='determinate')  # indeterminate
pb.pack(pady=10)
pb.start()

# Spinbox
s = ttk.Spinbox(page2, from_=1, to=10, command=show_spinbox)
s.pack(pady=10)  

# 树形控件
tree = ttk.Treeview(page2, show='tree')
tree.bind("<<TreeviewSelect>>", show_tree)
item1 = tree.insert("", 0, text="A")
tree.insert(item1, 0, text="A1")
tree.insert(item1, 1, text="A2")
item2 = tree.insert("", 1, text="B")
tree.insert(item2, 0, text="B1")
tree.pack(pady=10)

root.mainloop()
```

## 9.布局

```Python
import tkinter as tk
import ttkbootstrap as ttk
from ttkbootstrap.constants import *

root = ttk.Window(themename="sandstone")
root.title('my window')
root.geometry('450x600')

# 在root上创建一个Frame
frm1 = ttk.Frame(root)                        
frm1.pack(side='top', fill=tk.X, padx=5, pady=10)
# 在Frame上添加部件
ttk.Button(frm1, text='frm1', bootstyle=INFO).pack(fill=tk.X) 

# 在Frame上继续创建Frame
frm2 = ttk.Frame(root) 
frm2.pack(side='top', fill=tk.X, padx=5, pady=10)
frm21 = ttk.Frame(frm2)                       
frm21.pack(side='left')    
frm22 = ttk.Frame(frm2)
frm22.pack(side='right')

ttk.Button(frm21, text='frm21', bootstyle=SUCCESS).pack()
ttk.Button(frm22, text='frm22', bootstyle=SUCCESS).pack()

# frm3
frm3 = ttk.Frame(root)
frm3.pack(expand=True, fill=tk.BOTH, padx=5, pady=10)
ttk.Button(frm3, text='frm3', bootstyle=PRIMARY).pack(fill=tk.X)

tk.Text(frm3).pack(expand=True, fill=tk.Y, pady=10)

root.mainloop()
```

## 10.Canvas构建绘图应用

```Python
import tkinter as tk
from tkinter import filedialog, colorchooser, simpledialog
import ttkbootstrap as ttk
from ttkbootstrap.constants import *
from PIL import ImageGrab, Image, ImageTk
import threading

class DrawingApp:
    def __init__(self, master):
        self.master = master
        self.canvas = tk.Canvas(master, bg='white')
        self.canvas.pack(expand=True, fill=tk.BOTH)

        self.shape = tk.StringVar(value="Line")
        self.color = tk.StringVar(value="black")
        self.width = tk.IntVar(value=1)

        self.canvas.bind("<Button-1>", self.start_draw)         # 鼠标左键点击事件
        self.canvas.bind("<B1-Motion>", self.draw)              # 鼠标左键移动事件
        self.canvas.bind("<ButtonRelease-1>", self.stop_draw)   # 鼠标左键释放事件
        self.canvas.bind("<Button-3>", self.select_shape)       # 鼠标右键点击事件
        self.canvas.bind("<B3-Motion>", self.move_shape)        # 鼠标右键移动事件

        self.start = None
        self.drawn = None
        self.current_drawn = None
        self.history = []            
        self.selected = None

    @staticmethod
    def thread_it(func, *args):
        t = threading.Thread(target=func, args=args) 
        t.setDaemon(True)   
        t.start()

    def choose_color(self):
        self.color.set(colorchooser.askcolor()[1])

    # 鼠标左键点击
    def start_draw(self, event):
        self.start = event
        if self.shape.get() == "Text":
            self.canvas.unbind("<B1-Motion>")
        else:
            self.canvas.bind("<B1-Motion>", self.draw)

    # 鼠标左键移动
    def draw(self, event):
        if self.drawn not in self.history:
            self.canvas.delete(self.drawn)
        if self.shape.get() == "Line":
            self.drawn = self.canvas.create_line(self.start.x, self.start.y, event.x, event.y, fill=self.color.get(), width=self.width.get())
        elif self.shape.get() == "Rectangle":
            self.drawn = self.canvas.create_rectangle(self.start.x, self.start.y, event.x, event.y, outline=self.color.get(), width=self.width.get())
        elif self.shape.get() == "Oval":
            self.drawn = self.canvas.create_oval(self.start.x, self.start.y, event.x, event.y, outline=self.color.get(), width=self.width.get())
        elif self.shape.get() == "Arrow":
            self.drawn = self.canvas.create_line(self.start.x, self.start.y, event.x, event.y, arrow=tk.LAST, fill=self.color.get(), width=self.width.get())

    # 鼠标左键释放
    def stop_draw(self, event):
        if self.shape.get() == "Text":
            text = simpledialog.askstring("Input", "Enter text:")
            self.drawn = self.canvas.create_text(self.start.x, self.start.y, text=text, fill=self.color.get(), font=('Arial', 12, 'bold'))
            self.canvas.bind("<B1-Motion>", self.draw)
        elif self.shape.get() == "Image":
            file_path = filedialog.askopenfilename()
            img = Image.open(file_path)
            img = img.resize((self.canvas.winfo_width(), self.canvas.winfo_height()))
            self.image = ImageTk.PhotoImage(img)
            self.drawn = self.canvas.create_image(0, 0, image=self.image, anchor=tk.NW)
        self.history.append(self.drawn)

    # 鼠标右键点击
    def select_shape(self, event):
        try:
            self.selected = self.canvas.find_closest(event.x, event.y)[0]
        except IndexError:
            pass

    # 鼠标右键移动
    def move_shape(self, event):
        if self.selected:
            self.canvas.move(self.selected, event.x - self.canvas.coords(self.selected)[0], event.y - self.canvas.coords(self.selected)[1])

    # 导出保存图片
    def export_to_png(self):
        x = self.master.winfo_rootx() + self.canvas.winfo_x()
        y = self.master.winfo_rooty() + self.canvas.winfo_y()
        x1 = x + self.canvas.winfo_width()
        y1 = y + self.canvas.winfo_height()
        ImageGrab.grab().crop((x, y, x1, y1)).save("output.png")

    # 撤销
    def undo(self):
        if self.history:
            item = self.history.pop()
            self.canvas.delete(item)

if __name__ == '__main__':
    root = ttk.Window(themename="sandstone")
    root.title('my window')
    root.geometry('450x600')
    app = DrawingApp(root)

    menubar = tk.Menu(root)
    root.config(menu=menubar)

    filemenu = tk.Menu(menubar, tearoff=0)
    menubar.add_cascade(label="File", menu=filemenu)
    filemenu.add_command(label="Save as PNG", command=app.export_to_png)

    editmenu = tk.Menu(menubar, tearoff=0)
    menubar.add_cascade(label="Edit", menu=editmenu)
    editmenu.add_command(label="Undo", command=app.undo)

    colormenu = tk.Menu(menubar, tearoff=0)
    menubar.add_cascade(label="Color", menu=colormenu)
    colormenu.add_command(label="Choose Color", command=app.choose_color)

    widthmenu = tk.Menu(menubar, tearoff=0)
    menubar.add_cascade(label="Width", menu=widthmenu)
    for i in range(1, 11):
        widthmenu.add_radiobutton(label=str(i), variable=app.width, value=i)

    shapemenu = tk.Menu(menubar, tearoff=0)
    menubar.add_cascade(label="Shape", menu=shapemenu)
    for shape in ["Line", "Rectangle", "Oval", "Arrow", "Text", "Image"]:
        shapemenu.add_radiobutton(label=shape, variable=app.shape, value=shape)

    root.mainloop()
```

## 11.音乐播放器

```python
import os
import vlc
import tkinter as tk
from tkinter import ttk, filedialog
import ttkbootstrap as ttk
from ttkbootstrap.constants import *
import threading
import time
import wave
import numpy as np

class MusicPlayer:
    def __init__(self, root):
        self.root = root

        self.music_files = []
        self.current_file = None
        self.player = vlc.MediaPlayer()
        self.is_playing = False           # 是否正在播放
        self.is_seeking = False           # 是否在更新进度
        self.has_played = False           # 是否已播放

        self.waveform = None              # 波形提取相关
        self.waveform_length = 0

        self.createUI()

    def createUI(self):
        # 菜单栏
        menubar = tk.Menu(self.root)
        self.root.config(menu=menubar)
        filemenu = tk.Menu(menubar, tearoff=0)
        menubar.add_cascade(label='File', menu=filemenu)
        filemenu.add_command(label='Add Folder to Playlist', command=self.add_to_playlist)

        # 左边Frame
        left_frame = ttk.Frame(self.root)
        left_frame.pack(side=tk.LEFT, fill='both', expand=True)
        # 左边Frame：播放列表
        self.playlist = tk.Listbox(left_frame)
        self.playlist.pack(fill='both', expand=True, side=tk.LEFT)
        # 左边Frame：滚动条
        scrollbar = ttk.Scrollbar(left_frame, orient='vertical', command=self.playlist.yview)
        scrollbar.pack(side='right', fill='y')

        self.playlist.configure(yscrollcommand=scrollbar.set)

        # 右边Frame
        right_frame = ttk.Frame(self.root)
        right_frame.pack(side=tk.RIGHT, fill='both', expand=True, padx=10)
        # 右边Frame：动效
        self.visualizer = tk.Canvas(right_frame, width=200, height=95)
        self.visualizer.pack(pady=10, fill='x', expand=True)
        # 右边Frame：底部Frame
        btn_frame = ttk.Frame(right_frame)
        btn_frame.pack(side='bottom', pady=10)

        # 右边Frame：底部Frame：三个按钮
        self.prev_button = ttk.Button(btn_frame, text="Previous", command=lambda :self.thread_it(self.prev_music), bootstyle=SUCCESS)
        self.prev_button.pack(side='left', padx=5)

        self.play_button = ttk.Button(btn_frame, text="Play", command=lambda :self.thread_it(self.play_pause))
        self.play_button.pack(side='left', padx=5)

        self.next_button = ttk.Button(btn_frame, text="Next", command=lambda :self.thread_it(self.next_music), bootstyle=SUCCESS)
        self.next_button.pack(side='left', padx=5)

        # 右边Frame：进度条
        self.progress = ttk.Scale(right_frame, length=200, command=self.seek)
        self.progress.pack(pady=10, fill='x', expand=True)

        # 右边Frame：时间展示
        self.time_label = ttk.Label(right_frame, text="")
        self.time_label.pack()

        self.player.event_manager().event_attach(vlc.EventType.MediaPlayerEndReached, self.next_music)

        # 更新进度条
        t = threading.Thread(target=self.update_progress, daemon=True)
        t.start()

    # 播放和暂停
    def play_pause(self):
        # 如果正在播放则直接暂停
        if self.is_playing:
            self.player.pause()
            self.is_playing = False
            self.play_button['text'] = "Play"
        # 如果没有正在播放
        else:
            # 如果没有播放列表
            if not self.music_files:
                self.add_to_playlist()
            self.current_file = self.music_files[self.playlist.curselection()[0]]

            # 如果当前歌曲之前播放过一段时间则接着播放
            if self.player.get_media():
                self.player.play()
            # 否则选择当前选中歌曲开始播放
            elif os.path.isfile(self.current_file):
                self.player.set_media(vlc.Media(self.current_file))
                self.player.play()

            self.is_playing = True
            self.play_button['text'] = "Pause"

            self.waveform = wave.open(self.current_file, 'r')
            self.waveform_length = self.waveform.getnframes()

    # 添加到播放列表
    def add_to_playlist(self):
        folder_path = filedialog.askdirectory()
        if folder_path:
            self.music_files = [os.path.join(folder_path, file) for file in os.listdir(folder_path) if file.endswith((".mp3", ".wav", "flac"))]
            self.playlist.delete(0, tk.END)
            for item in self.music_files:
                self.playlist.insert(tk.END, os.path.basename(item))

    # 上一首
    def prev_music(self):
        cur_idx = self.playlist.curselection()[0]
        prev_idx = (cur_idx - 1) % len(self.music_files)
        self.playlist.select_clear(cur_idx)
        self.playlist.select_set(prev_idx)
        self.current_file = self.music_files[self.playlist.curselection()[0]]
        self.player.set_media(vlc.Media(self.current_file))
        self.player.play()
        self.progress.set(0)

    # 下一首
    def next_music(self, event=None):
        cur_idx = self.playlist.curselection()[0]
        next_idx = (cur_idx + 1) % len(self.music_files)
        self.playlist.select_clear(cur_idx)
        self.playlist.select_set(next_idx)
        self.current_file = self.music_files[self.playlist.curselection()[0]]
        self.player.set_media(vlc.Media(self.current_file))
        self.player.play()
        self.progress.set(0)

    # 更新进度条
    def update_progress(self):
        while True:
            if self.player.get_media() and self.is_playing:
                cur = self.player.get_time() // 1000
                total = self.player.get_length() // 1000

                if cur != 0 and total != 0:
                    self.is_seeking = True
                    if total - cur <= 1:
                        self.next_music()
                        continue
                    self.progress.set(round(cur / total, 2))
                    self.is_seeking = False

                text = '' 
                if cur > 60:
                    text += f'{cur//60}m{cur-cur//60*60}s'
                else:
                    text += f'{cur}s'
                if total > 60:
                    text += f'/{total//60}m{total-total//60*60}s'
                else:
                    text += f'/{total}s'
                self.time_label.config(text=text)

                self.update_waveform()
                time.sleep(0.5)

    # 拖动进度条事件的回调            
    def seek(self, event):
        if self.is_playing and not self.is_seeking:
            if 1 - float(event) < 0.01:
                return
            self.player.set_position(float(event))

    # 更新波形图
    def update_waveform(self):
        if self.is_playing:
            if self.player.get_length() == 0:
                return
            position = self.player.get_time() / self.player.get_length()
            start_frame = int(position * self.waveform_length)
            self.waveform.setpos(start_frame)

            # Read a small portion of the waveform
            frames = self.waveform.readframes(1000)
            samples = np.frombuffer(frames, dtype=np.int16)

            self.visualizer.delete('all')
            for i, sample in enumerate(samples):
                if i % 50 == 0:
                    x = i * 400 // len(samples)
                    y = sample * 200 // 32768
                    self.visualizer.create_rectangle(x, 100 - y, x + 10, 100 + y, fill="#007BFF")

    @staticmethod
    def thread_it(func, *args):
        t = threading.Thread(target=func, args=args) 
        t.setDaemon(True)   
        t.start()

if __name__ == '__main__':
    root = ttk.Window(themename="sandstone")
    root.title('Music Player')
    root.geometry('500x200')
    app = MusicPlayer(root)
    root.mainloop()
```