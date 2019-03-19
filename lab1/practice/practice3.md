# lab1-practice3

## 开启 A20 Gate
####1.等待 input register 为空并发送向 output port 2写命令
```
seta20.1: 
    inb $0x64, %al      # 将0x64地址的 i/o input 结果保存到 al
    testb $0x2, %al     # 测试 al 第二位是否为0
    jnz seta20.1        # 如果 zf 标准位为 0 则跳转

    movb $0xd1, %al     
    outb %al, $0x64     # 向0x64 写入 0xd1 数据，即向outport port 发送写命令
```
上述即判断 8042 input register 是否为空，不为空则等待为空

8042 的 status register 说明：
bit|meaning
-|-
0|output register (60h) 中有数据
1|input register (60h/64h) 有数据
2|系统标志（上电复位后被置为0）
3|data in input register is command (1) or data (0)
4|1=keyboard enabled, 0=keyboard disabled (via switch)
5|1=transmit timeout (data transmit not complete)
6|1=receive timeout (data transmit not complete)
7|1=even parity rec'd, 0=odd parity rec'd (should be odd)

#### 2.等待 input register 为空并向 0x60写入数据置为 A20
```
seta20.2:
    inb $0x64, %al
    testb $0x2, %al
    jnz seta20.2     # 等待input register 为空

    movb $0xdf, %al
    outb %al, $0x60  # 向输入缓存写入 0xdf = 10111111
```

## 加载 GDT
```
    lgdt gdtdesc            # 加载 gdt 指令，gdtdesc 是一个指令
    movl %cr0, %eax         
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0         # 设置 cr0 的 pe 位为1.
```

gdtdesc 解析：
```
gdtdesc:
    .word 0x17              # 0x17 是 gdtr 中的16位限长字段，0x17是24个字节，即3个段描述符
    .long gdt               # gdt 是 gdt的基质
```

gdt 解析：
```
gdt:
    SEG_NULLASM             # 这是一个空的段描述符
    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff) 
    # 设置一个 代码段描述符，0x0 是基质，0xffffffff是段限长，且设置粒度为 页。且设置type 为可执行可读。
    SEG_ASM(STA_W, 0x0, 0xffffffff)       # 设置一个 数据段描述符。
```

ucore 是把整个32G地址空间设置为一个段。


SEG_ASM 宏解析：
```
#define SEG_ASM(type,base,lim)                                  \
    .word (((lim) >> 12) & 0xffff), ((base) & 0xffff);          \
    .byte (((base) >> 16) & 0xff), (0x90 | (type)),             \
        (0xC0 | (((lim) >> 28) & 0xf)), (((base) >> 24) & 0xff)
```
.word 是16位的字，第一行是两个字，在段描述符中是限长的0 ~ 15位，段基址的0 ~ 15位

.byte 是8位的字节，第二行是四个字节，第一个字节是段基址的16 ~ 23位，第二个字节是 P DPL S Type，第三个字节是 G D 0 AVL 以及限长的16 ~ 19位。第四个字节是段基址的24 ~ 31位。

## 切换至 32 bit 模式
```
ljmp $PROT_MODE_CSEG, $protcseg
```

这是一个长跳转指令，PROT_MODE_CSEG = 0x8,此时表明即将跳转的段选择子为 0x8 >> 3,即GDT中第一个段描述符所描述的段。 $protcseg 是该段代码位于段中的偏移地址。

## 设置寄存器
```
    movw $PROT_MODE_DSEG, %ax
    movw %ax, %ds               # 设置 ds 寄存器，（数据段选择子）
    movw %ax, %es               # 设置 es    
    movw %ax, %fs               # 设置 fs
    movw %ax, %gs               # 设置 gs 
    movw %ax, %ss               # 设置 ss

    movl $0x0, %ebp             # 设置基址指针寄存器
    movl $start, %esp           # 设置栈顶指针寄存器，从 $start 开始，即0x7c00,那么他所表示的范围是 0x0 ~ 0x7bff（栈向低地址方向增长）
    call bootmain               # 调用 c 语言编写的 boot 主代码。
```

倘若重 bootmain 中跳出了（出错）那么执行下列程序，即死循环。
```
spin:
    jmp spin
```

至此，cpu 切换至保护模式，并且开始在保护模式下执行 boot 主程序。



