---
title: CUDA学习(三)
date: 2024-02-21 19:03:11
tags:
categories: CUDA
---

这一章将介绍线程块以及线程之间的通信机制和同步机制。

在GPU启动并行代码的实现方法是告诉CUDA运行时启动核函数的多个并行副本。我们将这些并行副本称为线程块(Block)。

CUDA运行时将把这些线程块分解为多个线程。当需要启动多个并行线程块时，只需将尖括号中的第一个参数由1改为想要启动的线程块数量。

在尖括号中，第二个参数表示CUDA运行时在每个线程块中创建的线程数量。假设尖括号中的变量为<<<N, M>>>总共启动的线程数量可以按照以下公式计算:
$$
N个线程块 * M个线程/线程块 = N*M个并行线程
$$
<!-- more -->

### 使用线程实现GPU上的矢量求和

在之前的代码中，我们才去的时调用N个线程块，每个线程块对应一个线程`add<<<N, 1>>>(dev_a, dev_b, dev_c);`。

如果我们启动N个线程，并且所有线程都在一个线程块中，则可以表示为`add<<<1, N>>>(dev_a, dev_b, dev_c);`。此外，因为只有一个线程块，我们需要通过线程索引来对数据进行索引(而不是线程块索引)，需要将`int tid = blockIdx.x;`修改为`int tid = threadIdx.x;`



### 在GPU上对更长的矢量求和

对于启动核函数时每个线程块中的线程数量，硬件也进行了限制。具体来说，最大的线程数量不能超过设备树形结构中maxThreadsPerBlock域的值。对目前的GPU来说一个线程块最多有1024个线程。如果要通过并行线程对长度大于1024的矢量进行相加的话，就需要将线程与线程块结合起来才能实现。

此时，计算索引可以表示为:

```c
int tid = threadIdx.x + blockIdx.x * blockDim.x;
```

blockDim保存的事线程块中每一维的线程数量，由于使用的事一维线程块，因此只用到blockDim.x。

此外，gridDim是二维的，而blockDim是三维的。

假如我们使用多个线程块处理N个并行线程，每个线程块处理的线程数量为128，那样可以启动N/128个线程块。然而问题在于，当N小于128时，比如127，那么N/128等于0，此时将会启动0个线程块。所以我们希望这个除法能够向上取整。我们可以不用调用 ceil()函数，而是
