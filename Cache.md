## lnstruction Tightly Coupled Memory and Data Tightly Coupled Memory
  `ITCM`和`DTCM`其实都是`SRAM`构成的存储，但是有独立的地址空间，所以可以直接访问这段空间获取数据，从而在嵌入式系统达到低延迟和低功耗的效果。
## 读L1 Cache有感
### 小Cache设计
  之前设计的`CPU`中包含`L1 Cache`和`TLB`，加入`TLB`之后直接将取指令的过程增加了一个周期，是不合理的。
搜索到`cyy`的`blog`, 谈到了相关的内容，在这里记录一下笔记。  
  通常我们在访问物理地址前，需要进行虚拟地址到物理地址的转化，于是必须先查`TLB`进行地址翻译，在将查到的地址送入到
`Cache`中，于是增加了访问的延迟。`L1`往往在处理器频率的关键路径上，必须进行优化。
  于是我们可以采用`VIPT(Virtually Indexed Physically Tagged)`，通过虚拟地址的一部分作为进行索引，这样`TLB`和`Cache`
的访问就可以并行，这样在`Cache`访问得到了`Tag`标记和可以`TLB`中翻译的结果进行比较，判断是否`Cache`命中。  
在同一进程中虚拟地址肯定是不会重复的，采用虚拟地址作为`index`可以和采用物理地址作为`index`起到一样的效果。  
但是在不同进程中如果存在两个进程使用相同的物理地址，但是虚拟地址不同，可能会有缓存一致性的问题。这个问题
在大部分的处理器中并没有得到解决，但是如果我们保证`VI`的部分在`page offset`内，那么`VT`就会和`PI`保持一致。
![](https://raw.githubusercontent.com/PorterLu/picgo/main/VIPT.webp)
  但是引入的问题是`L1 Cache`中一个`Way`的`Cache`大小只能小于等于`4KB`, 为了提高`Cache`容量可以
采用多路`Cache`，但是多路的`Cache`, 如果路数过多也会带来延迟。
### 超大容量Cache
#### 香山处理器
  在`L1 Cache`采用`VIPT`的情况下，由`L2`来维护`L1`中存在的别名。当`L1`向`L2`获取一个缓存行时，需要
向`L2`传递自己的虚拟地址位，如果这部分内容和`L2`中的记录不符的时候，`L2`就会向`L1`发送`Probe`请求
来请求已经失效的缓存行，来完成脏数据的写回。
#### IBM z14
  `IBM z14`设计了一个`VIVT`的目录，

##L1 Cache实验
### Linux中的共享内存

​	`Linux`系统在运行时会在多个进程之间交换数据，共享内存是通过`tmpfs`这个文件系统实现的，`tmpfs`文件系统的目录为`/dev/shm`，`/dev/shm`是驻留在内存中，因此读写的速度可以和内存一致，`/dev/shm`默认的大小为系统内存的一半，只用在使用`shm_open`时，`/dev/shm`才会真正地占用内存。

```c
int shm_open(const char* name, int oflag, mode_t mode);
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
int munmap(void *addr, size_t length);
int shm_unlink(const char *name);
int ftruncate(int fd, off_t length);
```

* `shm_open` ，用于创建或者打开共享内存文件。和一般文件`open`的区别在于，`shm_open`操作的文件一定是位于`tmpfs`文件系统里面的。
* `mmap`， 用于将文件映射到内存中。
* `munmap`，用于取消一段内存映射。
* `shm_unlink`, 用于删除`shm_open`创建的文件。
* `ftruncate`，用于重置文件的大小。

### 测试

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>

int main() {
    int fd = shm_open("test-shm", O_RDWR | O_CREAT, S_IRUSR | S_IWUSR);
    if (fd == -1) {
        printf("Unable to open shm\n");
        exit(1);
    }
    
    size_t size = 8192;
    ftruncate(fd, size);
    int size_int = size / 4;
    int *a = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    int *b = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    printf("a=%lx\n", a);
    printf("b=%lx\n", b);
    for (int i=0; i<size_int; i++) {
        a[i] = b[i] = 0;
    }
    clock_t t1 = clock();
    for (int j=0; j<1024*1024;j++) for (int i=0; i<size_int; i++) a[i] = b[i];
    clock_t t2 = clock();
    for (int j=0; j<1024*1024;j++) for (int i=0; i<size_int; i++) a[i] = a[i];
    clock_t t3 = clock();
    printf("t2-t1 = %ld\n", t2-t1);
    printf("t3-t2 = %ld\n", t3-t2);
    shm_unlink("test-shm");
    return 0;
}
```

![](https://raw.githubusercontent.com/PorterLu/picgo/main/cache_alias_test.png)

​	这里`a`和`b`存在`alias`, 在锐龙`5800h`，可以发现`alias`不会使得性能下降。

[浅谈现代处理器实现超大L1 Cache的方式 – 属于CYY自己的世界 (cyyself.name)](https://blog.cyyself.name/why-the-big-l1-cache-is-so-hard/)

