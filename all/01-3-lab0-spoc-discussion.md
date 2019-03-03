# lec2：lab0 SPOC思考题

## **提前准备**
（请在上课前完成，option）

- 完成lec2的视频学习
- git pull ucore_os_lab, os_tutorial_lab, os_course_exercises  in github repos。这样可以在本机上完成课堂练习。
- 了解代码段，数据段，执行文件，执行文件格式，堆，栈，控制流，函数调用,函数参数传递，用户态（用户模式），内核态（内核模式）等基本概念。思考一下这些基本概念在不同操作系统（如linux, ucore,etc.)与不同硬件（如 x86, riscv, v9-cpu,etc.)中是如何相互配合来体现的。
- 安装好ucore实验环境，能够编译运行ucore labs中的源码。
- 会使用linux中的shell命令:objdump，nm，file, strace，gdb等，了解这些命令的用途。
- 会编译，运行，使用v9-cpu的dis,xc, xem命令（包括启动参数），阅读v9-cpu中的v9\-computer.md文档，了解汇编指令的类型和含义等，了解v9-cpu的细节。
- 了解基于v9-cpu的执行文件的格式和内容，以及它是如何加载到v9-cpu的内存中的。
- 在piazza上就学习中不理解问题进行提问。

---

## 思考题

- 你理解的对于类似ucore这样需要进程/虚存/文件系统的操作系统，在硬件设计上至少需要有哪些直接的支持？至少应该提供哪些功能的特权指令？<br>
  答：进程需要支持中断处理服务，虚存需要支持从虚实地址的映射，文件系统需要支持稳定、持久的数据存储且断电不会丢失数据。<br>
  需要的特权指令：软中断指令，寻址、访存指令，I/O指令。

- 你理解的x86的实模式和保护模式有什么区别？物理地址、线性地址、逻辑地址的含义分别是什么？<br>
  答：实模式是80386为兼容8086的一种模式，寻址空间不超过1M，没有保护机制。而保护模式的寻址空间变为4G（32位），而且提供分页、分段机制，让不同特权级访问不同空间，可以保护操作系统的安全。<br>
  物理地址：硬件上用于访问存储以及外设的地址<br>
  线性地址：逻辑地址到物理地址变换的中间层，由逻辑地址经分段机制得到<br>
  逻辑地址：在有地址变换功能的计算机中，访存指令给出的地址。逻辑地址空间是应用程序直接使用的地址空间。

- 你理解的risc-v的特权模式有什么区别？不同 模式在地址访问方面有何特征？<br>
  答：查阅资料得，RISC-V有四种特权级：机器模式、用户模式、管理员模式、Hypervisor模式。机器模式的特权最高，用户模式用于应用程序，管理员模式用于操作系统，Hypervisor则是为了支持虚拟机监视器。<br>
  机器模式的访问不受限制，用户模式下可以访问应用程序，但不能访问系统部分，管理员模式在操作系统和SEE（管理员执行环境）、HAL（硬件抽象层）之间提供隔离，Hypervisor将在一个虚拟机监视器和HEE（Hypervisor执行环境）、运行在机器模式下的HAL之间提供隔离。

- 理解ucore中list_entry双向链表数据结构及其4个基本操作函数和ucore中一些基于它的代码实现（此题不用填写内容）

- 对于如下的代码段，请说明":"后面的数字是什么含义
```
 /* Gate descriptors for interrupts and traps */
 struct gatedesc {
    unsigned gd_off_15_0 : 16;        // low 16 bits of offset in segment
    unsigned gd_ss : 16;            // segment selector
    unsigned gd_args : 5;            // # args, 0 for interrupt/trap gates
    unsigned gd_rsv1 : 3;            // reserved(should be zero I guess)
    unsigned gd_type : 4;            // type(STS_{TG,IG32,TG32})
    unsigned gd_s : 1;                // must be 0 (system)
    unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
    unsigned gd_p : 1;                // Present
    unsigned gd_off_31_16 : 16;        // high bits of offset in segment
 };
```
答：表示该成员变量所占的位数。相加可知，总的位数为64位。

- 对于如下的代码段，

```
#define SETGATE(gate, istrap, sel, off, dpl) {            \
    (gate).gd_off_15_0 = (uint32_t)(off) & 0xffff;        \
    (gate).gd_ss = (sel);                                \
    (gate).gd_args = 0;                                    \
    (gate).gd_rsv1 = 0;                                    \
    (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;    \
    (gate).gd_s = 0;                                    \
    (gate).gd_dpl = (dpl);                                \
    (gate).gd_p = 1;                                    \
    (gate).gd_off_31_16 = (uint32_t)(off) >> 16;        \
}
```
如果在其他代码段中有如下语句，
```
unsigned intr;
intr=8;
SETGATE(intr, 1,2,3,0);
```
请问执行上述指令后， intr的值是多少？
<br>
0x10002

### 课堂实践练习

#### 练习一

1. 请在ucore中找一段你认为难度适当的AT&T格式X86汇编代码，尝试解释其含义。
seta20.1:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64 # 0xd1 means: write data to 8042's P2 port
解释：0x64是8042的一个端口，首先从0x64读取状态到al寄存器，判断是否为0x2，如果是，说明缓冲区非空，跳转到seta20.1处执行。若缓冲区为空，那么继续执行，将0xd1发送到0x64端口
2. (option)请在rcore中找一段你认为难度适当的RV汇编代码，尝试解释其含义。

#### 练习二

宏定义和引用在内核代码中很常用。请枚举ucore或rcore中宏定义的用途，并举例描述其含义。<br>
答：宏定义的用途：数据类型的定义，用于复杂数据类型的访问，常量的定义。<br>
比如#define STS_T16A 0x1是表示STS_T16A指代0x1<br>
#define SETGATE(gate, istrap, sel, off, dpl)用于对复杂的结构体的访问操作<br>

#### reference
 - [Intel格式和AT&T格式汇编区别](http://www.cnblogs.com/hdk1993/p/4820353.html)
 - [x86汇编指令集  ](http://hiyyp1234.blog.163.com/blog/static/67786373200981811422948/)
 - [PC Assembly Language, Paul A. Carter, November 2003.](https://pdos.csail.mit.edu/6.828/2016/readings/pcasm-book.pdf)
 - [*Intel 80386 Programmer's Reference Manual*, 1987](https://pdos.csail.mit.edu/6.828/2016/readings/i386/toc.htm)
 - [IA-32 Intel Architecture Software Developer's Manuals](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
