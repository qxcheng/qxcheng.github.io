---
categories:
- - Python
date: 2024-04-17 13:25:30
description: 'Python常用库3-表格处理'
id: '115'
tags:
- python
title: Python常用库3-表格处理
---


## 1.openpyxl

```Python
# pip install openpyxl
import openpyxl

# 新建表格并写入
wb = openpyxl.Workbook()
ws = wb.create_sheet(index=0)
for i in range(1,5):                    
    ws.cell(i, 1).value = "NAME"   # 第i行第一列
    ws.cell(i, 2).value = "AGE"    # 第i行第二列
    ws.cell(i, 3).value = "BIRTH"  # 第i行第三列
wb.save("result.xlsx")

# 加载表格并读取
wb = openpyxl.load_workbook('./result.xlsx')
ws = wb.active 
ws = wb.worksheets[0]
ws = wb['Sheet1']
for row in ws.rows:
    name = row[0].value
    age = row[1].value
    birth = row[2].value
    print(name, age, birth)

# csv读取
import csv
with open('./result.csv', encoding='gb18030') as f:
    f_csv = csv.reader(f)
    for row in f_csv:
        value = row[0]
        print(value)
```

## 2.常用功能封装

```python
from typing import Optional, Sequence
import re
import openpyxl
from openpyxl.utils import get_column_letter
from openpyxl.styles import Alignment, colors, Font, PatternFill, Border, Side

g_mycolor = {
    'yellow': '00FFFF00',
    'red': '00FF0000',
    'blue': '6495ED'
}

class Excel(object):
    def __init__(self, filename) -> None:
        self.filename = filename

# 读取表格
class ExcelRead(Excel):
    def __init__(self, filename) -> None:
        super().__init__(filename)
        self.wb = openpyxl.load_workbook(self.filename)

    def read_ws(self, title:str):
        try:
            ws = self.wb[title]
        except:
            raise KeyError('Sheet not existed!')

        return ws

# 写入表格
class ExcelWrite(Excel):
    def __init__(self, filename) -> None:
        super().__init__(filename)
        self.wb = openpyxl.Workbook()
        self.ws_index = 0

    def create_ws(self, title):
        ws = self.wb.create_sheet(index=self.ws_index, title=title)
        self.ws_index += 1
        excel_ws = ExcelWs(ws)
        return excel_ws

    def save(self):
        self.wb.save(self.filename)

# 表格的Sheet类
class ExcelWs(object):
    def __init__(self, ws) -> None:
        self.index = 1  # 初始行号
        self.ws = ws
        self.base_alignment = Alignment(horizontal='center',vertical='center')  

    # 写入表格
    def _write(self, row:Sequence, font:Font=None, border:Border=None, fill_color:str=None):
        for i, item in enumerate(row):
            self.ws.cell(self.index, i+1).value = item
            self.ws.cell(self.index, i+1).alignment = self.base_alignment
            if font:
                self.set_font(i+1, font)
            if border:
                self.set_border(i+1, border)
            if fill_color:
                self.set_fill_color(i+1, fill_color)

        self.index += 1
        return self.index - 1  # 返回写入行的行号

    # 写入普通行
    def write_row(self, row:Sequence):
        return self._write(row=row, 
                           font=Font(name="微软雅黑", size=11), 
                           border=Border(left=Side(style='thin'),bottom=Side(style='thin'),right=Side(style='thin'),top=Side(style='thin'))
                           )

    # 写入标题行
    def write_title(self, row:Sequence):
        return self._write(row=row, 
                           font=Font(name="微软雅黑", size=11), 
                           border=Border(left=Side(style='thin'),bottom=Side(style='thin'),right=Side(style='thin'),top=Side(style='thin')),
                           fill_color='blue'
                           )

    # 写入一行 并调整行的高度
    def write_row_align_height(self, row:Sequence):
        height = 1
        for i, item in enumerate(row):
            self.ws.cell(self.index, i+1).value = item
            self.ws.cell(self.index, i+1).alignment = self.base_alignment
            if isinstance(item, str):
                height = item.count('\n') if item.count('\n') > height else height
        self.ws.row_dimensions[self.index].height = 18 * height
        self.index += 1
        return self.index - 1

    # 设置边框
    def set_border(self, column:int, border:Border):
        self.ws.cell(self.index, column).border = border

    # 设置超链接 link示例：test.xlsx#sheet1!A1
    def set_link(self, column:int, link:str):
        self.ws.cell(self.index, column).hyperlink = link
        self.set_font(column, Font(color=colors.BLUE, underline='single'))

    # 设置字体
    def set_font(self, column:int, font:Font):
        self.ws.cell(self.index, column).font = font

    # 设置字体颜色
    def set_color(self, column:int, color:str):
        self.set_font(column, Font(color=g_mycolor[color]))

    # 设置背景色
    def set_fill_color(self, column:int, color:str):
        self.ws.cell(self.index, column).fill = PatternFill('solid', fgColor=g_mycolor[color])

    # 单元格合并
    def merge_cell(self, start_row, start_column, end_row, end_column):
        self.ws.merge_cells(start_row=start_row, start_column=start_column, end_row=end_row, end_column=end_column)

    # 调整宽度
    def align_width(self):
        dims = {}
        for row in self.ws.rows:
            for cell in row:
                if cell.value:
                    cell_len = 0.9*len(re.findall('([\u4e00-\u9fa5])', str(cell.value))) + 1.2*len(str(cell.value))
                    dims[cell.column] = max((dims.get(cell.column, 0), cell_len))            
        for col, value in dims.items():
            try:
                self.ws.column_dimensions[get_column_letter(col)].width = value + 2
            except:
                self.ws.column_dimensions[col].width = value + 2

if __name__ == '__main__':
    excel_write = ExcelWrite(filename='test.xlsx')
    ws: ExcelWs = excel_write.create_ws(title='MySheet')

    ws.write_title(('姓名', '年龄', '性别', '分数'))
    ws.write_row(('张三', '16', '男', 85))
    ws.write_row(('李四', '17', '男', 80))

    excel_write.save()

    excel_read = ExcelRead(filename='test.xlsx')
    rs = excel_read.read_ws(title='MySheet')
    for row in rs.rows:
        for item in row:
            print(item.value, end=' ')
        print()
    # 姓名 年龄 性别 分数
    # 张三 16 男 85
    # 李四 17 男 80
```