# lab1-practice4

## 扇区读取
读取一个扇区的代码如下：
```
static void
readsect(void *dst, uint32_t secno) {
    // wait for disk to be ready
    waitdisk();

    outb(0x1F2, 1);                         // count = 1
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors

    // wait for disk to be ready
    waitdisk();

    // read a sector
    insl(0x1F0, dst, SECTSIZE / 4);
}
```

以上代码通过对0x1f0-0x1f7地址进行相关读写，然后可以读取一个扇区数据，最终读取函数为 insl().

readsect() 函数是在 regsed() 函数中调用。regsed()函数的封装使得可以读取任意指定大小的硬盘空间。

## bootmain() 主函数
bootmain() 主函数：
```
void
bootmain(void) {
    // 读取第一个磁盘块（4K）空间
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    // 通过第一个字段判断是不是一个 elf 合法文件。
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

    struct proghdr *ph, *eph;

    // 加载 program header
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff); # header start address
    eph = ph + ELFHDR->e_phnum; # header end address
    for (; ph < eph; ph ++) { # 把 program header 加载置内存
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

    // 调用入口函数，即进入 ucore,这是一个无返回的函数，如果返回那么就出错
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();

bad: # 出错处理
    outw(0x8A00, 0x8A00);
    outw(0x8A00, 0x8E00);

    /* do nothing */
    while (1);
}
```