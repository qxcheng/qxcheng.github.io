---
categories:
- - Python
date: 2024-04-17 15:29:19
description: 'Python常用库13-图像处理'
id: '136'
tags:
- python
title: Python常用库13-图像处理
---


## 1.pillow

```Python
from PIL import Image

#1 基础
im = Image.open("test.png")
print(im.format, im.size, im.mode) 
# PNG (512,512) RGBA, size=(宽，高)
im.show() 

# 灰度转换
im = im.convert("L")  
# 对每个像素点执行函数运算
im = im.point(lambda i: i * 1.2) 

im = im.resize((128, 128)) # 缩放
im.save("result.png")      # 保存

# 生成JPEG缩略图
im.thumbnail((128, 128))
im.save("result.thumbnail", "JPEG")

#2 裁剪
im = Image.open("test.jpg")
# (左上角x, 左上角y, 右下角x, 右下角y)
box = (100, 100, 400, 400) 
out = im.crop(box)
out.show()

#3 粘贴
im2 = Image.open("test.png")
im2.paste(out, box)                        
im2.show()

#4 旋转
out = im.transpose(Image.FLIP_LEFT_RIGHT)    
out = im.transpose(Image.FLIP_TOP_BOTTOM)
out.show()

out = im.transpose(Image.ROTATE_90)
out = im.transpose(Image.ROTATE_180)
out = im.transpose(Image.ROTATE_270)
out.show()

out = im.rotate(45) 
out.show()

#5 通道分离与合并
im = Image.open("test.jpg")
r, g, b = im.split()               
im = Image.merge("RGB", (b, g, r)) 
im.show()

#6 效果增强
from PIL import ImageEnhance 
from PIL import ImageFilter  

im = Image.open("test.jpg")

# 对比度
enh = ImageEnhance.Contrast(im)
enh.enhance(1.3).show("30% more contrast")

# 滤镜
out = im.filter(ImageFilter.DETAIL)
out.show()

#7 图像阵列
im = Image.open("test.gif")
im.seek(1)  # 跳到下一帧
try:
    while 1:
        im.seek(im.tell()+1)
        im.show()
except EOFError:   
    # 在帧尾时会得到一个EOFError异常
    pass 

#8 与numpy互相转换
a = np.array(Image.open("test.jpg"))
print(a.shape, a.dtype)
b = [255, 255, 255] - a
im = Image.fromarray(b.astype('uint8'))
im.save("test2.jpg")
```

## 2.opencv

### 基础

```Python
import cv2 

img = cv2.imread('test.jpg', 1) # 图片读取 传0转换为灰度
cv2.namedWindow("test") 
cv2.imshow('test',img) # 图像显示
cv2.waitKey(0)

# 获取像素值，(高度,宽度)
(b,g,r) = img[100,200]                         
print(b,g,r)

# 图片ROI
dst = img[200:400,200:400]                     
cv2.imshow('test',dst) 
cv2.waitKey(0)

# 图像加权融合
dst1 = img[200:400,200:400]   
dst2 = img[0:200,0:200]
dst = cv2.addWeighted(dst1,0.5,dst2,0.5,0)
cv2.imshow('test',dst) 
cv2.waitKey(0)

# 灰度转换
grey = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY) 
#img = cv2.cvtColor(np.array(img), cv2.COLOR_RGBA2RGB)  
cv2.imshow('test',grey) 
cv2.waitKey(0)

# 图片缩放
dst = cv2.resize(img, (100,200)) # (宽度,高度)
cv2.imshow('test',dst) 
cv2.waitKey(0)

# 像素遍历
height   = img.shape[0]
weight   = img.shape[1]
channels = img.shape[2]
for row in range(height):               # 遍历高
    for col in range(weight):           # 遍历宽
        for c in range(channels):       # 遍历通道
            pv = img[row, col, c] 
            img[row, col, c] = 255 - pv # 颜色反转                            
cv2.imshow('test',img) 
cv2.waitKey(0)  

# 图像写入
cv2.imwrite('cv-test.jpg', img) 
cv2.imwrite('cv-test-50.jpg',img,[cv2.IMWRITE_JPEG_QUALITY,50]) #0-100 压缩比高
cv2.imwrite('cv-test.png',img,[cv2.IMWRITE_PNG_COMPRESSION,0])  #0-9   压缩比低
```

### 阈值化滤波

```python
import cv2
import numpy as np

img = cv2.imread('test.jpg', 0)

# 阈值化
ret,dst = cv2.threshold(img,127,255,cv2.THRESH_BINARY)
cv2.imshow('test',dst) 
cv2.waitKey(0) 
# cv2.THRESH_BINARY_INV
# cv2.THRESH_TRUNC
# cv2.THRESH_TOZERO
# cv2.THRESH_TOZERO_INV

# 自适应阈值化
dst = cv2.adaptiveThreshold(img,255,cv2.ADAPTIVE_THRESH_MEAN_C, cv2.THRESH_BINARY,11,2)
cv2.imshow('test',dst) 
cv2.waitKey(0) 
# cv2.ADAPTIVE_THRESH_GAUSSIAN_C

# OTSU阈值化
ret,dst = cv2.threshold(img,0,255,cv2.THRESH_BINARY+cv2.THRESH_OTSU)
cv2.imshow('test',dst) 
cv2.waitKey(0)

# 自定义滤波
kernel = np.ones((5,5),np.float32)/25
dst = cv2.filter2D(img,-1,kernel)
cv2.imshow('test',dst) 
cv2.waitKey(0)

# 平均滤波
dst = cv2.blur(img,(5,5))
cv2.imshow('test',dst) 
cv2.waitKey(0)

# 高斯均值滤波
dst = cv2.GaussianBlur(img,(5,5),1.5)   
cv2.imshow('test',dst) 
cv2.waitKey(0) 

# 中值滤波
dst = cv2.medianBlur(img,5)
cv2.imshow('test',dst) 
cv2.waitKey(0) 

# 双边滤波
dst = cv2.bilateralFilter(img,15,35,35) 
cv2.imshow('test',dst) 
cv2.waitKey(0) 
```

### 形态学处理

```python
import cv2

img = cv2.imread('test.jpg', 0)
kernel = np.ones((5,5),np.uint8)

# 腐蚀
dst = cv2.erode(img,kernel,iterations = 1)
cv2.imshow('test',dst) 
cv2.waitKey(0) 

# 膨胀
dst = cv2.dilate(img,kernel,iterations = 1)
cv2.imshow('test',dst) 
cv2.waitKey(0) 

# 开运算
opening = cv2.morphologyEx(img, cv2.MORPH_OPEN, kernel)
cv2.imshow('test',dst) 
cv2.waitKey(0) 

# 闭运算
dst = cv2.morphologyEx(img, cv2.MORPH_CLOSE, kernel)
cv2.imshow('test',dst) 
cv2.waitKey(0) 

# 形态学梯度
dst = cv2.morphologyEx(img, cv2.MORPH_GRADIENT, kernel)
cv2.imshow('test',dst) 
cv2.waitKey(0) 

# 顶帽
dst = cv2.morphologyEx(img, cv2.MORPH_TOPHAT, kernel)
cv2.imshow('test',dst) 
cv2.waitKey(0) 

# 黑帽
dst = cv2.morphologyEx(img, cv2.MORPH_BLACKHAT, kernel)
cv2.imshow('test',dst) 
cv2.waitKey(0) 
```

### 边缘轮廓

```python
import cv2
import numpy as np

img = cv2.imread('test.jpg', 0)

# 边缘检测
dst = cv2.GaussianBlur(img,(3,3),0)        
dst = cv2.Canny(dst,50,50) 
cv2.imshow('test',dst) 
cv2.waitKey(0)

# 轮廓检测
ret, thresh = cv2.threshold(img, 127, 255, 0)
contours, hierarchy = cv2.findContours(thresh, cv2.RETR_TREE, cv2.CHAIN_APPROX_SIMPLE)
cv2.drawContours(img, contours, -1, (0,255,0), 3)
cv2.imshow('test',img) 
cv2.waitKey(0)

# 轮廓拟合
cnt = contours[0]
x,y,w,h = cv2.boundingRect(cnt)
cv2.rectangle(img,(x,y),(x+w,y+h),(255,255,0),2)
cv2.imshow('test',img) 
cv2.waitKey(0)
```

### 绘制

```python
import cv2

# 绘制
dst = np.zeros((500,500,3), np.uint8)

cv2.line(dst,(100,20),(400,20),(0,255,0),20,cv2.LINE_AA)    #线段 dst/begin/end/color/line w/line type
cv2.rectangle(dst,(30,50),(120,200),(255,0,0),5)            #矩形 左上角/右下角/color/line
cv2.circle(dst,(250,100),(50),(0,255,0),2)                  #圆形 center/radius
cv2.ellipse(dst,(256,256),(150,100),0,0,180,(255,255,0),-1) #椭圆 center/轴/angle/begin/end

points = np.array([[150,50],[140,140],[200,170],[250,250],[150,50]],np.int32) #(5,2)
points = points.reshape((-1,1,2))                                             #(5,1,2)
cv2.polylines(dst,[points],True,(0,255,255))                                  #折线

font = cv2.FONT_HERSHEY_SIMPLEX
cv2.putText(dst,'xxxx',(100,400),font,1,(200,100,255),2,cv2.LINE_AA) #文字 文字内容/坐标/字体/字体大小/color/粗细/line type

cv2.imshow('test',dst) 
cv2.waitKey(0)
```

### 直方图

```python
import cv2
from matplotlib import pyplot as plt

grey = cv2.imread('test.jpg', 0)
img = cv2.imread('test.jpg', 1)

# 直方图绘制
color = ('b','g','r')
for i,col in enumerate(color):
    histr = cv2.calcHist([img],[i],None,[256],[0,256])
    plt.plot(histr,color = col)
    plt.xlim([0,256])
plt.show()

#灰度直方图均衡化
dst = cv2.equalizeHist(grey) 
cv2.imshow('test',dst) 
cv2.waitKey(0)

#彩色直方图均衡化
(b,g,r) = cv2.split(img)       # 通道分解
bH = cv2.equalizeHist(b)
gH = cv2.equalizeHist(g)
rH = cv2.equalizeHist(r)
dst = cv2.merge((bH,gH,rH)) # 通道合成
cv2.imshow('test',dst) 
cv2.waitKey(0)

#YUV直方图均衡化
imgYUV = cv2.cvtColor(img,cv2.COLOR_BGR2YCrCb)
channelYUV = cv2.split(imgYUV)
channelYUV[0] = cv2.equalizeHist(channelYUV[0])
channels = cv2.merge(channelYUV)
dst = cv2.cvtColor(channels,cv2.COLOR_YCrCb2BGR)
cv2.imshow('test',dst) 
cv2.waitKey(0)

# 彩图直方图
import cv2
import numpy as np

def ImageHist(image,type):
    color = (255,255,255)
    windowName = 'Grey'
    if type == 31:
        color = (255,0,0)
        windowName = 'B Hist'
    elif type == 32:
        color = (0,255,0)
        windowName = 'G Hist'
    elif type == 33:
        color = (0,0,255)
        windowName = 'R Hist'
    # 1 image 2 [0] 3 mask None 4 256 5 0-255
    hist = cv2.calcHist([image],[0],None,[256],[0.0,255.0])
    minV,maxV,minL,maxL = cv2.minMaxLoc(hist)
    histImg = np.zeros([256,256,3],np.uint8)
    for h in range(256):
        intenNormal = int(hist[h]*256/maxV)
        cv2.line(histImg,(h,256),(h,256-intenNormal),color)
    cv2.imshow(windowName,histImg)
    return histImg

img = cv2.imread('test.jpg',1)
channels = cv2.split(img)# RGB - R G B
for i in range(0,3):
    ImageHist(channels[i],31+i)
cv2.waitKey(0)
```

### 几何变换

```python
import cv2
import numpy as np

img = cv2.imread('test.jpg', 1)
height = img.shape[0]
width = img.shape[1]

# 图片缩放
matScale = np.float32([[0.5,0,0],[0,0.5,0]])
dst = cv2.warpAffine(img,matScale,(int(width/2),int(height/2)))
cv2.imshow('test',dst) 
cv2.waitKey(0)

# 仿射变换
# 左上角 左下角 右上角
matSrc = np.float32([[0,0],[0,height-1],[width-1,0]]) 
matDst = np.float32([[50,50],[300,height-200],[width-300,100]])
matAffine = cv2.getAffineTransform(matSrc,matDst)
dst = cv2.warpAffine(img,matAffine,(width,height))
cv2.imshow('test',dst) 
cv2.waitKey(0)

# 图片旋转
# center/angle/scale
matRotate = cv2.getRotationMatrix2D((height*0.5,width*0.5),45,1)
dst = cv2.warpAffine(img,matRotate,(height,width))
cv2.imshow('test',dst) 
cv2.waitKey(0)

# 透视变换
pts1 = np.float32([[200,200],[500,200],[200,500],[500,500]])
pts2 = np.float32([[0,0],[300,0],[0,300],[300,300]])
M = cv2.getPerspectiveTransform(pts1,pts2)
dst = cv2.warpPerspective(img,M,(300,300))
cv2.imshow('test',dst) 
cv2.waitKey(0)

# 图片镜像
img = cv2.imread('test.jpg',1)
cv2.imshow('src',img)
imgInfo = img.shape
height = imgInfo[0]
width = imgInfo[1]
deep = imgInfo[2]
newImgInfo = (height*2,width,deep)
dst = np.zeros(newImgInfo,np.uint8)#uint8 
for i in range(0,height):
    for j in range(0,width):
        dst[i,j] = img[i,j]
        dst[height*2-i-1,j] = img[i,j]
for i in range(0,width):
    dst[height,i] = (0,0,255)#BGR
cv2.imshow('dst',dst)
cv2.waitKey(0)

# 图片移位
img = cv2.imread('test.jpg',1)
imgInfo = img.shape
height = imgInfo[0]
width = imgInfo[1]

matShift = np.float32([[1,0,100],[0,1,200]])      # 2*3
dst = cv2.warpAffine(img,matShift,(height,width)) #1 data 2 mat 3 info
cv2.imshow('dst',dst)
cv2.waitKey(0)

# 图片移位2
dst = np.zeros(img.shape,np.uint8)
for i in range(0,height):
    for j in range(0,width-100):
        dst[i,j+100]=img[i,j]
cv2.imshow('image',dst)
cv2.waitKey(0)
```

### 视频合成分解

```python
#1 视频分解图片
import cv2
cap = cv2.VideoCapture("1.mp4") # 获取视频打开
isOpened = cap.isOpened         # 判断是否打开
print(isOpened)
fps = cap.get(cv2.CAP_PROP_FPS)                  # 帧率
width = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))   # w 
height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT)) # h
print(fps,width,height)
i = 0
while(isOpened):
    if i == 10:
        break
    else:
        i = i+1
    (flag,frame) = cap.read()        # 读取每一张 flag frame 
    fileName = 'image'+str(i)+'.jpg'
    print(fileName)
    if flag == True:
        cv2.imwrite(fileName,frame,[cv2.IMWRITE_JPEG_QUALITY,100])
print('end!')

#2 图片合成视频
import cv2
img = cv2.imread('image1.jpg')
imgInfo = img.shape
size = (imgInfo[1],imgInfo[0])
print(size)
videoWrite = cv2.VideoWriter('2.mp4',-1,5,size) # 创建写入对象 编码器/帧率/size
for i in range(1,11):
    fileName = 'image'+str(i)+'.jpg'
    img = cv2.imread(fileName)
    videoWrite.write(img)                       # 写入图像
print('end!')
```

### SVM算法

```python
import cv2
import numpy as np

rand1 = np.array([[155,48],[159,50],[164,53],[168,56],[172,60]]) # train_data
rand2 = np.array([[152,53],[156,55],[160,56],[172,64],[176,65]])
data = np.vstack((rand1,rand2))
data = np.array(data,dtype='float32')
label = np.array([[0],[0],[0],[0],[0],[1],[1],[1],[1],[1]])      # train_label
# [155,48] -- 0 女生 
# [152,53] ---1  男生

svm = cv2.ml.SVM_create()        # 创建svm对象
svm.setType(cv2.ml.SVM_C_SVC)    # svm type
svm.setKernel(cv2.ml.SVM_LINEAR) # line
svm.setC(0.01)
# 训练
result = svm.train(data,cv2.ml.ROW_SAMPLE,label)
# 预测
pt_data = np.vstack([[167,55],[162,57]])    #0 女生 1男生
pt_data = np.array(pt_data,dtype='float32')
print(pt_data)
(par1,par2) = svm.predict(pt_data)
print(par2)
```