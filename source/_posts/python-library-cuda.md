---
categories:
- - Python
date: 2024-04-17 14:55:15
description: python对CUDA扩展有不错的支持，CUDA通过大量线程的并行化可以大幅提高代码计算速度，一般python常用numba、pycuda套件来支持CUDA扩展。numba通过JIT编译器只需将numba装饰器应用到python函数中即可实现CUDA加速，而pycuda需要基于C/C++编写kernel，其移植性、直观性更佳，这里主要介绍pycuda的使用。
id: '132'
tags:
- python
title: Python常用库11-CUDA
---


python对CUDA扩展有不错的支持，CUDA通过大量线程的并行化可以大幅提高代码计算速度，一般python常用numba、pycuda套件来支持CUDA扩展。numba通过JIT编译器只需将numba装饰器应用到python函数中即可实现CUDA加速，而pycuda需要基于C/C++编写kernel，其移植性、直观性更佳，这里主要介绍pycuda的使用。

## 1.向量加法

示例使用了1个block，block中含有400个线程，每个线程计算向量加法最终结果的一个值。

```python
import numpy
import pycuda.autoinit
import pycuda.driver as cuda
from pycuda.compiler import SourceModule

mod = SourceModule("""
__global__ void vect_add(float *dest, float *a, float *b)
{
  const int i = threadIdx.x;
  dest[i] = a[i] + b[i];
}
""")

vect_add = mod.get_function("vect_add")

a = numpy.random.randn(400).astype(numpy.float32)
b = numpy.random.randn(400).astype(numpy.float32)

dest = numpy.zeros_like(a)
vect_add(cuda.Out(dest), cuda.In(a), cuda.In(b), block=(400,1,1), grid=(1,1))

print(dest-(a+b))
```

## 2.矩阵乘法

示例使用的是25x25个block，每个block中为16x16的线程，每个线程计算矩阵乘法最终结果的一个值。

```python
import numpy
import pycuda.autoinit
import pycuda.driver as cuda
from pycuda.compiler import SourceModule

mod = SourceModule("""
__global__ void matrix_mul(float *dest, float *a, float *b, int width)
{
    int i = threadIdx.x + blockDim.x * blockIdx.x;
    int j = threadIdx.y + blockDim.y * blockIdx.y;

    float sum = 0;
    for(int k=0;k<width;k++)
    {
        float a_k = a[j*width+k];
        float b_k = b[k*width+i];
        sum += a_k*b_k;
    }
    dest[j*width+i] = sum;
}
""")

matrix_mul = mod.get_function("matrix_mul")

a = numpy.random.randn(400, 400).astype(numpy.float32)
b = numpy.random.randn(400, 400).astype(numpy.float32)
dest = numpy.zeros_like(a)
width = numpy.int32(400)

matrix_mul(cuda.Out(dest), cuda.In(a), cuda.In(b), width, block=(16,16,1), grid=(25,25))

print(dest)
print(numpy.dot(a,b))
```

## 3.矩阵乘法优化

我们可以使用矩阵分块乘法对上面的矩阵乘法代码进行优化，例如将6x6的矩阵A切分成9个2x2的小块，以A\_11~A\_33命名每个小块，则对于矩阵乘法AxB=C（假设维度都为6x6），有A\_11xB\_11+A\_12xB\_21+A\_13xB\_31=C\_11，具体做法是将每个矩阵小块需要使用的数据先读取到共享内存中再进行矩阵小块的乘法，以减少对全局内存的访问，从而达到有效加速的目的。

```python
import numpy as np
import pycuda.autoinit
import pycuda.driver as cuda
from pycuda.compiler import SourceModule

mod = SourceModule("""
#define BLOCK_SIZE 16

typedef struct {
    int width;
    int height;
    int stride; 
    int __padding;    //为了和64位的elements指针对齐
    float* elements;
} Matrix;

// 读取矩阵元素
__device__ float GetElement(const Matrix A, int row, int col)
{
    return A.elements[row * A.stride + col];
}

// 赋值矩阵元素
__device__ void SetElement(Matrix A, int row, int col, float value)
{
    A.elements[row * A.stride + col] = value;
}

// 获取 16x16 的子矩阵
 __device__ Matrix GetSubMatrix(Matrix A, int row, int col) 
{
    Matrix Asub;
    Asub.width    = BLOCK_SIZE;
    Asub.height   = BLOCK_SIZE;
    Asub.stride   = A.stride;
    Asub.elements = &A.elements[A.stride * BLOCK_SIZE * row + BLOCK_SIZE * col];
    return Asub;
}

__global__ void matrix_mul(Matrix *A, Matrix *B, Matrix *C)
{
    int blockRow = blockIdx.y;
    int blockCol = blockIdx.x;
    int row = threadIdx.y;
    int col = threadIdx.x;

    Matrix Csub = GetSubMatrix(*C, blockRow, blockCol);

    // 每个线程通过累加Cvalue计算Csub的一个值
    float Cvalue = 0;

    // 为了计算Csub遍历所有需要的Asub和Bsub
    for (int m = 0; m < (A->width / BLOCK_SIZE); ++m) 
    {
        Matrix Asub = GetSubMatrix(*A, blockRow, m);
        Matrix Bsub = GetSubMatrix(*B, m, blockCol);

        __shared__ float As[BLOCK_SIZE][BLOCK_SIZE];
        __shared__ float Bs[BLOCK_SIZE][BLOCK_SIZE];

        As[row][col] = GetElement(Asub, row, col);
        Bs[row][col] = GetElement(Bsub, row, col);

        __syncthreads();

        for (int e = 0; e < BLOCK_SIZE; ++e)
            Cvalue += As[row][e] * Bs[e][col];

        __syncthreads();
    }

    SetElement(Csub, row, col, Cvalue);
}
""")

class MatrixStruct(object):
    def __init__(self, array):
        self._cptr = None

        self.shape, self.dtype = array.shape, array.dtype
        self.width = np.int32(self.shape[1])
        self.height = np.int32(self.shape[0])
        self.stride = self.width
        self.elements = cuda.to_device(array)                      # 分配内存并拷贝数组数据至device，返回其地址

    def send_to_gpu(self):
        self._cptr = cuda.mem_alloc(self.nbytes())                 # 分配一个C结构体所占的内存
        cuda.memcpy_htod(int(self._cptr), self.width.tobytes())    # 拷贝数据至device，下同
        cuda.memcpy_htod(int(self._cptr)+4, self.height.tobytes())
        cuda.memcpy_htod(int(self._cptr)+8, self.stride.tobytes())
        cuda.memcpy_htod(int(self._cptr)+16, np.intp(int(self.elements)).tobytes())

    def get_from_gpu(self):
        return cuda.from_device(self.elements, self.shape, self.dtype)  # 从device取回数组数据

    def nbytes(self):
        return self.width.nbytes * 4 + np.intp(0).nbytes

a = np.random.randn(400,400).astype(np.float32)
b = np.random.randn(400,400).astype(np.float32)
c = np.zeros_like(a)

A = MatrixStruct(a)
B = MatrixStruct(b)
C = MatrixStruct(c)
A.send_to_gpu()
B.send_to_gpu()
C.send_to_gpu()

matrix_mul = mod.get_function("matrix_mul")
matrix_mul(A._cptr, B._cptr, C._cptr, block=(16,16,1), grid=(25,25))
result = C.get_from_gpu()
print(np.dot(a,b))
print(result)
```