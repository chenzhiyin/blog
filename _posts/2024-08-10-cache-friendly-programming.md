---
title: 性能分析与优化：Locality of CPU Cache
date: 2024-08-10 20:01:40 +0800
categories: [Performance Optimization]
tags: [optimization, cache]
math: true
---
缓存的局部性原理是计算机科学中的一个重要概念，它描述了一个系统在处理数据时，某些数据项会比其他的更频繁地被访问的现象。局部性原理是基于这样一个观察：计算机程序倾向于重复访问相同的数据集或者邻近的数据集。局部性原理主要分为两种类型：

* **时间局部性（Temporal Locality）**：如果一个数据项被访问，那么它在不久的将来很可能会被再次访问。例如，循环中使用的变量就表现出很强的时间局部性，因为它们在每次迭代时都会被使用。

* **空间局部性（Spatial Locality）**：如果一个数据项被访问，那么它附近的数据项也很可能会在不久的将来被访问。这通常是因为程序倾向于按照数据在内存中的顺序进行访问，如数组遍历。

缓存（Cache）是一种小而快速的存储器，它存储最近或经常访问的数据项，以减少访问主存储器（通常是RAM）的次数。缓存的设计利用了局部性原理，通过保留近期访问过的数据（时间局部性）和相邻的数据（空间局部性），来提高数据访问速度和整体系统性能。

当程序运行时，缓存会尝试预测哪些数据将会被需要，并将这些数据加载到缓存中。如果预测正确，那么当CPU需要这些数据时，它可以直接从缓存中快速获取，而不是从较慢的主存储器中获取。这种预测的准确性直接影响程序的性能。

缓存的局部性原理不仅适用于CPU和内存之间的交互，也适用于其他层次的存储系统，如硬盘与RAM之间，甚至是网络存储环境中。理解和利用局部性原理可以帮助设计更高效的计算机系统和算法。

缓存局部性原理在性能优化中的应用非常广泛，它可以帮助设计更高效的算法、数据结构和系统架构。以下是一些具体的应用场景：

1. **算法优化**：
    * *循环交换*：在多层嵌套循环中，通过改变循环的顺序（例如，循环展开）来增强空间局部性，使得数据访问更加连续。
    * *分块（Blocking）*：在处理大型矩阵或数组时，将数据分成小块来处理，以确保当前处理的数据块能够适合于缓存，从而减少缓存未命中（cache misses）。

2. **数据结构选择**：
    * *线性数据结构*：如数组，由于其连续的内存布局，具有很好的空间局部性，适合顺序访问。
    * *树结构优化*：如B树，相比于二叉树，B树更加宽，减少了树的深度，提高了节点的利用率，增强了空间局部性。

3. **系统架构设计**：
    * *多级缓存*：现代计算机通常具有多级缓存（L1, L2, L3等），每一级缓存的大小和速度不同，设计时会考虑不同级别的局部性原理。
    * *预取策略（Prefetching）*：基于局部性原理，系统可以预测哪些数据将会被访问，并提前将其加载到缓存中。在程序中，也可**调用prefetch指令显式告知CPU进行预取**。

4. **编译器优化**（如**PGO**技术）：
    * *代码重排*：编译器可以重新安排指令的顺序，以减少缓存行的冲突和提高缓存的利用率。
    * *数据对齐*：编译器可以对数据进行对齐，以确保数据结构的布局符合缓存行的边界，从而提高访问效率。

5. **操作系统层面**：
    * *页面置换算法*：操作系统在管理虚拟内存时，会使用到局部性原理来决定哪些页面应该保留在物理内存中。
    * *文件系统设计*：文件系统可以将相关数据和元数据放置在相邻的位置，以提高磁盘I/O的局部性。

6. **数据库系统**：
    * *索引优化*：数据库索引（如B+树）的设计利用了空间局部性，以减少磁盘I/O操作。
    * *查询优化*：数据库查询优化器会考虑数据的局部性来优化查询计划，例如，通过选择合适的连接顺序。

在进行性能优化时，理解和应用缓存局部性原理是至关重要的。通过减少缓存未命中和提高数据访问效率，可以显著提升软件和硬件系统的性能。
下面重点介绍基于**循环交换**和**分块**技术的性能优化与分析。
## **Loop Interchange（循环交换）**

* v0
```c
#define N (1024*4)  // 4K
double a[N][N];
double b[N][N];
double c[N][N];
void cal(){
  for (int i = 0; i < N; i++) {
    for (int j = 0; j < N; j++) {
      for (int k = 0; k < N; k++) {
        c[i][j] += a[i][k] * b[k][j];
      }
    }
  }
}
```

* v1
```c
#define N (1024*4)  // 4K
double a[N][N];
double b[N][N];
double c[N][N];
void cal(){
  for (int i = 0; i < N; i++) {
    for (int k = 0; k < N; k++) { // loop interchage k<->j
      for (int j = 0; j < N; j++) {
        c[i][j] += a[i][k] * b[k][j];
      }
    }
  }
}
```

### Performance Result
* v0

```shell
time ./v0

real    3m30.171s
user    3m29.790s
sys     0m0.121s
```

* v1

```shell
time ./v1

real    2m8.837s
user    2m8.564s
sys     0m0.107s
```

### Performance Analysis
* v0

```shell
perf stat -e L1-dcache-load-misses,L1-dcache-loads,L1-dcache-stores -- ./v0

Performance counter stats for './v0':

    66,570,520,104      L1-dcache-load-misses:u     #    7.45% of all L1-dcache accesses   (50.00%)
   893,150,900,682      L1-dcache-loads:u                                                  (62.50%)
   137,438,779,571      L1-dcache-stores:u                                                 (62.50%)


     210.979498703 seconds time elapsed
```

* v1

```shell
perf stat -e L1-dcache-load-misses,L1-dcache-loads,L1-dcache-stores -- ./v1

Performance counter stats for './v1':

        96,156,351      L1-dcache-load-misses:u     #    0.01% of all L1-dcache accesses   (50.00%)
   897,451,304,512      L1-dcache-loads:u                                                  (62.50%)
   137,378,122,931      L1-dcache-stores:u                                                 (62.50%)

     134.383241366 seconds time elapsed
```

可以发现，交换循环层次的次序后，改善代码的**空间局部性**。该代码片段性能有超过50%的提升。

原因在于，v0中，最内层次的循环k表示二维数组b的列下标，因此相邻两次计算中，数组b中参与计算的元素是不连续的，内存地址相差8*1024*4个字节。v1中，最内层次的循环j表示二维数组a和c的行下标。在相邻两次的计算中，c[i][j]/c[i][j+1], b[k][j]/b[k][j+1]的内存地址都是连续的。更符合缓存空间局部性原理。因此性能更好。

实际上，CPU的L1 cache line是以64B为最小单位来管理缓存的。这样，当某次“c[i][j] += a[i][k] * b[k][j]”计算造成了CPU L1 cache miss后，相应的数据片段（64字节）会被load到cache中，这样下次计算如果用到数据的内存地址是连续的，就不会再发生cache miss。所以，可以看到，优化后的v1版本L1-DCACHE-LOAD-MISS发生次数减少了99.9%以上。这就是为什么v1版本最终性能有如此大的提高。

## **Blocking（分块）**
* v0
```c
#define N (1024*4)  // 4K
double a[N][N];
double b[N][N];
void cal(){
  for (int i = 0; i < N; i++) {
    for (int j = 0; j < N; j++) {
      a[i][j] += b[j][i];
    }
  }
}
```

* v1
```c
#define N (1024*4)  // 4K
double a[N][N];
double b[N][N];
double c[N][N];
void cal(){
  for (int ii = 0; ii < N; ii += 8) {
    for (int jj = 0; jj < N; jj += 8) {
      int i_N = ii + 8;
      int j_N = jj + 8;
      // traverse in 8*8 blocks
      for (int i = ii; i < i_N; i++) {
        for (int j = jj; j < j_N; j++) {
          a[i][j] += b[j][i];
        }
      }
    }
  }
}
```

对以上代码重复调用cal函数100次。

### Performance Result
* v0

```shell
time ./v0

real    0m5.493s
user    0m5.383s
sys     0m0.084s
```

* v1

```shell
time ./v1

real    0m2.878s
user    0m2.793s
sys     0m0.073s
```

### Performance Analysis
* v0

```shell
perf stat -e L1-dcache-load-misses,L1-dcache-loads,L1-dcache-stores -- ./v0

 Performance counter stats for './v0':

     1,723,322,276      L1-dcache-load-misses:u          #   10.27% of all L1-dcache accesses
    16,778,593,463      L1-dcache-loads:u
     3,356,279,566      L1-dcache-stores:u

       5.475642130 seconds time elapsed

```

* v1

```shell
perf stat -e L1-dcache-load-misses,L1-dcache-loads,L1-dcache-stores -- ./v1

 Performance counter stats for './v1':

       215,813,155      L1-dcache-load-misses:u          #    1.08% of all L1-dcache accesses
    19,897,029,252      L1-dcache-loads:u
     3,879,850,766      L1-dcache-stores:u

       2.861752781 seconds time elapsed

```
在本例中，通过对数组切片，来保证了数据访问的**时间局部性**，从而减少cache的miss次数。

虽然b[j][i]和b[j+1][i]不在同一条cacheline上，但是b[j][i]不会立即从L1 Data Cache中淘汰，只要b[j][i+1]计算和b[j][i]计算的时间相近（取决与CPU L1 DATA CACHE大小），b[j][i]相应的cache line就会在b[j][i+1]计算时，被CPU使用。因此本例通过切片，缩短了同一cache line上数据被CPU使用的距离。

时间局部性也被称为"Reuse Distances" (<https://easyperf.net/blog/2024/02/12/Memory-Profiling-Part5>)

实际上，除了基于L1 DATA CACHE的优化，基于L1 INSTRUCTION CACHE也有优化措施。例如在量化场景，为了能够减少交易决策执行的时延，可以周期性地调用关键代码，使其保留在cache中。这样，当真正需要成交时，能够降低相关代码的执行时延。
