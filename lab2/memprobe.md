# 物理内存探测

内存探测，我个人的理解就是在 boot 阶段通过一定的方法（中断）探测整个系统中可以使用的内存块，以及内存块的一些信息。

同时将这些信息保存在 0x8000 这个物理地址地址处。

保存的数据结构如下：
```
struct e820map {
    int nr_map;  // 不太清楚这个值的作用，他最后写了一个$12345到这个里面
    struct {
        uint64_t addr;
        uint64_t size;
        uint32_t type;
    } __attribute__((packed)) map[E820MAX]; // 保存每一个物理块的信息，size = 20 Byte
};
```

这样，在0x8004处即是第一个物理块信息，依次0x8004+20是第二个，0x8004+40是第三个

内存探测使用中断的方式，是INT 15,具体中断参数如下：
- eax：e820h：INT 15的中断调用参数；
- edx：534D4150h (即4个ASCII字符“SMAP”) ，这只是一个签名而已；
- ebx：如果是第一次调用或内存区域扫描完毕，则为0。 如果不是，则存放上次调用之后的计数值；
- ecx：保存地址范围描述符的内存大小,应该大于等于20字节；
- es:di：指向保存地址范围描述符结构的缓冲区，BIOS把信息写入这个结构的起始地址。

eax,edx,ebx,ecx初始化固定值之后就不用动了，然后di需要每次更新，因为每次写入一个内存映射地址描述符需要更改其起始地址。

具体探测代码如下，在bootasm.S中，相比lab1就增加这一部分代码：
```
probe_memory:
    movl $0, 0x8000     // 让e820map的nr_map = 0
    xorl %ebx, %ebx     // ebx初始置0
    movw $0x8004, %di   // di初始置0x8004,即第一个内存映射地址描述符起始地址
start_probe:
    movl $0xE820, %eax  // eax 初始0xe820
    movl $20, %ecx      // ecx 初始20，即内存映射地址描述符大小
    movl $SMAP, %edx    // edx固定SMAP值
    int $0x15           // 调用 INT 15 中断
    jnc cont            // 中断成功 eflags CF位置0，失败置1，CF=0也意味着还需继续探测
    movw $12345, 0x8000 // 置nr_map，探测出错
    jmp finish_probe    // 结束
cont:
    addw $20, %di       // 继续探测，di更新，作为下一个内存映射地址描述符起始地址
    incl 0x8000         // nr_map ++
    cmpl $0, %ebx       
    jnz start_probe     // 返回ebx是否为0，不为0意味着还需继续探测，为0就探测完毕
finish_probe:
```

## 物理内存分布
```
|-------------------------------< 空闲空间起始
|--------- 管理空闲区域 ---------|
|-------------------------------< ucore end
|----------- ucore -------------|
|-------------------------------< 0x00100000 (1M) ELFHDR -> e_entry
|-------- BIOS/空闲/CGA.. ------|
|-------------------------------< 0x00011000
|---- elf header 数据区(4K) ----|
|-------------------------------< 0x00010000 (64K)
|--------- bootloader ----------|
|-------------------------------< 0x00007c00
|--------- 系统堆栈区 -----------|
|-------------------------------< 0x00000000
```
