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
