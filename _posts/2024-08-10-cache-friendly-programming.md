---
title: 性能分析与优化：更好地利用CPU Cache
date: 2024-08-10 20:01:40 +0800
categories: [Performance Optimization]
tags: [optimization, cache]
math: true
---
## Loop Interchange
### Code
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
```
time ./v0

real    3m30.171s
user    3m29.790s
sys     0m0.121s
```

* v1
```
time ./v1

real    2m8.837s
user    2m8.564s
sys     0m0.107s
```
### Performance Analysis
* v0
```
perf stat -e L1-dcache-load-misses,L1-dcache-loads,L1-dcache-stores -- ./v0

Performance counter stats for './v0':

    66,570,520,104      L1-dcache-load-misses:u          #    7.45% of all L1-dcache accesses   (50.00%)
   893,150,900,682      L1-dcache-loads:u                                                       (62.50%)
   137,438,779,571      L1-dcache-stores:u                                                      (62.50%)


     210.979498703 seconds time elapsed
```

* v1
```
perf stat -e L1-dcache-load-misses,L1-dcache-loads,L1-dcache-stores -- ./v1

Performance counter stats for './v1':

        96,156,351      L1-dcache-load-misses:u          #    0.01% of all L1-dcache accesses   (50.00%)
   897,451,304,512      L1-dcache-loads:u                                                       (62.50%)
   137,378,122,931      L1-dcache-stores:u                                                      (62.50%)

     134.383241366 seconds time elapsed
```

可以发现，交换循环层次的次序后，该代码片段性能有超过50%的提升。
原因在于，v0中，最内层次的循环k表示二维数组b的列下标，因此相邻两次计算中，数组b中参与计算的元素是不连续的，内存地址相差8*1024*4个字节。v1中，最内层次的循环j表示二维数组a和c的行下标。在相邻两次的计算中，c[i][j]/c[i][j+1], b[k][j]/b[k][j+1]的内存地址都是连续的。更符合缓存空间局部性原理。因此性能更好。
实际上，CPU的L1 cache line是以64B为最小单位来管理缓存的。这样，当某次“c[i][j] += a[i][k] * b[k][j]”计算造成了CPU L1 cache miss后，相应的数据片段（64字节）会被load到cache中，这样下次计算如果用到数据的内存地址是连续的，就不会再发生cache miss。所以，可以看到，优化后的v1版本L1-DCACHE-LOAD-MISS发生次数减少了99.9%以上。这就是为什么v1版本最终性能有如此大的提高。

## Tiling
### Code
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
```
time ./v0

real    0m5.493s
user    0m5.383s
sys     0m0.084s
```

* v1
```
time ./v1

real    0m2.878s
user    0m2.793s
sys     0m0.073s
```
### Performance Analysis
* v0
```
perf stat -e L1-dcache-load-misses,L1-dcache-loads,L1-dcache-stores -- ./v0

 Performance counter stats for './v0':

     1,723,322,276      L1-dcache-load-misses:u          #   10.27% of all L1-dcache accesses
    16,778,593,463      L1-dcache-loads:u
     3,356,279,566      L1-dcache-stores:u

       5.475642130 seconds time elapsed

```

* v1
```
perf stat -e L1-dcache-load-misses,L1-dcache-loads,L1-dcache-stores -- ./v1

 Performance counter stats for './v1':

       215,813,155      L1-dcache-load-misses:u          #    1.08% of all L1-dcache accesses
    19,897,029,252      L1-dcache-loads:u
     3,879,850,766      L1-dcache-stores:u

       2.861752781 seconds time elapsed

```
在本例中，通过对数组切片，来保证了数据访问的局部性，从而减少cache的miss次数。
