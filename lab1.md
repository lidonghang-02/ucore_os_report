
# 练习1：理解通过make生成执行文件的过程
### 题1：操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)
```makefile
# create ucore.img
UCOREIMG        := $(call totarget,ucore.img)

$(UCOREIMG): $(kernel) $(bootblock)
        $(V)dd if=/dev/zero of=$@ count=10000
        $(V)dd if=$(bootblock) of=$@ conv=notrunc
        $(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)
```
1. `UCOREIMG        := $(call totarget,ucore.img)`
	- `UCOREIMG`是一个变量，用于保存ucore.img的路径
	- `$(call totarget,ucore.img)`是函数调用，`call`调用了一个名为`totarget`的内建函数
	- `totarget`是存储在`./tools/function.mk`中的函数，内容如下
	```
	totarget = $(addprefix $(BINDIR)$(SLASH),$(1))
	```
	  它的作用是将给定的参数添加到路径前缀中，生成一个目标路径。
	  其中`BINDIR :=bin`,`SLASH := /`,`$(1)`是参数
2. `$(UCOREIMG): $(kernel) $(bootblock)`
	- 声明了生成ucore.img的规则，依赖于`kernel`和`bootblock`文件。
	- 其中`kernel = $(call totarget,kernel)`，`bootblock = $(call totarget,bootblock)`
3.  下面这些命令是构建`ucore.img`的命令。其中`V := @`，使`dd`命令不会被打印出来（若`V`为空时，`dd`命令会被打印出来）。 
```makefile
   $(V)dd if=/dev/zero of=$@ count=10000
   $(V)dd if=$(bootblock) of=$@ conv=notrunc
   $(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
``` 
- `dd` 是一个 UNIX 系统下的命令，用于复制文件并进行相应的格式转换和数据处理。
- 第一条命令创建了一个全0的文件，大小为512 * 10000 字节
- 第二条命令把`bootblock`文件的内容写入`ucore.img`。`conv=notrunc`的意思是在写入的时候不断输出文件
- 第三条命令把`kernel`文件的内容写入`ucore.img`，并且从`ucore.img`的第二个512字节块处（`seek=1`）开始学入。
4. `$(call create_target,ucore.img)`
	- 调用了`create_target`函数，函数内容如下
		`create_target = $(eval $(call do_create_target,$(1),$(2),$(3),$(4),$(5)))`
		继续调用`do_create_target`函数
```makefile
# add packets and objs to target (target, #packes, #objs[, cc, flags])

define do_create_target

__temp_target__ = $(call totarget,$(1))

__temp_objs__ = $$(foreach p,$(call packetname,$(2)),$$($$(p))) $(3)

TARGETS += $$(__temp_target__)

ifneq ($(4),)

$$(__temp_target__): $$(__temp_objs__) | $$$$(dir $$$$@)

$(V)$(4) $(5) $$^ -o $$@

else

$$(__temp_target__): $$(__temp_objs__) | $$$$(dir $$$$@)

endif

endef
```
该宏接受五个参数：
-   `target`：目标的名称。
-   `packets`：包（packets）的数量。
-   `objs`：对象（objs）的数量。
-   `cc`（可选）：C编译器的命令。
-   `flags`（可选）：编译器的标志。
在宏的定义中，执行了以下操作：
1.  将`target`参数传递给`totarget`函数，并将结果存储在`__temp_target__`变量中。
2.  使用`packetname`函数将`packets`参数转换为包的名称，并将结果存储在`__temp_objs__`变量中。
3.  将`__temp_target__`添加到`TARGETS`变量中，用于跟踪所有的目标。
4.  根据是否提供了`cc`参数，创建一个目标规则。
    -   如果提供了`cc`参数，则使用给定的`cc`和`flags`编译器命令来构建目标。目标的依赖是`__temp_objs__`。
    -   如果未提供`cc`参数，则目标只有依赖关系，没有特定的构建命令。
5.  使用`|`符号和`$$$(dir $$$$@)`表示在构建目标之前，确保目标所在的目录存在。
### 题2：一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？
一个符合规范的硬盘主引导扇区（Master Boot Record，MBR）应该具备以下特征：
1. 大小为512字节：MBR是硬盘的第一个扇区，其大小固定为512字节。这个扇区包含了引导代码、分区表和引导签名等信息。
2. 引导代码：MBR扇区的前446字节通常用于存储引导代码，也称为主引导程序（Master Boot Code）。这段代码负责引导操作系统的加载过程。
3. 分区表：MBR扇区的接下来的64字节用于存储分区表。分区表可以包含最多四个主分区（Primary Partition）或者三个主分区和一个扩展分区（Extended Partition）。
4. 分区表项：每个分区表项占据16字节，用于描述一个分区的起始扇区、大小以及分区类型等信息。
5. 引导签名：MBR扇区的最后两个字节是一个特殊的引导签名，通常为0x55和0xAA。这个签名用于标识MBR扇区的有效性，操作系统引导加载程序会检查这个签名来确认MBR的合法性。
-------
# 练习2：使用qemu执行并调试lab1中的软件
#### 一、从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。
![[pic/Pasted image 20230811134032.png]]
#### 二、在初始化位置0x7c00设置实地址断点,测试断点正常。
![[pic/Pasted image 20230811134334.png]]
#### 三、从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。
![[pic/Pasted image 20230811134441.png]]
![[pic/Pasted image 20230811134536.png]]
#### 四、自己找一个bootloader或内核中的代码位置，设置断点并进行测试。
- 测试`kernel.asn`中的`kern_init()`函数
![[pic/Pasted image 20230811134822.png]]
-----
# 练习3： 分析bootloader进入保护模式的过程
#### 实模式和保护模式
- **实模式**：实模式是x86处理器的最初的工作模式，它是为了向后兼容早期的8086处理器而设计的。在是模式下，处理器以16位的寻址模式工作，可以访问段寄存器和偏移量的组合来寻址内存。在实模式下，没有内存保护机制，所有代码和数据都可以直接访问。实模式下的操作系统和应用程序必须自行管理内存分配和保护。
- **保护模式**：保护模式是x86处理器的一种高级工作模式，它引入了许多新的特性和功能。在保护模式下，处理器可以使用32或64位的寻址模式，可以访更大的内存空间。保护模式使用段选择子和偏移量的组合来寻址内存，并通过分段机制和分页机制提供了内存保护和虚拟内存管理。在保护模式下，操作系统可以将内存划分为多个独立的段，为不同的任务或进程提供隔离和保护。同时，保护模式还引入了特权级别（Ring）的概念，可以限制不同代码的权限和访问级别。
#### 保护模式寻址过程：
1. GDT：GDT（Global Descriptor Table，全局描述符表）是x86架构中的一张表，用于存储段描述符的信息。GDT是在保护模式下使用的，它定义了不同内存段的属性、访问权限和基址等信息。其中包含了一系列的段描述符，每个段描述符包含了以下信息：
	- 基址（Base）：指定了段的起始物理地址或线性地址。基址确定了段在内存中的位置，由gdtr寄存器存放
	- 段限长（Limit）：指定了段的大小或范围，定义了 段的结束地址
	- 属性标志（Attributes）：包含了一些标志位，用于指定段的属性和访问权限。例如：是否可读写、是否可执行、特权级别等
	- 段类型（Segment type）：指定了段的类型，如代码段、数据段、系统段等
2. 寻址过程
	- 寻址时，先找到gdtr寄存器。得到GDT的基址
	- 有了基址，又有段寄存器中保存的索引，得到段寄存器所指的GDT中的描述符
	- 通过描述符得到对应的内存段的起始地址
	- 将起始地址和偏移地址相加，得到线性地址（虚拟地址）
	- 线性地址经过变换，最终得到物理地址
![[pic/Pasted image 20230811143605.png]]
#### 阅读lab1/boot/bootasm.S源码
1. 为何开启A20，以及如何开启A20
	- A20是为了确保x86处理器在保护模式下可以寻址大于1MB的系统内存。A20总线能被软件关闭或打开，以此来阻止或允许地址总线收到A20传来的信号。在引导系统时，BIOS先打开A20总线来统计和测试所有的系统内存。而当BIOS准备将计算机的控制权交给操作系统时会先将A20总线关闭。
	- 键盘控制器方法开启A20
	```asm
	seta20.1:
		# 等待8042缓冲区为空
		inb $0x64, %al    # 从I/O端口0x64（状态寄存器）读取一个字节
		testb $0x2, %al   # 进行按位逻辑与(AND)操作，并设置相应的标志位。目的是检查键盘控制    器输入缓冲区是否为空
		jnz seta20.1      # 如果上一跳指令设置了非零标志位（输入缓冲区不为空）,则跳转
		# 0xd1->发送数据到8042的P2
		movb $0xd1, %al   # 0xd1写入al寄存器，用于开启A20
		outb %al, $0x64   # 将al寄存器的值写入I/O端口0x64
		
	seta20.2:
		# 等待缓冲区为空
		inb $0x64, %al    
		testb $0x2, %al
		jnz seta20.2
		# 向P2写入数据，将OR2（A20）置1
		movb $0xdf, %al 
		outb %al, $0x60   # 向0x60写入0xdf
    ```

2. 如何初始化GDT表
	```asm
	# Bootstrap GDT
	.p2align 2                                  # 强制4字节对齐
	gdt:
	    SEG_NULLASM                             # 空段
	    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)   # 引导加载程序和内核的代码段
	    SEG_ASM(STA_W, 0x0, 0xffffffff)         # 引导加载程序和内核的数据段
	gdtdesc:
	    .word 0x17                              # sizeof(gdt) - 1
	    .long gdt                               # gdt的地址
	```
	- 这里定义了一个GDT（全局描述符表），GDT由三个段描述符组成：空段描述符、代码段描述符和数据段描述符。其中，代码段和数据段描述符的段界限设置为0xffffffff，表示占据整个4GB线性地址空间。
	- `SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)` 定义了一个代码段描述符，使用宏定义 `SEG_ASM` 来生成描述符。这段代码定义了一个可执行、可读的代码段，起始地址为0x0，限长为0xffffffff（4GB）。
	-  `SEG_ASM(STA_W, 0x0, 0xffffffff)` 定义了一个数据段描述符，也使用 `SEG_ASM` 宏定义。这段代码定义了一个可写的数据段，起始地址为0x0，限长为0xffffffff（4GB）
其中SEG_NULLASM和SEG_ASM在asm.h文件中宏定义，定义如下：
```asm
/* 定义一个空段描述符 */
#define SEG_NULLASM \
.word 0, 0; \
.byte 0, 0, 0, 0


#define SEG_ASM(type,base,lim) \
.word (((lim) >> 12) & 0xffff), ((base) & 0xffff); \
.byte (((base) >> 16) & 0xff), (0x90 | (type)), \
(0xC0 | (((lim) >> 28) & 0xf)), (((base) >> 24) & 0xff)
```
- 上面代码中，`SEG_NULLASM`定义了一个空段描述符，该段描述符的所有字段都为零。具体来说，`.word 0,0`生成两个16位的零字，`.byte 0,0,0,0`生成四个8位的零字节。
- 在`SEG_ASM`中定义一个非空的段描述符，它接受三个参数：type，base和lim。`type`表示段的类型和访问权限，`base`表示段的起始地址，`lim`表示段的限长。
	-  `.word (((lim) >> 12) & 0xffff), ((base) & 0xffff)` 生成两个16位的字段，用于存储段的限长和起始地址的低16位。
	-   `.byte (((base) >> 16) & 0xff), (0x90 | (type)), (0xC0 | (((lim) >> 28) & 0xf)), (((base) >> 24) & 0xff)` 生成四个8位的字段，用于存储起始地址的高8位、类型和访问权限、段限长的高四位以及起始地址的高8位。

3. 如何使能和进入保护模式
常量定义
```
.set PROT_MODE_CSEG, 0x8 # 内核代码段选择子
.set PROT_MODE_DSEG, 0x10 # 内核数据段选择子
.set CR0_PE_ON, 0x1 # 保护模式使能标志
```
进入保护模式
```
lgdt gdtdesc             # 将GDT的地址加载到gdtr寄存器
movl %cr0, %eax          # 将控制寄存器cr0的内容加载到eax寄存器
orl $CR0_PE_ON, %eax     # 通过逻辑或将将CR0_PE_ON标志位置为1
movl %eax, %cr0          # 将eax中内容写入cr0寄存器，启动保护模式
ljmp $PROT_MODE_CSEG, $protcseg  # 跳转到保护模式的入口点，开始在保护模式下执行代码
```
这些指令用于切换处理器到32位保护模式。首先，使用`lgdt`指令加载GDT（全局描述符表）的地址和大小信息。然后，将CR0寄存器的PE标志设置为1，以启用保护模式。最后，使用`ljmp`指令跳转到32位代码段（C代码段）。

------
# 练习4： 分析bootloader加载ELF格式的OS的过程。
#### bootloader读取磁盘扇区
1. 等待磁盘准备好
2. 发出读取扇区的命令
3. 发出读取扇区的命令
4. 把磁盘扇区数据读到指定内存
- 代码实现
```c
/* readsect - 读取一个扇区到@dst */
static void readsect(void *dst, uint32_t secno)
{
	// 等待磁盘准备
	waitdisk();
	
	outb(0x1F2, 1); // count = 1
	outb(0x1F3, secno & 0xFF);
	outb(0x1F4, (secno >> 8) & 0xFF);
	outb(0x1F5, (secno >> 16) & 0xFF);
	outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
	outb(0x1F7, 0x20); // cmd 0x20 - 读取扇区
	
	// 等待磁盘准备
	waitdisk();
	
	// 读取一个扇区
	insl(0x1F0, dst, SECTSIZE / 4);
}
```
- `waitdisk()`函数实现
```c
static void waitdisk(void)
{
	while ((inb(0x1F7) & 0xC0) != 0x40)
/* 什么也不做 */;
}
```
#### ELF文件格式
ELF(Executable and Linkable Format)文件格式是Linux系统下的一种常用的目标文件格式，由ELF头部(ELF Header)、程序头部表(Program Header Table)、节头部表(Section Header Table)和各个节(Section)组成，有三种主要类型：
- 用于执行的可执行文件(executable file)，用于提供程序的进程映像，加载的内存执行。 
-   用于连接的可重定位文件(relocatable file)，可与其它目标文件一起创建可执行文件和共享目标文件。
-   共享目标文件(shared object file),连接器可将它与其它可重定位文件和共享目标文件连接成其它的目标文件，动态连接器又可将它与可执行文件和其它共享目标文件结合起来创建一个进程映像。
#### bootloader加载ELF格式的OS
1. 通过`readseg()`函数读取硬盘上的第一页（引导扇区）到内存中的`ELFHDR`地址处
2. 通过下面代码加载ELF
```c
/* bootmain - 引导加载程序的入口点 */
void bootmain(void)
{
// 从磁盘读取第一页
	readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
	
	// 通过检查ELF头部的魔数来检查ELF头部的有效性
	//ELF_MAGIC是ELF文件的魔数，用于标识有效的ELF文件。
	if (ELFHDR->e_magic != ELF_MAGIC)
	{
		//如果ELF头部的魔数和ELF_MAGIC不匹配，则跳转bad处。
		goto bad;
	}
	
	struct proghdr *ph, *eph;
	
	// 加载每个程序段（忽略 ph 标志）
	//遍历ELF头部中的每个程序段，通过readseg()函数将每个程序段从硬盘读取到指定的内存位置。
	
	//计算出程序头部表(Program Header Table)的地址，并将其指针赋值给ph。
	ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
	eph = ph + ELFHDR->e_phnum;
	//遍历程序头部表中的每个程序段描述符
	for (; ph < eph; ph++)
	{
		//将每个程序段从硬盘读取到指定的内存位置
		//p_va是程序段的虚拟地址
		//p_memsz是程序段在内存中的大小
		//p_offset是程序段在文件中的偏移量
		readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
	}
	
	// 从ELF header调用入口点(e_entry)，启动操作系统的执行
	// 注意：不返回
	//使用位掩码& 0xFFFFFF是为了去除可能存在的高位标志位
	((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
	
bad:
	outw(0x8A00, 0x8A00);
	outw(0x8A00, 0x8E00);
	
	/* do nothing */
	while (1)
	;
}
```
-----
# 练习5：实现函数调用堆栈跟踪函数
[题目要求](https://objectkuan.gitbooks.io/ucore-docs/content/lab1/lab1_2_1_5_ex5.html)
#### ebp和eip寄存器
1. `ebp` 寄存器（基指针寄存器）：
	- `ebp` 寄存器用于在函数中创建和管理栈帧。栈帧是在函数调用期间分配的一块内存区域，用于存储局部变量、函数参数和其他与函数执行相关的信息。
	- 在函数的入口处，`ebp` 寄存器通常被设置为当前栈帧的基址。它指向当前栈帧的底部，也就是上一个栈帧的 `ebp` 寄存器的值。
	- 通过 `ebp` 寄存器，可以在栈上访问函数的局部变量、参数以及上一个栈帧中的数据。
2. `eip` 寄存器（指令指针寄存器）：
	- `eip` 寄存器存储着下一条将要执行的指令的地址。它指示了程序当前所在的指令位置。
	- 在函数调用时，`eip` 寄存器存储着函数调用后将要返回的地址，也就是调用指令之后的下一条指令地址。
	- 当函数执行完毕时，通过恢复上一个栈帧的 `ebp` 和 `eip` 值，程序可以返回到调用函数的位置继续执行。
总结：
- `ebp` 寄存器用于管理栈帧，指向当前栈帧的底部。
- `eip` 寄存器存储下一条指令的地址，用于控制程序执行流程。
#### 编译器建立函数调用关的过程
[函数堆栈](https://learningos.github.io/ucore_os_webdocs/lab1/lab1_3_3_1_function_stack.html)
1. 函数调用前的准备：
	- 在函数调用之前，编译器会将函数参数按约定放入堆栈或寄存器中。参数的顺序和传递方式（寄存器或堆栈）取决于编译器和目标体系结构的约定。
	- 编译器将函数的返回地址（即调用指令的下一条指令地址）压入堆栈。
	- 在每个函数体之前插入类似如下的汇编指令
	```asm
	pushl   %ebp
	movl   %esp , %ebp
	```
2. 建立新的栈帧
	- 当函数被调用时，编译器会为该函数创建一个新的栈帧。栈帧是用于存储局部变量、函数参数和其他与函数执行相关的信息的内存区域。
	- 编译器通过将当前的基指针ebp压入堆栈，并将ebp设置为指向新栈帧的底部，来建立新的栈帧。
	- 在栈帧中，编译器可能会为局部变量和临时变量分配空间。
2. 函数调用
	- 编译器将控制转移到被调用函数的入口点，即将指令指针eip设置为被调用函数的起始地址。
	- 此时，被调用函数的栈帧中的局部变量和函数参数可以被访问和修改。
3. 函数返回
	- 当函数执行完毕时，编译器通过恢复上一个栈帧的ebp和eip值来返回调用函数的位置。
	- 编译器会将返回值放入约定的寄存器或堆栈位置，然后释放当前函数的堆栈。
#### 实现lab1/kern/debug/kdebug.c中的print_stackframe函数
1. 函数内容如下
```
/* LAB1 您的代码：第 1 步 */
/* (1)调用read_ebp()获取ebp的值。类型是 (uint32_t)；
* (2)调用read_eip()获取eip的值。类型是 (uint32_t)；
* (3) 从 0 .. STACKFRAME_DEPTH
* (3.1) ebp、eip的printf值
* (3.2) (uint32_t)调用参数 [0..4] = 地址 (uint32_t)ebp 中的内容 +2 [0..4]
* (3.3) cprintf("\n");
* (3.4)调用print_debuginfo(eip-1)打印C调用函数名和行号等。
* (3.5) 弹出一个调用栈帧
* 注意：调用函数的返回地址 eip = ss:[ebp+4]
* 调用函数的 ebp = ss:[ebp]
*/
```
2. 代码实现
```c
void print_stackframe(void)
{
	uint32_t eip, ebp;
	uint32_t arg_addressp;
	ebp = read_ebp();
	eip = read_eip();
	for (int i = 0; i < STACKFRAME_DEPTH; i++)
	{
		cprintf("ebp: 0x%08x, eip: 0x%08x ", ebp, eip);
		cprintf("args: ");
		for (int j = 0; j < 5; j++)
		{
			arg_addressp = *((uint32_t *)(ebp + (j + 2) * sizeof(uint32_t)));
			cprintf("0x%08x ", arg_addressp);
		}
		cprintf("\n");
		print_debuginfo(eip - 1);
		ebp = *((uint32_t *)ebp);
		eip = *((uint32_t *)(ebp + 4));
	}
}
```
3. 运行结果
![[pic/Pasted image 20230815131625.png]]
----
# 练习6：完善中断初始化和处理
[题目要求](https://objectkuan.gitbooks.io/ucore-docs/content/lab1/lab1_2_1_6_ex6.html)
#### 中断描述符表（保护模式下的中断向量表）
中断描述符表（Interrupt Descriptor Table）把每个中断或异常编号和一个指向中断服务例程的描述符联系起来。同 GDT 一样，IDT 是一个 8 字节的描述符数组，但 IDT 的第一项可以包含一个描述符。CPU 把中断（异常）号乘以 8 做为 IDT 的索引。IDT 可以位于内存的任意位置，CPU 通过 IDT 寄存器（IDTR）的内容来寻址 IDT 的起始地址。
```
+------------------------------------------------------------------------+
|       Offset 31:16   |  Selector    |        Reserve          |  Type  |
|        (31 bits)     |  (16 bits)   |       (8 bits)          |(8 bits)|
+------------------------------------------------------------------------+
```
在这个结构中
-   `Offset 31:16`：这个字段占据 16 位，表示中断处理代码的入口地址的高 16 位（即代码段偏移的高字节）。
-   `Selector`：这个字段占据 16 位，表示中断处理代码所在的代码段的选择子（即代码段的描述符索引）。
-   `Type`：这个字段占据 8 位，表示中断门的类型和特性。
通过将 `Offset 31:16` 和 `Selector` 组合在一起，可以确定中断处理代码的入口地址。具体的地址计算公式是：
```
entry_address = (Offset 31:16 << 16) | Selector
```
需要注意的是，IDT 表项的其余字段，如 `Reserved` 和 `Type`，用于指定中断门的一些特性，如门类型、特权级等，但不直接表示中断处理代码的入口地址。
#### kern/trap/trap.c中对中断向量表进行初始化的函数idt_init()的完善
```c
extern uintptr_t __vectors[];
for (int i = 0; i < sizeof(idt) / sizeof(struct gatedesc); i++)
{
	SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
}
SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);
lidt(&idt_pd);
```
1. 首先通过`extern uintptr_t __vectors[]`声明了一个外部变量，用于存储中断服务程序（ISR）的入口地址
2. 循环遍历IDT数组中的每个条目，循环范围是从0到IDT数组的大小除以一个IDT条目的大小
3. 调用宏`SETGATE`来设置IDT中的一个条目。参数解释如下：
	- 第一个参数 `idt[i]` 是要设置的 IDT 条目。
	- 第二个参数 `0` 表示该条目是一个陷阱门（而不是中断门）。
	- 第三个参数 `GD_KTEXT` 是中断处理程序所在的代码段的选择子。
	- 第四个参数 `__vectors[i]` 是中断处理程序的入口地址，即 ISR 的入口地址。
	- 第五个参数 `DPL_KERNEL` 表示门的特权级别设置为内核级别。
4. 宏`SETGATE`代码如下
```c
#define SETGATE(gate, istrap, sel, off, dpl) { \
	(gate).gd_off_15_0 = (uint32_t)(off) & 0xffff; \
	(gate).gd_ss = (sel); \
	(gate).gd_args = 0; \
	(gate).gd_rsv1 = 0; \
	(gate).gd_type = (istrap) ? STS_TG32 : STS_IG32; \
	(gate).gd_s = 0; \
	(gate).gd_dpl = (dpl); \
	(gate).gd_p = 1; \
	(gate).gd_off_31_16 = (uint32_t)(off) >> 16; \
}
```
- 以上代码是一个宏定义，它定义了一个名为 `SETGATE` 的宏，用于设置中断描述表（IDT）中的一个条目（门）的值。该宏接受以下参数：
	-  `gate`：表示要设置的 IDT 条目的变量名。
	-   `istrap`：一个布尔值，表示该条目是中断门还是陷阱门。如果为真（非零），则表示中断门；如果为假（零），则表示陷阱门。
	-   `sel`：表示中断处理程序所在的代码段的选择子（代码段的描述符索引）。
	-   `off`：表示中断处理程序的入口地址。
	-   `dpl`：表示门的特权级别（Descriptor Privilege Level）。
5. `SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);`调用宏 `SETGATE` 来设置一个特殊的 IDT 条目，用于从用户态切换到内核态。这个条目使用了与上面相同的宏参数，只是特权级别被设置为 `DPL_USER`，表示门的特权级别设置为用户级别。
6. `lidt(&idt_pd);`使用 `lidt` 指令加载 IDT。参数 `&idt_pd` 是 IDT 位置描述符的地址，通过传递给 `lidt` 指令，告诉 CPU IDT 在内存中的位置。
#### 完善trap.c中的中断处理函数trap()
- 实现一
```c
case IRQ_OFFSET + IRQ_TIMER:
	ticks++;
	if (ticks % TICK_NUM == 0)
	{
		print_ticks();
	}
	break;
case T_SWITCH_TOU:
	//检查是否处于内核模式
	if ((tf->tf_cs & 3) == 0)
	{
		// 发生了从内核模式切换到用户模式的陷阱
		//设置为用户代码段选择子USER_CS
		tf->tf_cs = USER_CS;
		//设置为用户数据段选择子USER_DS
		tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
		//见效栈指针tf_esp的值，为用户模式的堆栈留出空间
		tf->tf_esp = tf->tf_esp - sizeof(struct pushregs);
		//设置标志寄存器的IF(Interrupt Flag)位，允许中断
		tf->tf_eflags |= FL_IF; 
	}
	else
	{
		panic("T_SWITCH_TOU in user mode.\n");
	}
	break;
case T_SWITCH_TOK:
	//检查是否处于用户模式
	if ((tf->tf_cs & 3) == 3)
	{
		// 发生了从用户模式切换到内核模式的陷阱
		//设置内核代码段选择子
		tf->tf_cs = KERNEL_CS;
		//设置内核数据段选择子
		tf->tf_ds = tf->tf_es = KERNEL_DS;
		//将栈段寄存器设置为内核数据段选择子
		tf->tf_ss = KERNEL_DS;
		//清楚标志寄存器的IF位，禁止中断
		tf->tf_eflags &= ~FL_IF; 
	}
	else
	{
		panic("T_SWITCH_TOK in kernel mode.\n");
	}
break;
```
- 实现二
声明临时变量
```c
struct trapframe switchk2u, *switchu2k;
```
代码实现
```c
case T_SWITCH_TOU:
	//检查当前代码段选择子是否为用户代码段选择子
	if (tf->tf_cs != USER_CS)
	{
		//使用临时变量switchk2u存储当前的陷阱帧内容
		switchk2u = *tf;
		switchk2u.tf_cs = USER_CS;
		switchk2u.tf_ds = switchk2u.tf_es = switchk2u.tf_ss = USER_DS;
		//为用户模式的堆栈留出空间
		switchk2u.tf_esp = (uint32_t)tf + sizeof(struct trapframe) - 8;
  
		// 设置eflags，确保ucore可以在用户模式下使用io。
		// 如果 CPL > IOPL，那么 cpu 将产生一般保护。
		//设置IF位，允许中断
		switchk2u.tf_eflags |= FL_IOPL_MASK;
  
		// 设置临时栈
		// 然后iret会跳转到右边的栈
		//将当前陷阱帧指针前移一个位置
		//并指向switch2u，这样iret指令会从正确的栈中返回
		*((uint32_t *)tf - 1) = (uint32_t)&switchk2u;
	}
	break;
case T_SWITCH_TOK:
	//检查是否是内核代码段选择子
	if (tf->tf_cs != KERNEL_CS)
	{
		tf->tf_cs = KERNEL_CS;
		tf->tf_ds = tf->tf_es = KERNEL_DS;
		//清楚eflags的IOPL位，以禁止在用户模式下使用I/O
		tf->tf_eflags &= ~FL_IOPL_MASK;
		//计算内核模式下的陷阱帧指针
		switchu2k = (struct trapframe *)(tf->tf_esp - (sizeof(struct trapframe) - 8));
		//将当前陷阱帧内容复制到switchu2k指向的内核模式陷阱帧中
		memmove(switchu2k, tf, sizeof(struct trapframe) - 8);
		//iret从正确的栈中返回
		*((uint32_t *)tf - 1) = (uint32_t)switchu2k;
	}
	break;
```