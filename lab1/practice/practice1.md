# lab1-practice1

## makefile 学习

[参考](https://www.cnblogs.com/wang_yb/p/3990952.html)

也太复杂了吧（小声bb）
### 1.概述
makefile 基本格式
```
target ... : prerequisites ...
    command
    ...
    ...
```

- target        - 目标文件, 可以是 Object File, 也可以是可执行文件
- prerequisites - 生成 target 所需要的文件或者目标
- command       - make需要执行的命令 (任意的shell命令), Makefile中的命令必须以 [tab] 开头

五个组成部分：
- 显示规则 :: 说明如何生成一个或多个目标文件(包- 括 生成的文件, 文件的依赖文件, 生成的命令)
- 隐晦规则 :: make的自动推导功能所执行的规则
- 变量定义 :: Makefile中定义的变量
- 文件指示 :: Makefile中引用其他Makefile; 指定Makefile中有效部分; 定义一个多行命令
- 注释     :: Makefile只有行注释 "#", 如果要使用或者输出"#"字符, 需要进行转义, "\#"

### 2.变量
变量定义可以使用 = 或者 :=

区别：
- = 既可以使用前面也可以使用后面定义的变量
- := 只能使用前面定义的变量

变量替换：
```
# Makefile内容
SRCS := programA.c programB.c programC.c
OBJS := $(SRCS:%.c=%.o)

all:
    @echo "SRCS: " $(SRCS)
    @echo "OBJS: " $(OBJS)

# bash中运行make
$ make
SRCS:  programA.c programB.c programC.c
OBJS:  programA.o programB.o programC.o
```

变量追加值使用 +=。

### 3.命令前缀
Makefile 中书写shell命令时可以加2种前缀 @ 和 -, 或者不用前缀.

3种格式的shell命令区别如下:

- 不用前缀 :: 输出执行的命令以及命令执行的结果, 出错的话停止执行
- 前缀 @   :: 只输出命令执行的结果, 出错的话停止执行
- 前缀 -   :: 命令执行有错的话, 忽略错误, 继续执行

### 4.伪目标
伪目标并不是一个"目标(target)", 不像真正的目标那样会生成一个目标文件.

典型的伪目标是 Makefile 中用来清理编译过程中中间文件的 clean 伪目标, 一般格式如下(.PHONY 关键字):
```
.PHONY: clean   <-- 这句没有也行, 但是最好加上
clean:
    -rm -f *.o
```

### 5.引用其他 makefile
语法: include <filename>  (filename 可以包含通`配符和路径)
```
# Makefile 内容
all:
    @echo "主 Makefile begin"
    @make other-all
    @echo "主 Makefile end"

include ./other/Makefile

########################################

# ./other/Makefile 内容
other-all:
    @echo "other makefile begin"
    @echo "other makefile end"
```

### 6.查看 c 文件依赖关系
使用 gcc -MM filename.c 命令

这样在写目标依赖时就不要来一个个查看依赖文件了。

### 7.隐含规则中的 命令变量 和 命令参数变量
变量名|含义
- | -
RM | rm -f
AR | ar
CC | cc
CXX | g++
```
# Makefile 内容
all:
    @echo $(RM)
    @echo $(AR)
    @echo $(CC)
    @echo $(CXX)
```

变量名|含义
-|-
ARFLAGS|AR命令的参数
CFLAGS|C语言编译器的参数
CXXFLAGS|C++语言编译器的参数
```
test: test.o
    $(CC) -o test test.o

$ make CFLAGS=-Wall             <-- 命令中加的编译器参数自动追加入下面的编译中了
cc -Wall   -c -o test.o test.c
cc -o test test.o
```

### 8.向引用的 makefile 传参
使用 expert 关键字
```
# Makefile 内容
export VALUE1 := export.c    <-- 用了 export, 此变量能够传递到 ./other/Makefile 中
VALUE2 := no-export.c        <-- 此变量不能传递到 ./other/Makefile 中

all:
    @echo "主 Makefile begin"
    @cd ./other && make
    @echo "主 Makefile end"


# ./other/Makefile 内容
other-all:
    @echo "other makefile begin"
    @echo "VALUE1: " $(VALUE1)
    @echo "VALUE2: " $(VALUE2)
    @echo "other makefile end"
```

### 9.定义命令包
命令包有点像是个函数, 将连续的相同的命令合成一条, 减少 Makefile 中的代码量
```
define <command-name>
command
...
endef
```

### 10.条件判断
条件判断的关键字主要有 ifeq ifneq ifdef ifndef

例如：（其他同理）
```
# Makefile 内容
SRCS := program.c

all:
ifdef SRCS
    @echo $(SRCS)
else
    @echo "no SRCS"
endif

# bash 中执行 make
$ make
program.c
```

### 11.makefile 自带函数

### 12.约定的伪目标
伪目标|含义
-|-
all|所有目标的目标，其功能一般是编译所有的目标
clean|删除所有被make创建的文件
install|安装已编译好的程序，其实就是把目标可执行文件拷贝到指定的目录中去
print|列出改变过的源文件
tar|把源程序打包备份. 也就是一个tar文件
dist|创建一个压缩文件, 一般是把tar文件压成Z文件. 或是gz文件
TAGS|更新所有的目标, 以备完整地重编译使用
check 或 test|一般用来测试makefile的流程

### 13.other
.SUFFIXES: 估计依赖文件后缀

.DELETE_ON_ERROR: make出错删除生成文件

### 14.call 语法
call - 创建新的参数化函数
语法:
```
$(call <expression>,<parm1>,<parm2>,<parm3>...)
```

示例：
```
# Makefile 内容 $(1)是传入的第一个参数
log = "====debug====" $(1) "====end===="

all:
    @echo $(call log,"正在 Make")

# bash 中执行 make
$ make
====debug==== 正在 Make ====end====
```

## 通道
0 是一个文件描述符，表示标准输入(stdin)
1 是一个文件描述符，表示标准输出(stdout)
2 是一个文件描述符，表示标准错误(stderr)

```
[root@redhat box]# ls 
a.txt 
[root@redhat box]# ls a.txt b.txt 
ls: b.txt: No such file or directory 由于没有b.txt这个文件, 于是返回错误值, 这就是所谓的2输出 
a.txt 而这个就是所谓的1输出
```

使用 > 就是重定向
```
[root@redhat box]# ls a.txt b.txt 1>file.out 2>file.err 
执行后,没有任何返回值. 原因是, 返回值都重定向到相应的文件中了,而不再前端显示 
[root@redhat box]# cat file.out 
a.txt 
[root@redhat box]# cat file.err 
ls: b.txt: No such file or directory
```

& 是一个描述符，如果1或2前不加&，会被当成一个普通文件。

- 1>&2 意思是把标准输出重定向到标准错误.

- 2>&1 意思是把标准错误输出重定向到标准输出。

- &>filename 意思是把标准输出和标准错误输出都重定向到文件filename中

## dd 命令
[参考](https://blog.csdn.net/qq_33160790/article/details/77488160)

dd：用指定大小的块拷贝一个文件，并在拷贝的同时进行指定的转换。

参数注释：

1. if=文件名：输入文件名，缺省为标准输入。即指定源文件。< if=input file >
2. of=文件名：输出文件名，缺省为标准输出。即指定目的文件。< of=output file >
3. ibs=bytes：一次读入bytes个字节，即指定一个块大小为bytes个字节。
    obs=bytes：一次输出bytes个字节，即指定一个块大小为bytes个字节。
    bs=bytes：同时设置读入/输出的块大小为bytes个字节。
4. cbs=bytes：一次转换bytes个字节，即指定转换缓冲区大小。
5. skip=blocks：从输入文件开头跳过blocks个块后再开始复制。
6. seek=blocks：从输出文件开头跳过blocks个块后再开始复制。
注意：通常只用当输出文件是磁盘或磁带时才有效，即备份到磁盘或磁带时才有效。
7. count=blocks：仅拷贝blocks个块，块大小等于ibs指定的字节数。
8. conv=conversion：用指定的参数转换文件。
    ascii：转换ebcdic为ascii
    ebcdic：转换ascii为ebcdic
    ibm：转换ascii为alternate ebcdic
    block：把每一行转换为长度为cbs，不足部分用空格填充
    unblock：使每一行的长度都为cbs，不足部分用空格填充
    lcase：把大写字符转换为小写字符
    ucase：把小写字符转换为大写字符
    swab：交换输入的每对字节
    noerror：出错时不停止
    notrunc：不截短输出文件
    sync：将每个输入块填充到ibs个字节，不足部分用（NUL）字符补齐。

shell 特殊变量涵义：
[参考](https://blog.csdn.net/u011341352/article/details/53215180/)
变量|含义
-|-
$0|当前脚本的文件名
$n|传递给脚本或函数的参数。n 是一个数字，表示第几个参数。例如，第一个参数是$1，第二个参数是$2。
$#|传递给脚本或函数的参数个数。
$*|传递给脚本或函数的所有参数。
$@|传递给脚本或函数的所有参数。被双引号(" ")包含时，与 $* 稍有不同，下面将会讲到。
$?|上个命令的退出状态，或函数的返回值。一般情况下，大部分命令执行成功会返回 0，失败返回 1。
$$|当前Shell进程ID。对于 Shell 脚本，就是这些脚本所在的进程ID。

## 练习1.1答案
kernel.img 的生成需要提前生成两个文件，一个是 kernel，一个是 bootblock

- kernel 是 ucore 的内核部分
- bootblock 是硬盘第一个扇区部分即主引导扇区部分代码（包含 bootloader 的部分）


ucore.img 生成 makefile 代码：
```
# create ucore.img
UCOREIMG	:= $(call totarget,ucore.img) # 生成路径，UCOREIMG := bin/ucore.img

$(UCOREIMG): $(kernel) $(bootblock) # 生成 ucore.img这个目标，需要有两个依赖文件，一个是 kernel,一个是 bootblock
	$(V)dd if=/dev/zero of=$@ count=10000 # 复制一个空间（空空间）到目标文件（即ucore.img），大小为10000*512B。
	$(V)dd if=$(bootblock) of=$@ conv=notrunc # 将 bootblock 复制到 ucore.img第一个block
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc # 从第二个 block 开始复制 kernel 到 ucore.img
```

bootblock 生成代码：
```
bootfiles = $(call listf_cc,boot) # 获取 boot 目录下 .c .S 文件列表
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc)) # 由宏编译生成 .o 文件，这一块实际上由 eval 关键字，让一个宏变成一个 makefile 表达式即（target ... : prerequisites ...;command 的形式）。这样就生成 .o 文件

bootblock = $(call totarget,bootblock) # 生成路径 bootblock := bin/bootblock

$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock) # 生成 bootblock.o 文件
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJDUMP) -t $(call objfile,bootblock) | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock) # 此处将bootblock.o 复制给bootblock.out
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock) # 使用sign 工具处理 bootblock.out，生成 bootblock  具体命令是：bin/sign bootblock.out bin/bootblock
```

sign 生成代码：

同样，这段代码最后转换成一个 makefile 表达式，通过宏 + eval 生成，从而直接生成 sign
```
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)
```

kernel 生成代码：
```
kernel = $(call totarget,kernel) # 生成目标路径 bin/kernel

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS) # KOBJS 是 kernel 依赖的一堆 .o 文件
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS) # 连接生成 kernel target
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
```

更多细节没有看。只看了个大概。

## 练习1.2答案
sign.c 核心代码:
```
    char buf[512];
    memset(buf, 0, sizeof(buf));
    FILE *ifp = fopen(argv[1], "rb");
    int size = fread(buf, 1, st.st_size, ifp);
    ···
    fclose(ifp);
    buf[510] = 0x55;
    buf[511] = 0xAA;
    FILE *ofp = fopen(argv[2], "wb+");
    size = fwrite(buf, 1, 512, ofp);
    ···
```

sign.c编译成为 sign 工具，然后在编译生成 bootblock 的时候，首先是生成的 bootblock.out，然后使用 sign 工具，sign bootblock.out bootblock

sign 的主要作用就是将 bootblock.out 的内容复制到 bootblock 中，并且在其中进行大小检测。最后在512 Byte的最后两个字节写入 0x55AA。以保证向硬盘第一个扇区写入数据的规范性。

