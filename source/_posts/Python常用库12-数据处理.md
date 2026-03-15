---
categories:
- - Python
date: 2024-04-17 15:20:00
description: 'Python常用库12-数据处理'
id: '134'
tags:
- python
title: Python常用库12-数据处理
---


## 1.numpy

### 1.1创建

```python
import numpy as np
np.set_printoptions(threshold=np.inf)  # 解决大型数组打印不全

print(np.array([1,2,3], dtype=np.float32)) # [1. 2. 3.]
print(np.arange(3))        # [0 1 2]
print(np.linspace(1,10,4)) # [1. 4. 7. 10.] 4个点，间距=(10-1)/(4-1)
print(np.linspace(1,10,4, endpoint=False)) # [1. 3.25 5.5 7.75] 4个点，间距=(10-1)/4

print(np.ones((3,3)))    # 3x3元素全为1.的矩阵
print(np.zeros((3,3)))   # 3x3元素全为0.的矩阵
print(np.full((3,3), 2)) # 3x3元素全为2的矩阵
print(np.eye(3))         # 3x3单位矩阵 

a = np.arange(12).reshape(3,4) # 3x4的矩阵
print(np.ones_like(a))    # 3x4元素全为1的矩阵
print(np.zeros_like(a))   # 3x4元素全为0的矩阵
print(np.full_like(a, 2)) # 3x4元素全为2的矩阵  

np.random.seed(10)        # 随机数种子
print(np.random.random()) # 产生随机数

print(np.random.randint(10,20,(4,4))) # 由[10,20)的随机整数构成的4x4矩阵
print(np.random.rand(4,4))  # 4x4服从0~1均匀分布的随机矩阵
print(np.random.randn(4,4)) # 4x4服从标准正态分布的随机矩阵    
print(np.random.uniform(0,10,(4,4))) # 从[0,10)均匀取值的4x4随机矩阵
print(np.random.normal(2,3,(4,4))) # 4x4服从均值为2标准差为3的正态分布的随机矩阵
print(np.random.poisson(2,(4,4)))  # 4x4服从随机事件发生率为2的泊松分布的随机矩阵

a = np.arange(10)
print(np.random.choice(a,(3,3),replace=False,p=a/np.sum(a)))
# 从一维数组a中以概率p抽取3x3的元素，replace表示是否可以重用元素

a = np.arange(12).reshape(3,4) 
np.random.shuffle(a)         # 就地打乱数组，随机交换行
print(a)       

a = np.arange(12).reshape(3,4)
b = np.random.permutation(a) # 打乱数组，生成新数组
print(b)
```

### 1.2属性

```python
import numpy as np

a = np.arange(12).reshape(3,4) # 3x4矩阵
print(a.shape) # (3, 4)
print(a.dtype) # int32
print(a.ndim)  # 2 维度的数量
print(a.size)  # 12 元素个数
print(a.itemsize) # 4 元素的字节大小

print(a.mean()) # 5.5 所有元素平均值
print(a.sum())  # 66 所有元素的和
print(a.sum(axis=0)) # 竖向相加 [12 15 18 21]
print(a.sum(axis=1)) # 横向相加 [6 22 38]
print(a.argsort(axis=0)) # 3x4 竖向排序后从小到大元素对应的索引
print(a.argmax(axis=0))  # 1x4 竖向排序后最大元素对应的索引

print(a[1:4:2]) # 0维取第1行 起始:终止:步长
print(a[:,1])   # 0维取全部，一维取第1列
print(a[:,1:3]) # 0维取全部，一维取第1、2列

print(a.astype(np.float)) # 修改数组元素的类型
print(a.tolist())  # 将数组转化为列表
print(a.flatten()) # 展开成一维数组
print(a.reshape((4,3))) # reshape成4x3矩阵
print(a.swapaxes(0,1))  # 维度调换
```

### 1.3常用

```python
# 根据布尔值掩码改变元素
a = np.array([1,2,3,4,5], dtype=np.int)
a[a < 3] = 0   
print(a) # [0 0 3 4 5]

# 去重
print(np.unique([1, 2, 2, 5, 3, 4, 3]))
# 去重并按顺序返回一维数组 [1 2 3 4 5]

# 裁剪
a = np.array([1,2,3,5,6,7,8,9])
print(np.clip(a, 3, 8))  
# 所有元素最小为3，最大为8 array([3, 3, 3, 5, 6, 7, 8, 8])

# np.where
a = np.array([1, 2, 3, 4, 5])
print(np.where(a < 3, a, 0)) 
# 满足条件返回a，不满足返回0 [1 2 0 0 0]
print(np.where(a < 3))       
# 返回满足条件的索引 (array([0, 1], dtype=int64),)

# np.nonzero()
print(np.nonzero([0,1,2,3]))  #返回非0值的索引 (array([1, 2, 3], dtype=int64),)

import numpy as np

# 连接
a = np.array([[1, 2], 
              [3, 4]])
b = np.array([[5, 6]])   
print(np.concatenate((a, b), axis=0))
# [[1 2]
#  [3 4]
#  [5 6]]

# 堆叠
a = np.array([1, 2, 3])
b = np.array([4, 5, 6])
print(np.stack((a, b), axis=0)) # 2x3
# [[1 2 3]
#  [4 5 6]]
print(np.stack((a, b), axis=1)) # 3x2
# [[1 4]
#  [2 5]
#  [3 6]]

# 整体重复
print(np.tile([0,0],5))        
# [0,0]横向填充5次 [0 0 0 0 0 0 0 0 0 0]
print(np.tile([0,0],(5,2)))    
# [0,0]先竖向填充5次，再横向填充两次
# [[0 0 0 0]
#  [0 0 0 0]
#  [0 0 0 0]
#  [0 0 0 0]
#  [0 0 0 0]]

# 单独重复
a = np.array([[1,2],
              [3,4]])
print(np.repeat(a, 3, axis=1))
# [[1 1 1 2 2 2]
#  [3 3 3 4 4 4]]

# 填充
a = np.array([1, 1, 1])
b = np.pad(a,(1,2),'constant', constant_values=(0,2)) 
print(b) 
# [0 1 1 1 2 2] 前面填充1位0 后面填充2位2

a = np.array([[1,1],
              [2,2]])
b = np.pad(a,((1,1),(2,2)),'constant', constant_values=(0,3)) 
print(b)
# 先在行上填充一行0、行下填充一行3，再在列左填充两列0、列右填充两列3
# [[0 0 0 0 3 3]
#  [0 0 1 1 3 3]
#  [0 0 2 2 3 3]
#  [0 0 3 3 3 3]]

# 旋转
mat = np.arange(9).reshape(3,3)
print(mat)
print(np.rot90(mat, 1)) # 逆时针旋转90度
print(np.rot90(mat, 2)) # 逆时针旋转90度2次，相当于旋转180度

# 生成网格
x = np.array([0, 1, 2])
y = np.array([0, 1])
X, Y = np.meshgrid(x, y)
print(X)
print(Y)
```

### 1.4数学

```python
import numpy as np

print(np.abs(-1), np.fabs(-1))
print(np.sqrt(4), np.square(2))
print(np.ceil(4.5), np.floor(4.5))
print(np.exp(0), np.log(np.e), np.log10(10), np.log2(2))
print(np.cos(0), np.sin(0), np.tan(0))
print(np.cosh(0), np.sinh(0), np.tanh(0))

print(np.sign(2))   # 1 符号函数  
print(np.rint(4.6)) # 5.0 四舍五入
print(np.mod(5, 2)) # 1 取余
print(np.modf(4.6)) # 返回小数、整数两部分
print(np.copysign(3, -6))   # -3.0 -6的符号赋给3
print(np.gradient([1,3,6,10])) # [2. 2.5 3.5 4.] 梯度计算 二维时先0轴梯度再1轴，返回两个非独立数组

print(np.logical_and(1, 0)) # False 逻辑与
print(np.logical_or(1, 0))  # True  逻辑或
print(np.logical_not(1))    # False 逻辑非

print(np.maximum(1, 3))         # 3 对应取较大值
print(np.minimum([1,4], [2,3])) # [1 3] 对应取较小值

a = np.arange(6) # [0 1 2 3 4 5]
print(np.min(a))
print(np.max(a))
print(np.argmin(a)) # 最小值的下标
print(np.argmax(a)) 
print(np.sum(a,axis=None))  # 15
print(np.mean(a,axis=None)) # 2.5 均值
print(np.median(a)) # 2.5 中位数
print(np.average(a,axis=None,weights=[1,2,3,4,5,6])) # 加权平均值
print(np.std(a,axis=None)) # 标准差
print(np.var(a,axis=None)) # 方差
print(np.ptp(a))  # 最大值-最小值 
print(np.corrcoef(a, a)) # 相关系数
print(np.unravel_index(a,(2,3))) # 根据shape转换为多维下标
```

### 1.5矩阵

```python
a = np.random.randn(3,3)
m = np.mat(a) # 将数组转化为矩阵numpy.matrix
print(m.A)    # 将矩阵转换为数组
print(m.T)    # 矩阵转置           
print(m.I)    # 求逆矩阵
print(np.linalg.det(m))     # 求行列式 
print(np.linalg.eigvals(m)) # 求特征值

v = np.random.randn(3)      
x = np.linalg.solve(m,v)  #求 mx = v
print(x)
```

### 1.6存储

```python
a = np.arange(12).reshape(3,4)
np.savetxt('np.csv',a,fmt='%d',delimiter=',')       # 保存到csv
b = np.loadtxt('np.csv',dtype=np.int,delimiter=',') # 加载csv文件
print(b)

a = np.arange(24).reshape(2,3,4)
a.tofile('np.dat',sep=',',format='%d') # 多维数据的存储
b = np.fromfile('np.dat',dtype=np.int,sep=',')
b = b.reshape(2,3,4) 
print(b)

a = np.arange(24).reshape(2,3,4)
np.save('np.npy',a) # 快捷存储
b = np.load('np.npy')
print(b)
```

## 2.pandas

### 2.1 Series

带标签的一维数组，根据索引计算

```python
import pandas as pd
import numpy as np

#1 创建
print(pd.Series([9,8,7]))     
# 自动索引 0 9 1 8 2 7 
print(pd.Series([9,8,7], index=['a','b','c'])) 
# 自定义索引 a 9 b 8 c 7
print(pd.Series({'a':9,'b':8,"c":7})) 
# a 9 b 8 c 7

#2 操作
s = pd.Series([9,8,7,6],['a','b','c','d'])
print(s.index)  # 获取索引
print(s.values) # 获取值

#3 索引
print(s[0], s['a'])  # 9 9
print(s[:3])         # a 9 b 8 c 7
print('c' in s)      # True
print(s.get('f', 0)) # 0 索引不存在时返回0

s['a'] = 90
s['b','c'] = 80
s[s > s.median()] = 100
print(s) # a 100 b 80 c 80 d 6
print(s.drop(['b','c'])) # a 100 d 6 不改变s

s2 = np.exp(s)
print(s+s2)
```

### 2.2 DataFrame

带标签的二维数组

```python
import pandas as pd
import numpy as np

#1 创建
print(pd.DataFrame(np.arange(10).reshape(2,5)))
#   0 1 2 3 4
# 0 0 1 2 3 4
# 1 5 6 7 8 9

dt = {'one': pd.Series([1,2,3],['a','b','c']),
      'two': pd.Series([9,8,7,6],['a','b','c','d'])}
print(pd.DataFrame(dt))
#   one two
# a 1.0  9
# b 2.0  8
# c 3.0  7
# d NaN  6
print(pd.DataFrame(dt,index=['b','c','d'],columns=['two','three']))
#   two three
# b  8   NaN
# c  7   NaN
# d  6   NaN

#2 属性
dt = {'one': [1,2,3,4], 'two': [9,8,7,6]}
d = pd.DataFrame(dt,index=['a','b','c','d'])
#   one two
# a  1   9
# b  2   8
# c  3   7
# d  4   6
print(d.index)   
# Index(['a', 'b', 'c', 'd'], dtype='object')
print(d.columns) 
# Index(['one', 'two'], dtype='object')
print(d.values)
print(d['one'])
print(d['one']['a']) # 1

#3 运算
dt = {'one': [1,2,3,4], 'two': [9,8,7,6]}
d = pd.DataFrame(dt,index=['a','b','c','d'])
#   one two
# a  1   9
# b  2   8
# c  3   7
# d  4   6

print(d.add(1, fill_value=100))
print(d.sub(1))
print(d.mul(2))
print(d.div(pd.Series([2,3], index=['one','two'])))
#    one    two
# a  0.5  3.000000
# b  1.0  2.666667
# c  1.5  2.333333
# d  2.0  2.000000

print(d > 5)
print(d > pd.Series([2,3], index=['one','two']))
#     one   two
# a  False  True
# b  False  True
# c   True  True
# d   True  True

#4 索引排序
d = pd.DataFrame(np.arange(20).reshape(4,5),index=['c','a','d','b'])
#     0   1   2   3   4
# c   0   1   2   3   4
# a   5   6   7   8   9
# d  10  11  12  13  14
# b  15  16  17  18  19

print(d.sort_index())
print(d.sort_index(axis=1, ascending=False))
#     4   3   2   1   0
# c   4   3   2   1   0
# a   9   8   7   6   5
# d  14  13  12  11  10
# b  19  18  17  16  15

#5 值排序
np.random.seed(0)
d = pd.DataFrame(np.random.randint(1,9,(4,5)),index=['c','a','d','b'])
#    0  1  2  3  4
# c  5  8  6  1  4
# a  4  4  8  2  4
# d  6  3  5  8  7
# b  1  1  5  3  2

print(d.sort_values(2, ascending=False))
print(d.sort_values('d', axis=1))
#    1  2  0  4  3
# c  8  6  5  4  1
# a  4  8  4  4  2
# d  3  5  6  7  8
# b  1  5  1  2  3
```

### 2.3统计函数

```python
import pandas as pd

# Series:
s = pd.Series([5,8,9,4],['a','b','c','d'])
print(s.argmin(), s.argmax()) # 使用自动索引
print(s.idxmin(), s.idxmax()) # 自定义索引

# Series和DataFrame:
s = pd.Series([5,8,2,4],['a','b','c','d'])
print(s.sum())      # 19 
print(s.count())    # 4
print(s.mean())     # 4.75
print(s.median())   # 4.5
print(s.min())      # 2
print(s.max())      # 8
print(s.var())      # 6.25
print(s.std())      # 2.5
print(s.describe()) # 计算所有基本的统计值

print(s.cumsum())  # 依次给出前1、2、...、n个数的和
print(s.cumprod()) # 依次给出前1、2、...、n个数的积
print(s.cummax())  # 依次给出前1、2、...、n个数的最大值
print(s.cummin())  # 依次给出前1、2、...、n个数的最小值

print(s.rolling(2).sum())  # 依次计算相邻2个元素的和
print(s.rolling(2).mean()) # 依次计算相邻2个元素的平均值
print(s.rolling(2).var())  # 依次计算相邻2个元素的方差
print(s.rolling(2).std())  # 依次计算相邻2个元素的标准差
print(s.rolling(2).min())  # 依次计算相邻2个元素的最小值
print(s.rolling(2).max())  # 依次计算相邻2个元素的最大值

# 矩阵
s2 = pd.Series([3,7,1,9],['a','b','c','d'])
print(s.cov(s2))  # 计算协方差矩阵
print(s.corr(s2)) # 计算相关系数矩阵

d = pd.DataFrame(np.arange(20).reshape(4,5),index=['c','a','d','b'])
#     0   1   2   3   4
# c   0   1   2   3   4
# a   5   6   7   8   9
# d  10  11  12  13  14
# b  15  16  17  18  19
print(d.describe())

# 读取文件
print(pd.read_csv("result.csv"))     #读取 CSV 格式的数据集
print(pd.read_excel("result.xlsx"))  #读取 Excel 数据集
```

## 3.matplotlib

### 3.1基本绘图

```python
import matplotlib.pyplot as plt

plt.title('Test')    # 标题
plt.xlabel("Time")   # x轴标签
plt.ylabel("Num")    # y轴标签
plt.grid(True)       # 显示网格线

# 设置横坐标(0,10),纵坐标(0,6)
plt.axis([0,10,0,6]) 

# 绘制，x轴为(0,1,2,3)
plt.plot([1,2,3,4])  

# 绘制(x,y)
plt.plot([0,2,4,6],[1,2,3,4]) 

plt.savefig('plt', dpi=600)   # 保存为图片
plt.show()                    # 显示
```

### 3.2绘图样式

```python
import matplotlib.pyplot as plt

a = np.arange(0., 5., 0.2)
plt.plot(a,a,'go-', a,a*1.5,'rx', a,a**2,'b-.', linewidth=2.0) 

'''
颜色字符：b蓝色 g绿色 r红色 c青绿色 m洋红色 y黄色 k黑色 w白色

风格字符：-实线 --破折线 -.点划线 :虚线 ''无线条

标记字符：
.点标记 ,像素标记 o实心圈 
v倒三角 ^上三角 >右三角 <左三角
1下花三角 2上花三角 3左花三角 4右花三角
s实心方形 p实心五角 *星形 h竖六边形 H横六边形
+十字 x标记 D菱形 d瘦菱形 垂直线
'''
```

### 3.3绘图区域

```python
import matplotlib.pyplot as plt

a = np.arange(0.0,5.0,0.02)

for i in range(1,5):
    # 设置两行两列，索引为1,2,3,4
    plt.subplot(2,2,i) 
    plt.plot(a, a**2)

plt.show()
```

### 3.4文本箭头

```python
import matplotlib
import matplotlib.pyplot as plt

# 全局中文显示
matplotlib.rcParams['font.family']='SimHei' 
matplotlib.rcParams['font.style']='normal'  
matplotlib.rcParams['font.size']=18 
plt.ylabel('数量')

# 局部中文显示
plt.xlabel('时间',fontproperties='Simhei',fontsize=20,color='red') 

# 文本和箭头注解
a = np.arange(0.0,5.0,0.02)
plt.plot(a, a**2)

plt.text(1,2,'mu=10',fontsize=15)

plt.annotate('mu=10',xy=(2,1),xytext=(3,1.5), arrowprops=dict(facecolor='black',shrink=0.1,width=1))
```

### 3.5绘制图表

```python
import matplotlib.pyplot as plt
import numpy as np

#1 饼图
labels = 'Apple','Banana','Cherry','Date'
sizes  = [15,30,45,10]
explode = (0,0,0.1,0) # 表示突出谁
plt.pie(sizes,explode=explode,labels=labels,autopct='%1.1f%%',
shadow=False,startangle=90)
plt.axis('equal')     # 绘制圆形的饼
plt.show()

#2 散点图绘制
fig, ax = plt.subplots()
ax.plot(10*np.random.randn(100),10*np.random.randn(100),'o')
ax.set_title('Scatter')
plt.show()

#3 直方图绘制
np.random.seed(0)
mu, sigma = 100, 20 
a = np.random.normal(mu, sigma, size=100)
plt.hist(a,10,histtype='stepfilled',facecolor='b',alpha=0.75) 
# 10是直方的个数

#4 极坐标绘制
N = 20
theta = np.linspace(0.0,2*np.pi,N,endpoint=False)
radii = 10*np.random.rand(N)
width = np.pi/4*np.random.rand(N)
ax = plt.subplot(111,projection='polar')
bars = ax.bar(theta,radii,width=width,bottom=0.0) 
for r,bar in zip(radii,bars):
    bar.set_facecolor(plt.cm.viridis(r/10.))
    bar.set_alpha(0.5)
plt.show()
```