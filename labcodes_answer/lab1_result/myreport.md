# Makefile note
lab1_1 读makefile

## makefile快速入门
文章：https://seisman.github.io/how-to-write-makefile/overview.html

## 注意事项
一些没见过的或者是不清楚的写法

### = 和：=

> 1、“=”
> 
> make会将整个makefile展开后，再决定变量的值。也就是说，变量的值将会是整个makefile中最后被指定的值。看例子：
> 
>     x = foo
>     y = $(x) bar
>     x = xyz
> 
> 在上例中，y的值将会是 xyz bar ，而不是 foo bar 。
> 
> 2、“:=”
> 
> “:=”表示变量的值决定于它在makefile中的位置，而不是整个makefile展开后的最终值。
> 
>     x := foo
>     y := $(x) bar
>     x := xyz
> 
> 在上例中，y的值将会是 foo bar ，而不是 xyz bar 了。

### $(xxx )
函数调用，常见的函数调用和用法可以看这个文章：
https://blog.csdn.net/qq_31860135/article/details/84253289

这个makefile中频繁出现的有：
- `$(shell ...)`<br />
    和外部通信
- `$(call ...)`<br />
    调用一个自定义的函数
- `$(patsubst ...)`
    ```Makefile
    # 用来匹配pattern把text中的结果用replace替换。有点像replace函数。
    $(patsubst <pattern>,<replacement>,<text>)
    ```
- `$(addprefix ...)`
    ```Makefile
    $(addprefix <prefix>,<names...>)
    #名称：加前缀函数——addprefix。
    #功能：把前缀<prefix>加到<names>中的每个单词后面。
    #返回：返回加过前缀的文件名序列。
    #示例：$(addprefix src/,foo bar)返回值是“src/foo src/bar”。
    ```
- `$(filter ...)`
- `$(if ...)`
    ```Makefile
    $if(expr, stat, ...)
    # if expr then stat else ...
    ```
- `$(wildcard ...)`
    <br />扩展通配符：https://www.cnblogs.com/haoxing990/p/4629454.html
- `$(addsuffix ...)`
    <br/> 添加后缀
- `$(basename ...)`
    ```Makefile
    $(basename <names...>)
    #名称：取前缀函数——basename。
    #功能：从文件名序列<names>中取出各个文件名的前缀部分。
    #返回：返回文件名序列<names>的前缀序列，如果文件没有前缀，则返回空字串。
    #示例：$(basename src/foo.c src-1.0/bar.c hacks)返回值是“src/foo src-1.0/bar hacks”。
    ```
- `$(foreach ...)`
    ```Makefile
    names := a b c d
    files := $(foreach n,$(names),$(n).o)
    # output a.o b.o c.o d.o
    ```
- `$(sort ...)`
    <br />排序
- `$(dir ...)`
    <br />取目录
- `$(eval ...)`
    <br />执行text

### function.mk

- `listf`   用来查找$(1)目录下的所有$(2)后缀的文件，比如`$(call listf, ./, c S)`查找当前目录下的所有.c, .S 文件
- `toobj`   确定obj的绝对路径，其中$(OBJDIR)在Makefile中定义为`obj`
- `todep`   转换成dependency files,结果为`$(OBJDIR)/$(2)/$(1).d`
- `totarget`    转换成target文件，即bin目录下的
- `packetname`  给文件添加前缀`OBJPREFIX := __objs_`
- `cc_template` 
    ```Makefile
    # cc compile template, generate rule for dep, obj: (file, cc[, flags, dir])
    # $4：dir 目录
    # $3：flags 编译选项
    # $2：cc 编译命令
    # $1：file 依赖文件
    define cc_template
    $$(call todep,$(1),$(4)): $(1) | $$$$(dir $$$$@)
        @$(2) -I$$(dir $(1)) $(3) -MM $$< -MT "$$(patsubst %.d,%.o,$$@) $$@"> $$@
    $$(call toobj,$(1),$(4)): $(1) | $$$$(dir $$$$@)
        @echo + cc $$<
        $(V)$(2) -I$$(dir $(1)) $(3) -c $$< -o $$@
    ALLOBJS += $$(call toobj,$(1),$(4))
    endef
    ```
    函数中`$@`代表目标文件，`$<`代表第一个依赖文件,cc的参数中,`-I`指明了头文件目录，`-MM`表示log依赖关系（不包含标准库），`-MT`就是在log中的target的name.
    这个函数第一步生成了.d文件,第二步生成了.o文件

- `do_cc_compile`
    ```Makefile
    # compile file: (#files, cc[, flags, dir])
    define do_cc_compile
    $$(foreach f,$(1),$$(eval $$(call cc_template,$$(f),$(2),$(3),$(4))))
    endef
    ```
    对`$(1)`中的每一个项，调用`cc_template`,这个的结果就是全部的.o文件

- `do_add_files_to_packet`
- `do_add_objs_to_packet`
- `do_create_target`
- `do_finish_all`

还有一些封装
```Makefile
# compile file: (#files, cc[, flags, dir])
cc_compile = $(eval $(call do_cc_compile,$(1),$(2),$(3),$(4)))

# add files to packet: (#files, cc[, flags, packet, dir])
add_files = $(eval $(call do_add_files_to_packet,$(1),$(2),$(3),$(4),$(5)))

# add objs to packet: (#objs, packet)
add_objs = $(eval $(call do_add_objs_to_packet,$(1),$(2)))

# add packets and objs to target (target, #packes, #objs, cc, [, flags])
create_target = $(eval $(call do_create_target,$(1),$(2),$(3),$(4),$(5)))

read_packet = $(foreach p,$(call packetname,$(1)),$($(p)))

add_dependency = $(eval $(1): $(2))

finish_all = $(eval $(call do_finish_all))
```

## Makefile

首先是kernel，编译在bin目录下，使用了tools中的kernel.ld链接kernel.
然后编译了bootblock
接着生成了sign
最后创建了ucore.img

### kernel.ld
这是一个简单的ld文件，具体的语法可以看这个文章：
https://www.jianshu.com/p/42823b3b7c8e

主要信息就是，kernel的入口是`kern_init`，有`.text .rodata .stab stabstr .data .bss /DISCARD/`这几个节。discard表示丢弃了这几个节。
关于.eh_frame的说法，看起来是提供栈回溯的一些功能，可以丢掉：
https://lore.kernel.org/patchwork/patch/1158993/ <br />
https://stackoverflow.com/questions/26300819/why-gcc-compiled-c-program-needs-eh-frame-section

### bootblock
看一下这个<br />
`$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)` <br />
指定了入口点`start`，指定了一个地址0x7c00...再看看

### ucore.img
依赖是kernel和bootblock，然后使用`dd`指令，将bootblock和kernel分别放在第一个块和这个块后面。
就是说，ucore.img的结构简单的为：
——————————————————————————————————————
|           |                         |  
| bootblock |        kernel           |  
|           |                         |  
———————————————————————————————————————

## 问题
1.操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令 和命令参数的含义，以及说明命令导致的结果)
    见上面的分析。
2.一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？
    通过sign.c可以看出来，引导区是512字节，最后两个字节是0x55AA.至于为啥要用这个文件，可能是因为这个文件是打分用的？emmmmm

# GDB + QEMU
TODO

# bootloader

## 准备工作
bootloader被加载到`0x7c00`，此时`%cs=0 %ip=7c00`，可以用gdb断下来进行调试。

### bootasm.S

```asm
#include <asm.h>

# Start the CPU: switch to 32-bit protected mode, jump into C.
# The BIOS loads this code from the first sector of the hard disk into
# memory at physical address 0x7c00 and starts executing in real mode
# with %cs=0 %ip=7c00.

# 段选择子，数字是在GDT中的offset
.set PROT_MODE_CSEG,        0x8                     # kernel code segment selector
.set PROT_MODE_DSEG,        0x10                    # kernel data segment selector
# TODO
.set CR0_PE_ON,             0x1                     # protected mode enable flag

# start address should be 0:7c00, in real mode, the beginning address of the running bootloader  入口地址
.globl start
start:
.code16                                             # Assemble for 16-bit mode
    cli                                             # Disable interrupts
    cld                                             # String operations increment

    # Set up the important data segment registers (DS, ES, SS). 置零
    xorw %ax, %ax                                   # Segment number zero
    movw %ax, %ds                                   # -> Data Segment
    movw %ax, %es                                   # -> Extra Segment
    movw %ax, %ss                                   # -> Stack Segment

    # Enable A20:
    #  For backwards compatibility with the earliest PCs, physical
    #  address line 20 is tied low, so that addresses higher than
    #  1MB wrap around to zero by default. This code undoes this.

seta20.1:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al                                 # 如果 %al 第低2位为1，则ZF = 0, 则跳转
    jnz seta20.1                                    # 如果 %al 第低2位为0，则ZF = 1, 则不跳转

    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port

seta20.2:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1

    # 全局描述符表：存放8字节的段描述符，段描述符包含段的属性。
    # 段选择符：总共16位，高13位用作全局描述符表中的索引位，GDT的第一项总是设为0，
    #   因此孔断选择符的逻辑地址会被认为是无效的，从而引起一个处理器异常。GDT表项
    #   最大数目是8191个，即2^13 - 1.
    # Switch from real to protected mode, using a bootstrap GDT
    # and segment translation that makes virtual addresses
    # identical to physical addresses, so that the
    # effective memory map does not change during the switch.
    lgdt gdtdesc
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0

    # Jump to next instruction, but in 32-bit code segment.
    # Switches processor into 32-bit mode.
    ljmp $PROT_MODE_CSEG, $protcseg     # ?

.code32                                             # Assemble for 32-bit mode
protcseg:
    # Set up the protected-mode data segment registers
    movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
    movw %ax, %ds                                   # -> DS: Data Segment
    movw %ax, %es                                   # -> ES: Extra Segment
    movw %ax, %fs                                   # -> FS
    movw %ax, %gs                                   # -> GS
    movw %ax, %ss                                   # -> SS: Stack Segment

    # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
    movl $0x0, %ebp
    movl $start, %esp
    call bootmain

    # If bootmain returns (it shouldn't), loop.
spin:
    jmp spin

# Bootstrap GDT
.p2align 2                                          # force 4 byte alignment
gdt:
    SEG_NULLASM                                     # null seg
    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)           # code seg for bootloader and kernel
    SEG_ASM(STA_W, 0x0, 0xffffffff)                 # data seg for bootloader and kernel

gdtdesc:
    .word 0x17                                      # sizeof(gdt) - 1
    .long gdt                                       # address gdt

```
上面代码的注释非常清楚，结合指导书来看，没有什么不易理解的点。

如何从实模式转换到保护模式？
可以查看标准答案：[答案](./report.md). <br />
补充：
1. 开启A20的代码实际上只是把`0xdf - > 0x60`，seta20.1用来告知需要向P2 port写入， seta20.2用来写0xdf。
P2的端口的定义和使用方法都在指导书中给出了，请看[A20](https://objectkuan.gitbooks.io/ucore-docs/lab1/lab1_appendix_a20.html)和
```asm
seta20.1:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al                                 # 如果 %al 第低2位为1，则ZF = 0, 则跳转
    jnz seta20.1                                    # 如果 %al 第低2位为0，则ZF = 1, 则不跳转

    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port

seta20.2:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
```
2. 在转化最后使用了一个长跳转跳到下一条代码,因为这两组代码不在一个code segment。
```asm
# Jump to next instruction, but in 32-bit code segment.
# Switches processor into 32-bit mode.
ljmp $PROT_MODE_CSEG, $protcseg     # ?
```

### bootloader加载 ELF文件

主要参考指导书，比较值得注意的是如何和磁盘进行交互。快速理解的方式就是对着port定义表看代码。
补充：
1. 关于从磁盘读取的代码：
```c
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
此处的`secno`中保存交互使用的`LBA`，共28bits，`LBA`的29bits表示主从盘。

insl定义在x86.h中
```c
static inline void
insl(uint32_t port, void *addr, int cnt) {
    asm volatile (
            "cld;"
            "repne; insl;"
            : "=D" (addr), "=c" (cnt)
            : "d" (port), "0" (addr), "1" (cnt)
            : "memory", "cc");
}
```
用到了扩展内联汇编，看这里[扩展内联汇编](https://objectkuan.gitbooks.io/ucore-docs/lab0/lab0_2_3_1_4_extend_gcc_asm.html),还有注意这个地方的cnt单位是`dword`。
# execise 5 and 6
比较简单，练习五只需要理解栈帧在栈中实际上形成了一个链表：
```
|           |
|           |
-------------      
|    ebp    |______      
-------------      |
|           |      |
|           |      |
|           |      |
-------------      |
|   ebp     |<------
-------------
|           |
|           |
-------------
|   ebp     |
-------------
|           |

caller的栈帧,caller's ebp = *(callee's ebp) ;
```
练习六略

# Challenge 1 & 2
switchto* 函数建议通过 中断处理的方式实现。主要要完成的代码是在 trap 里面处理 T_SWITCH_TO* 中断，并设置好返回的状态。

## T_SWITCH_TOU
从`KERNEL MODE`转到`USER MODE`，那我要从内核栈里pop出来用户保存的信息。
相反，就要把用户的信息再push回内核栈中。

查看全部需要修改的代码：
```c
// kern/trap/trap.c:171
// 这个地方需要处理中断信号，即为这项工作的叶函数。
case T_SWITCH_TOU:
case T_SWITCH_TOK:
    panic("T_SWITCH_** ??\n");
    break;
```

```c
// kern/init/init.c:38
// 注释掉即可

//LAB1: CAHLLENGE 1 If you try to do it, uncomment lab1_switch_test()
// user/kernel mode switch test
//lab1_switch_test();

// kern/init/init.c:85
// 使用SYSCALL实现这两个用户态的函数
static void
lab1_switch_to_user(void) {
    //LAB1 CHALLENGE 1 : TODO
}

static void
lab1_switch_to_kernel(void) {
    //LAB1 CHALLENGE 1 :  TODO
}
```

由于最后的grade的测评标准是基于`lab1_print_cur_status`函数的，那我们完全的信任该函数，这个函数检查了如下四项寄存器：`cs, ds, es, ss`和一个状态`@ring`，注意到以下写法：
```c
    cprintf("%d: @ring %d\n", round, reg1 & 3); //cs寄存器的low 2-bit表明了`CPL`
```

- 如何产生一个软中断？
    使用`int %xx`即可产生一个中断向量为xx的软中断。由硬件来进行查IDT表确定入口(vectors.S)，然后entry处push错误号和中断向量，跳转到`__alltraps`。这里面进行了一系列的push，构造一个trap frame，然后push进去esp来作为trap函数的参数。注意这个参数，它在后面大有用处。在trap函数返回后，恢复现场，通过iret返回用户态。
- 如何处理这个软中断？
    答案中采用了两种不同的方法进行状态的转换，但是思路是一样的，修改trap frame的段寄存器的值和状态字，使得它在返回的时候状态切换。

    运行到`trap_dispatch()`函数时的栈帧
    ```
    |   ret addr        |   <-  returen addr of trap
    ---------------------  
    |   tf              |
    ---------------------    
    |   ret addr        |   <-  return addr of trapentry
    ---------------------
    |   tf              |
    ---------------------
    |                   |
    |                   |
    |                   |
    |   trap frame      |
    |                   |
    |                   |
    |                   |
    ---------------------
    |    ret addr       |   <- return addr of user function
    ---------------------  
    |                   |
 
    ```
    ```c
    case T_SWITCH_TOU:
        if (tf->tf_cs != USER_CS) {     //判断状态
            switchk2u = *tf;            //新的trapframe copy原本的tf

            //进行段寄存器的修改
            switchk2u.tf_cs = USER_CS;
            switchk2u.tf_ds = switchk2u.tf_es = switchk2u.tf_ss = USER_DS;
            // ？ 这个地方有一个疑问
            switchk2u.tf_esp = (uint32_t)tf + sizeof(struct trapframe) - 8;
		
            // set eflags, make sure ucore can use io under user mode.
            // if CPL > IOPL, then cpu will generate a general protection.
            switchk2u.tf_eflags |= FL_IOPL_MASK;
		
            // set temporary stack
            // then iret will jump to the right stack
            // 修改了__alltraps中维护的esp，即trap()的参数。
            *((uint32_t *)tf - 1) = (uint32_t)&switchk2u;
        }
        break;
    case T_SWITCH_TOK:
        if (tf->tf_cs != KERNEL_CS) { 
            tf->tf_cs = KERNEL_CS;
            tf->tf_ds = tf->tf_es = KERNEL_DS;
            tf->tf_eflags &= ~FL_IOPL_MASK;
            // 在用户函数的栈帧上向上expand出来trap frame。
            switchu2k = (struct trapframe *)(tf->tf_esp - (sizeof(struct trapframe) - 8));
            //复制tf到扩展出来的栈
            memmove(switchu2k, tf, sizeof(struct trapframe) - 8);
            //同上
            *((uint32_t *)tf - 1) = (uint32_t)switchu2k;
        }
        break;
    ```

## challenge 2
比较简单，只需要在对应的处理分支中，将challenge1的代码粘贴进去即可。


<br /><br/><br />
<hr />
 LAB1 END
<hr />
<br/><br /><br/>







