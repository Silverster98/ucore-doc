# 工具使用

[参考:清华大学 ucore 实验指导书](https://chyyuu.gitbooks.io/ucore_os_docs/content/)

## 1.gcc 简单语法
使用 gcc 编译一段 hello world 程序；
```
#include <stdio.h>
int
main(void)
{
    printf("Hello, world!\n");
    return 0;
}
```

```
gcc -Wall hello.c -o hello
```

- -Wall 参数指出编译器开启所有常用警告。
- -o 参数指出将 .c 文件编译成机器码并保持在可执行文件 hello 中。

执行该可执行文件：
```
./hello
```

- ./ 代指当前目录

## 2. AT&T 汇编语法
x86 汇编有两种语法，一种是 AT&T，一种是 Intel。

ucore 中使用的是 AT&T 语法，下面列出二者不同点。

```
    * 寄存器命名原则
        AT&T: %eax                      Intel: eax
    * 源/目的操作数顺序 
        AT&T: movl %eax, %ebx           Intel: mov ebx, eax
    * 常数/立即数的格式　
        AT&T: movl $_value, %ebx        Intel: mov eax, _value
      把value的地址放入eax寄存器
        AT&T: movl $0xd00d, %ebx        Intel: mov ebx, 0xd00d
    * 操作数长度标识 
        AT&T: movw %ax, %bx             Intel: mov bx, ax
    * 寻址方式 
        AT&T:   immed32(basepointer, indexpointer, indexscale)
        Intel:  [basepointer + indexpointer × indexscale + imm32]
```

## 3.gcc 基本内联汇编
gcc 提供两类内联汇编：基本内联汇编语句和扩展内联汇编语句

内联汇编语句格式很简单，如下：
```
asm("statements");
```

例如：
```
asm("nop"); asm("cli");
```

"asm" 和 "\_\_asm\_\_" 的含义是完全一样的。如果有多行汇编，则每一行都要加上 "\n\t"。其中的 “\n” 是换行符，"\t” 是 tab 符，在每条命令的 结束加这两个符号，是为了让 gcc 把内联汇编代码翻译成一般的汇编代码时能够保证换行和留有一定的空格。对于基本asm语句，GCC编译出来的汇编代码就是双引号里的内容。例如：
```
    asm("pushl %eax\n\t"
        "movl $0,%eax\n\t"
        "popl %eax"
    );
```

对于如下程序：由于我们在内联汇编中改变了 edx 和 ebx 的值，但是由于 gcc 的特殊的处理方法，即先形成汇编文件，再交给 GAS 去汇编，所以 GAS 并不知道我们已经改变了 edx和 ebx 的值，如果程序的上下文需要 edx 或 ebx 作其他内存单元或变量的暂存，就会产生没有预料的多次赋值，引起严重的后果。对于变量 _boo也存在一样的问题。为了解决这个问题，就要用到扩展 GCC 内联汇编语法。
```
    asm("movl %eax, %ebx");
    asm("xorl %ebx, %edx");
    asm("movl $0, _boo);
```

## 4.gcc 扩展内联汇编
GCC扩展内联汇编的基本格式是：
```
asm [volatile] ( Assembler Template
   : Output Operands
   [ : Input Operands
   [ : Clobbers ] ])
```
其中，\_\_asm__ 表示汇编代码的开始，其后可以跟 \_\_volatile__（这是可选项），其含义是避免 “asm” 指令被删除、移动或组合，在执行代码时，如果不希望汇编语句被 gcc 优化而改变位置，就需要在 asm 符号后添加 volatile 关键词：asm volatile(...)；或者更详细地说明为：\_
\_asm__ \_\_volatile__(...)；然后就是小括弧，括弧中的内容是具体的内联汇编指令代码。 "" 为汇编指令部分，例如，"movl %%cr0,%0\n\t"。数字前加前缀 “％“，如％1，％2等表示使用寄存器的样板操作数。可以使用的操作数总数取决于具体CPU中通用寄存器的数量，如Intel可以有8个。指令中有几个操作数，就说明有几个变量需要与寄存器结合，由gcc在编译时根据后面输出部分和输入部分的约束条件进行相应的处理。由于这些样板操作数的前缀使用了”％“，因此，在用到具体的寄存器时就在前面加两个“％”，如%%cr0。输出部分（output operand list），用以规定对输出变量（目标操作数）如何与寄存器结合的约束（constraint）,输出部分可以有多个约束，互相以逗号分开。每个约束以“＝”开头，接着用一个字母来表示操作数的类型，然后是关于变量结合的约束。

基本示例如下：
```
#define read_cr0() ({ \
unsigned int __dummy; \
__asm__( \
    "movl %%cr0,%0\n\t" \
    :"=r" (__dummy)); \
__dummy; \
})
```

```
:"=r" (__dummy)
```
“＝r”表示相应的目标操作数（指令部分的%0）可以使用任何一个通用寄存器，并且变量__dummy 存放在这个寄存器中.

输入部分（input operand list）：输入部分与输出部分相似，但没有“＝”。如果输入部分一个操作数所要求使用的寄存器，与前面输出部分某个约束所要求的是同一个寄存器，那就把对应操作数的编号（如“1”，“2”等）放在约束条件中。在后面的例子中，可看到这种情况。修改部分（clobber list,也称 乱码列表）:这部分常常以“memory”为约束条件，以表示操作完成后内存中的内容已有改变，如果原来某个寄存器的内容来自内存，那么现在内存中这个单元的内容已经改变。乱码列表通知编译器，有些寄存器或内存因内联汇编块造成乱码，可隐式地破坏了条件寄存器的某些位（字段）。 注意，指令部分为必选项，而输入部分、输出部分及修改部分为可选项，当输入部分存在，而输出部分不存在时，冒号“：”要保留，当“memory”存在时，三个冒号都要保留，例如
```
#define __cli() __asm__ __volatile__("cli": : :"memory")
```

下面是一个例子：
```
int count=1;
int value=1;
int buf[10];
void main()
{
    asm(
        "cld \n\t"
        "rep \n\t"
        "stosl"
    :
    : "c" (count), "a" (value) , "D" (buf)
    );
}
```

得到主要汇编代码为(实际有好多，看不懂)：
```
movl count,%ecx
movl value,%eax
movl buf,%edi
#APP
cld
rep
stosl
#NO_APP
```

cld,rep,stos这几条语句的功能是向buf中写上count个value值。冒号后的语句指明输入，输出和被改变的寄存器。通过冒号以后的语句，编译器就知道你的指令需要和改变哪些寄存器，从而可以优化寄存器的分配。其中符号"c"(count)指示要把count的值放入ecx寄存器。类似的还有：
```
a eax
b ebx
c ecx
d edx
S esi
D edi
I 常数值，(0 - 31)
q,r 动态分配的寄存器
g eax,ebx,ecx,edx或内存变量
A 把eax和edx合成一个64位的寄存器(use long longs)
```

也可以让gcc自己选择合适的寄存器。如下面的例子：
```
asm("leal (%1,%1,4),%0"
    : "=r" (x)
    : "0" (x)
);
```

这段代码到的主要汇编代码为：
```
movl x,%eax
#APP
leal (%eax,%eax,4),%eax
#NO_APP
movl %eax,x
```

几点说明：
- [1] 使用q指示编译器从eax, ebx, ecx, edx分配寄存器。 使用r指示编译器从eax, ebx, ecx, edx, esi, edi分配寄存器。
- [2] 不必把编译器分配的寄存器放入改变的寄存器列表，因为寄存器已经记住了它们。
- [3] "="是标示输出寄存器，必须这样用。
- [4] 数字%n的用法：数字表示的寄存器是按照出现和从左到右的顺序映射到用"r"或"q"请求的寄存器．如果要重用"r"或"q"请求的寄存器的话，就可以使用它们。
- [5] 如果强制使用固定的寄存器的话，如不用%1，而用ebx，则：
```
asm("leal (%%ebx,%%ebx,4),%0"
    : "=r" (x)
    : "0" (x) 
);
```

## 5.make 和 makefile
GNU make(简称make)是一种代码维护工具，在大中型项目中，它将根据程序各个模块的更新情况，自动的维护和生成目标代码。

makefile 规则：
```
target ... : prerequisites ...
    command
    ...
    ...
```

target也就是一个目标文件，可以是object file，也可以是执行文件。还可以是一个标签（label）。prerequisites就是，要生成那个target所需要的文件或是目标。command也就是make需要执行的命令（任意的shell命令）。 这是一个文件的依赖关系，也就是说，target这一个或多个的目标文件依赖于prerequisites中的文件，其生成规则定义在 command中。如果prerequisites中有一个以上的文件比target文件要新，那么command所定义的命令就会被执行。这就是makefile的规则。也就是makefile中最核心的内容。

demo：
```
edit : main.o kbd.o command.o display.o \
       insert.o search.o files.o utils.o
        cc -o edit main.o kbd.o command.o display.o \
                   insert.o search.o files.o utils.o

main.o : main.c defs.h
        cc -c main.c
kbd.o : kbd.c defs.h command.h
        cc -c kbd.c
command.o : command.c defs.h command.h
        cc -c command.c
display.o : display.c defs.h buffer.h
        cc -c display.c
insert.o : insert.c defs.h buffer.h
        cc -c insert.c
search.o : search.c defs.h buffer.h
        cc -c search.c
files.o : files.c defs.h buffer.h command.h
        cc -c files.c
utils.o : utils.c defs.h
        cc -c utils.c
clean :
        rm edit main.o kbd.o command.o display.o \
           insert.o search.o files.o utils.o
```

## gdb 使用
[参考](https://chyyuu.gitbooks.io/ucore_os_docs/content/lab0/lab0_2_3_3_gdb.html)

## c 语言复习
- inline 内联关键字。该关键字解决函数频繁调用时函数栈资源被消耗的问题。当使用该关键字后，在函数被调用时，编译器直接将函数替换为函数的语句，而不用调用函数（即直接调用的是函数代码段）。关键字inline 必须与函数定义体放在一起才能使函数成为内联，仅将inline 放在函数声明前面不起任何作用。使用限制： inline只适合涵数体内代码简单的函数数使用，不能包含复杂的结构控制语句例如while、switch，并且内联函数本身不能是直接递归函数(自己内部还调用自己的函数)。总结，在头文件中，定义一些比较简单的函数。
```
void Foo(int x, int y);
inline void Foo(int x, int y) // inline 与函数定义体放在一起
{
}
```

- static 关键字。静态局部变量使用static修饰符定义，即使在声明时未赋初值，编译器也会把它初始化为0。且静态局部变量存储于进程的全局数据区，即使函数返回，它的值也会保持不变。即它就相当于一个全局变量，如果在函数体内，也是全局的，更改它的值，再次调用，值不变。并且仅在当前文件内有效。

- 指针在加一个数的时候，结果应为该指针值 + sizeof(type)*num
```
int a;
int * p = &a;
p++;

假设 &a = 0xc0;
则   p = 0xc4; // sizoef(int) = 4
```

## GNU \_\_attribute__ 机制
[参考](https://blog.csdn.net/tabactivity/article/details/78558457)

GNU C 的一大特色就是__attribute__ 机制。\_\_attribute__ 可以设置函数属性（Function Attribute）、变量属性（Variable Attribute）和类型属性（Type Attribute）。其位置约束为：放于声明的尾部。\_\_attribute__ 语法格式为:  \_\_attribute__ ((attribute-list))

- \_\_attribute__((always_inline)) 总是内联
- #define __noinline \_\_attribute\_\_((noinline)) 定义了一个宏，不内联。

