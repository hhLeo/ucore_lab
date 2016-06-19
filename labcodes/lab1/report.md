# Lab1 Report
## 黄欢 2013011331 计33

## [EX1]

### [EX1.1] 操作系统镜像文件 ucore.img 是如何一步一步生成的?(需要比较详细地解释 Makefile 中 每一条相关命令和命令参数的含义,以及说明命令导致的结果)

为了生成bin/ucore.img，由如下代码知，需要先生成bootblock和kernel
```
$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
```
为了生成bin/bootblock，由如下代码知，需要先生成bootasm.o、bootmain.o、sign
```
$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
```
为了生成obj/boot/bootasm.o, obj/boot/bootmain.o，由如下代码知，实际代码由宏批量生成
```
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
```
生成bootasm.o需要bootasm.S，实际命令为
```
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
```
其中关键的参数为
```
	-ggdb  生成可供gdb使用的调试信息。
	-m32  生成适用于32位环境的代码。
	-gstabs  生成stabs格式的调试信息。这样ucore的monitor可以显示出便于开发者阅读的函数调用栈信息。
	-nostdinc  不使用标准库。
	-fno-stack-protector  不生成用于检测缓冲区溢出的代码。
	-Os  为减小代码大小而进行优化。
	I<dir>  添加搜索头文件的路径
```
生成bootmain.o需要bootmain.c，实际命令为
```
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o
```
新增的关键参数有
```
	fno-builtin  除非用__builtin_前缀，否则不进行builtin函数的优化
```
为了生成bin/sign，由如下代码知，
```
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)
```
实际命令为
```
gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
```
首先生成bootblock.o
```
ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 bj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
```
其中关键的参数为
```
	-m <emulation>  模拟为i386上的连接器
	-nostdlib  不使用标准库
	-N  设置代码段和数据段均可读写
	-e <entry>  指定入口
	-Ttext  制定代码段开始位置
```
拷贝二进制代码bootblock.o到bootblock.out
```
objcopy -S -O binary obj/bootblock.o obj/bootblock.out
```
其中关键的参数为
```
	-S  移除所有符号和重定位信息
	-O <bfdname>  指定输出格式
```
使用sign工具处理bootblock.out，生成bootblock
```
bin/sign obj/bootblock.out bin/bootblock
```
为了生成bin/kernel，由如下代码知，需要先生成kernel.ld init.o readline.o stdio.o kdebug.o kmonitor.o panic.o clock.o console.o intr.o picirq.o trap.o trapentry.o vectors.o pmm.o  printfmt.o string.o，kernel.ld已存在
```
$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
```
为了生成obj/kern/*/*.o 这些.o文件，相关makefile代码为
```
$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))
```
以obj/kern/init/init.o为例，编译需要init.c，实际命令为
```
gcc -Ikern/init/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/init/init.c -o obj/kern/init/init.o
```
生成kernel时，makefile的几条指令中有@前缀的都不必需
必需的命令只有
```
ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel obj/kern/init/init.o obj/kern/libs/readline.o obj/kern/libs/stdio.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/debug/panic.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/intr.o obj/kern/driver/picirq.o obj/kern/trap/trap.o obj/kern/trap/trapentry.o obj/kern/trap/vectors.o obj/kern/mm/pmm.o obj/libs/printfmt.o obj/libs/string.o
```
其中新出现的关键参数为
```
	-T <scriptfile>  让连接器使用指定的脚本
```
生成一个有10000个块的文件，每个块默认512字节，用0填充
```
dd if=/dev/zero of=bin/ucore.img count=10000
```
把bootblock中的内容写到第一个块
```
dd if=bin/bootblock of=bin/ucore.img conv=notrunc
```
从第二个块开始写kernel中的内容
```
dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
```
### [EX1.2] 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?

```
>sign.c
	char buf[512];
	buf[510] = 0x55;
    buf[511] = 0xAA;
```
一个磁盘主引导扇区只有512字节。且第510个（倒数第二个）字节是0x55，第511个（倒数第一个）字节是0xAA。

## [EX2] 使用qemu执行并调试lab1中的软件。（要求在报告中简要写出练习过程）

### [EX2.1] 从 CPU 加电后执行的第一条指令开始,单步跟踪 BIOS 的执行。

单步跟踪的方法如下：

1 修改 lab1/tools/gdbinit,内容为:
```
file bin/kernel
set architecture i8086
target remote :1234
break kern_init
continue
```
2 在 lab1目录下，执行
```
make debug
```
3 在看到gdb的调试界面(gdb)后，在gdb调试界面下执行如下命令
```
si
```
即可单步跟踪BIOS了。

4 在gdb界面下，可通过如下命令来看BIOS的代码
```
 x /2i $pc  //显示当前eip处的汇编指令
```
附：gdb的单步命令

在gdb中，有next, nexti, step, stepi等指令来单步调试程序，他们功能各不相同，区别在于单步的“跨度”上。
```
next 单步到程序源代码的下一行，不进入函数。
nexti 单步一条机器指令，不进入函数。
step 单步到下一个不同的源代码行（包括进入函数）。
stepi 单步一条机器指令。
```

### [EX2.2] 在初始化位置0x7c00 设置实地址断点,测试断点正常。

1 修改 lab1/tools/gdbinit,内容为:
```
file bin/kernel
set architecture i8086
target remote :1234
b *0x7c00
c
x /2i $pc
set architecture i386
```

2 运行"make debug"便可得到
```
    Breakpoint 2, 0x00007c00 in ?? ()
    => 0x7c00:      cli    
       0x7c01:      cld    
```
### [EX2.3] 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。

在调用qemu 时增加-d in_asm -D q.log 参数，便可以将运行的汇编指令保存在q.log 中。
在q.log中读到"call bootmain"前执行的命令，与bootasm.S和bootblock.asm中的代码相同。

## [EX3]分析bootloader进入保护模式的过程。（要求在报告中写出分析）

BIOS将通过读取硬盘主引导扇区到内存，并转跳到对应内存中的位置执行bootloader。请分析bootloader是如何完成从实模式进入保护模式的。

提示：需要阅读小节“保护模式和分段机制”和lab1/boot/bootasm.S源码，了解如何从实模式切换到保护模式，需要了解：

    为何开启A20，以及如何开启A20
    如何初始化GDT表
    如何使能和进入保护模式

答：初始状态：%cs=0 $pc=0x7c00

1.清理环境：包括将flag置0和将段寄存器置0
```
.code16                                             # Assemble for 16-bit mode
    cli                                             # Disable interrupts
    cld                                             # String operations increment

    # Set up the important data segment registers (DS, ES, SS).
    xorw %ax, %ax                                   # Segment number zero
    movw %ax, %ds                                   # -> Data Segment
    movw %ax, %es                                   # -> Extra Segment
    movw %ax, %ss                                   # -> Stack Segment
```
开启A20：通过将键盘控制器上的A20线置于高电位，全部32条地址线可用，可以访问4G的内存空间。
```
    # Enable A20:
    #  For backwards compatibility with the earliest PCs, physical
    #  address line 20 is tied low, so that addresses higher than
    #  1MB wrap around to zero by default. This code undoes this.
seta20.1:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port

seta20.2:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
```
初始化GDT表：GDT表已经储存在引导区中，载入即可
```
        lgdt gdtdesc
```
进入保护模式：通过将cr0寄存器PE位置1便开启了保护模式
```
        movl %cr0, %eax
        orl $CR0_PE_ON, %eax
        movl %eax, %cr0
```
通过长跳转更新cs的基地址
```
    # Jump to next instruction, but in 32-bit code segment.
    # Switches processor into 32-bit mode.
    ljmp $PROT_MODE_CSEG, $protcseg
```
设置段寄存器，并建立堆栈
```
.code32                                             # Assemble for 32-bit mode
protcseg:
    # Set up the protected-mode data segment registers
    movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
    movw %ax, %ds                                   # -> DS: Data Segment
    movw %ax, %es                                   # -> ES: Extra Segment
    movw %ax, %fs                                   # -> FS
    movw %ax, %gs                                   # -> GS
    movw %ax, %ss                                   # -> SS: Stack Segment
```
转到保护模式完成，进入boot主方法
```
        call bootmain
```
## [EX4] 分析bootloader加载ELF格式的OS的过程。（要求在报告中写出分析）

通过阅读bootmain.c，了解bootloader如何加载ELF文件。通过分析源代码和通过qemu来运行并调试bootloader&OS，

    bootloader如何读取硬盘扇区的？
    bootloader是如何加载ELF格式的OS？

提示：可阅读“硬盘访问概述”，“ELF执行文件格式概述”这两小节。

答：
readsect函数：从设备的第secno扇区读取1扇区数据到dst位置
```
/* readsect - read a single sector at @secno into @dst */
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
readseg调用readsect，可以从设备读取任意长度的内容。
```
/* *
 * readseg - read @count bytes at @offset from kernel into virtual address @va,
 * might copy more than asked.
 * */
static void
readseg(uintptr_t va, uint32_t count, uint32_t offset) {
    uintptr_t end_va = va + count;

    // round down to sector boundary
    va -= offset % SECTSIZE;

    // translate from bytes to sectors; kernel starts at sector 1
    uint32_t secno = (offset / SECTSIZE) + 1;

    // If this is too slow, we could read lots of sectors at a time.
    // We'd write more to memory than asked, but it doesn't matter --
    // we load in increasing order.
    for (; va < end_va; va += SECTSIZE, secno ++) {
        readsect((void *)va, secno);
    }
}
```
在bootmain函数中，首先读取ELF的头部，通过储存在头部的幻数判断是否是合法的ELF文件，ELF头部有描述ELF文件应加载到内存什么位置的描述表，先将描述表的头地址存在ph，按照描述表将ELF文件中数据载入内存，ELF文件0x1000位置后面的0xd1ec比特被载入内存0x00100000，ELF文件0xf000位置后面的0x1d20比特被载入内存0x0010e000，根据ELF头部储存的入口信息，找到内核的入口。
```
/* bootmain - the entry of bootloader */
void
bootmain(void) {
    // read the 1st page off disk
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    // is this a valid ELF?
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

    struct proghdr *ph, *eph;

    // load each program segment (ignores ph flags)
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();

bad:
    outw(0x8A00, 0x8A00);
    outw(0x8A00, 0x8E00);

    /* do nothing */
    while (1);
}
```

## [EX5]实现函数调用堆栈跟踪函数 （需要编程）

答：
ss:ebp指向的堆栈位置储存着caller的ebp，以此为线索可以得到所有使用堆栈的函数ebp。 
ss:ebp+4指向caller调用时的eip。
ss:ebp+8等是（可能的）参数。

实现步骤：
```
     /* LAB1 YOUR CODE : STEP 1 */
     /* (1) call read_ebp() to get the value of ebp. the type is (uint32_t);
      * (2) call read_eip() to get the value of eip. the type is (uint32_t);
      * (3) from 0 .. STACKFRAME_DEPTH
      *    (3.1) printf value of ebp, eip
      *    (3.2) (uint32_t)calling arguments [0..4] = the contents in address (unit32_t)ebp +2 [0..4]
      *    (3.3) cprintf("\n");
      *    (3.4) call print_debuginfo(eip-1) to print the C calling function name and line number, etc.
      *    (3.5) popup a calling stackframe
      *           NOTICE: the calling funciton's return addr eip  = ss:[ebp+4]
      *                   the calling funciton's ebp = ss:[ebp]
      */
```
输出：
```
moocos-> make qemu
(THU.CST) os is loading ...

Special kernel symbols:
  entry  0x00100000 (phys)
  etext  0x001032c3 (phys)
  edata  0x0010ea16 (phys)
  end    0x0010fd20 (phys)
Kernel executable memory footprint: 64KB
ebp:0x00007b08 eip:0x001009a6 args:0x00010094 0x00000000 0x00007b38 0x00100092 
    kern/debug/kdebug.c:305: print_stackframe+21
ebp:0x00007b18 eip:0x00100c95 args:0x00000000 0x00000000 0x00000000 0x00007b88 
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp:0x00007b38 eip:0x00100092 args:0x00000000 0x00007b60 0xffff0000 0x00007b64 
    kern/init/init.c:48: grade_backtrace2+33
ebp:0x00007b58 eip:0x001000bb args:0x00000000 0xffff0000 0x00007b84 0x00000029 
    kern/init/init.c:53: grade_backtrace1+38
ebp:0x00007b78 eip:0x001000d9 args:0x00000000 0x00100000 0xffff0000 0x0000001d 
    kern/init/init.c:58: grade_backtrace0+23
ebp:0x00007b98 eip:0x001000fe args:0x001032fc 0x001032e0 0x0000130a 0x00000000 
    kern/init/init.c:63: grade_backtrace+34
ebp:0x00007bc8 eip:0x00100055 args:0x00000000 0x00000000 0x00000000 0x00010094 
    kern/init/init.c:28: kern_init+84
ebp:0x00007bf8 eip:0x00007d68 args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8 
    <unknow>: -- 0x00007d67 --
```
其对应的是第一个使用堆栈的函数，bootmain.c中的bootmain。 bootloader设置的堆栈从0x7c00开始，使用"call bootmain"转入bootmain函数。 call指令压栈，所以bootmain中ebp为0x7bf8。

## [EX6] 完善中断初始化和处理（需要编程）

### [EX6.1] 中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

中断向量表一个表项占用8字节，其中2-3字节是段选择子，0-1字节和6-7字节拼成位移，两者联合便是中断处理程序的入口地址。

### [EX6.2] 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。在idt_init函数中，依次对所有中断入口进行初始化。使用mmu.h中的SETGATE宏，填充idt数组内容。每个中断的入口由tools/vectors.c生成，使用trap.c中声明的vectors数组即可。

答：实验分析：
```
1.所有中断服务入口地址存在__vectors.
2.设置中断描述表的中断服务入口（the entries of ISR in Interrupt Description Table (IDT)）.
3.load the IDT
```
### [EX6.3] 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字”100 ticks”。

答：实验分析：
```
1.每次循环给tick加1.
2.每100次循环调用print_ticks().
```
## [EX7]

### [EX7.1]扩展proj4,增加syscall功能，即增加一用户态函数（可执行一特定系统调用：获得时钟计数值），当内核初始完毕后，可从内核态返回到用户态的函数，而用户态的函数又通过系统调用得到内核态的服务（通过网络查询所需信息，可找老师咨询。如果完成，且有兴趣做代替考试的实验，可找老师商量）。需写出详细的设计和分析报告。完成出色的可获得适当加分。

提示： 规范一下 challenge 的流程。

答：
在idt_init中，将用户态调用SWITCH_TOK中断的权限打开。 SETGATE(idt[T_SWITCH_TOK], 1, KERNEL_CS, __vectors[T_SWITCH_TOK], 3);

在trap_dispatch中，将iret时会从堆栈弹出的段寄存器进行修改 对TO User
```
        tf->tf_cs = USER_CS;
        tf->tf_ds = USER_DS;
        tf->tf_es = USER_DS;
        tf->tf_ss = USER_DS;
```
对TO Kernel
```
        tf->tf_cs = KERNEL_CS;
        tf->tf_ds = KERNEL_DS;
        tf->tf_es = KERNEL_DS;
```
在lab1_switch_to_user中，调用T_SWITCH_TOU中断。 注意从中断返回时，会多pop两位，并用这两位的值更新ss,sp，损坏堆栈。 所以要先把栈压两位，并在从中断返回后修复esp。
```
    asm volatile (
        "sub $0x8, %%esp \n"
        "int %0 \n"
        "movl %%ebp, %%esp"
        : 
        : "i"(T_SWITCH_TOU)
    );
```
在lab1_switch_to_kernel中，调用T_SWITCH_TOK中断。 注意从中断返回时，esp仍在TSS指示的堆栈中。所以要在从中断返回后修复esp。
```
    asm volatile (
        "int %0 \n"
        "movl %%ebp, %%esp \n"
        : 
        : "i"(T_SWITCH_TOK)
    );
```
但这样不能正常输出文本。根据提示，在trap_dispatch中转User态时，将调用io所需权限降低。
```
    tf->tf_eflags |= 0x3000;
```

## [Others]

### 完成实验后，请分析ucore_lab中提供的参考答案，并请在实验报告中说明你的实现与参考答案的区别
能做的部分，自己分析；未知的部分，参考提供的答案进行理解。
所有部分仔细地实践一遍。

### 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点，并简要说明你对二者的含义，关系，差异等方面的理解（也可能出现实验中的知识点没有对应的原理知识点）
```
lab1

实模式BIOS
base+offset：线性地址（cs+eip）

bootloader：
1.实模式（16位，1G）->保护模式（32,4）
2.加载段机制（保护模式）
3.从硬盘读取kernel到内存（多扇区）
4.跳转到ucore，控制权交给ucore

段机制：
段：4G，从0开始

页机制在段机制的基础上完成

段选择址Index
段描述符
全局描述符表（GDT）：空间、地址、大小（映射关系）
GDTR（地址信息）
CS、EIP、基址（base）、长度（limit）

优先级（特权级）0>1>2>3
系统寄存器()

加载ELF格式的ucore OS kernel

#### C 函数调用的实现
esp当前栈
ebp指针链表

参数 函数返回值

#### gcc 内联汇编
内联汇编：GCC对C的扩张，可直接在C语言中插入汇编

asm(assembler template
    :output operands (optional)
    :input operands (optional)
    :clobbers (optional)
    );

volatile：不需要做进一步的优化、调整顺序
%0：第一个用到的寄存器
r：代表任意寄存器


int80：软中断：系统调用

#### X86中断处理过程
中断向量表（中断描述符表IDT -- OS建立）
中断、异常

X86中断处理：（硬件层面）（例程和两个表是OS实现的）
确定中断服务例程（ISE）
中断号
->IDT里的每一项：中断门/陷阱门（里面存的段选择址）
->全局描述符表GDT里存的段描述符（里面有一个基地址Base address）
->（+IDT里的offset=线性地址）指向ISR（中断服务例程）

IDT：中断号
IDTR：中断描述符

保存&恢复
硬件&软件

段描述符：
CS低两位：0：内核态；1：用户态
是否有特权级区别

1.同一特权级（内核态，硬件，一旦产生中断的时候硬件会压栈压进去，压在同一个栈里面）
Error code：特意的严重的异常，不是每一个中断或者异常都会产生Error code
压入EIP和CS：当前被打断的那个地址，或者是当前被打断的下一条地址
EFLAGS：当前被打断的时候的标志性的内容

2.不同特权级（用户态，切换，不同的栈）
（+）ESP和SS：产生中断时，在用户态里面的那个栈的地址

iret中断服务例程返回
ret
retf

系统调用/trap陷入/软中断：
用户态：int80

#### 练习1
lab1_result:
    ls
    make clean
    make V=
//详细编译执行过程：
gcc:编译得.o目标文件
ld:目标文件->执行程序（eg.bootloader.out）
dd:把Bootloader放到一个虚拟的硬盘里面去
(eg.生成一个虚拟硬盘叫uCore.img count，硬件模拟器会基于这个虚拟硬盘中的数据来执行相应的这个代码）

生成了两个软件
第一个是Bootloader 第二叫kernel（uCore的组成部分）


    less Makefile//看makefile

    less tools/sign.c //完成了特征的标记

#### 练习2
qemu GDB执行和调试

    less Makefile
    /lab1-mon
它这里面大致是干了两个事情
1.让qemu把它执行的指令给记录下来
把log信息给记录下来
放到这个地方 q.log
2.和GDB结合来调试正在执行的Bootloader
(还没有到uCore)
    less tools/lab1init
（还在Bootloader阶段，初始化的一些执行指令，GDB能够识别的一些命令）

    make lab1-mon
(gdb)x /10i $pc 
//把当前(bootloader)的10条指令都显示出来
(gdb)continue
(gdb)quit

在ECLIPSE环境下可提升它调试
GDB可调试比较简单的

#### 练习3
我们刚才说到的三个功能
打开A20 建立全局描述符表
然后使能 进入保护模式 32位的保护模式
都是在这个汇编里面完成的

在moocos/ucoreprojs/proj1里面
    make
sign.c：一个工具，不是Bootloader组成部分，而是生成主引导扇区的一个辅助工具，通过它生成了一个合格的Bootloader主引导扇区
bootblock：
    make qemu
显示一个字符串 Hello world!!
并口 串口和CGA

#### 练习4
labcodes_answer/lab1_result/bin
    file kernel
中断初始化、产生时钟中断、把这个信息显示在屏幕上等等
（uCore完成的基本功能）

Bootloader怎么能够把uCore给加载到内存当中去？
第一步 是怎么能够读取硬盘中的信息
这需要Bootloader能够访问硬盘
第二个是Bootloader把这个硬盘数据读出来之后
要把其中ELF格式这个文件给分析出来
从而知道我们uCore它的代码段
应该放在什么地方
应该有多大一块空间放这个代码段数据
哪一段空间是放数据段的数据
然后把它加载内存当中去
然后同时还知道
跳转到uCore哪个地方去执行

Project2
这个代码主要是能够读扇区和分析这个文件
读扇区：函数 readsect
读取扇区 用到了这个in b和out b这种机器指令
（内联汇编来实现的）
它采取了一种IO空间的寻址方式
能够把外设的数据给读到内存中来
这也是一种X86里面的寻址方式
除了正常memory方式之外
还有IO一种寻址方式
不用特别看
只是知道我们这个Bootloader
知道怎么去从哪开始
把相应的扇区给读进来
以及它读多大 读完之后
它就需要去进一步的分析
了解相应的格式ELF的格式

bootmain里面：
它有一个对ELF格式一个判断
它怎么知道读进来这个扇区的数据
是一个ELF格式的文件呢
它其实是读取了一个ELF的header
从磁盘中读取ELFheader
然后判断它的一个特殊的成员变量 e magic
看它是否等于一个特定的值
如果等于特定值
就认为是确实是一个合法的ELF格式的文件


#### 练习6

lidt：load idt

intr_enable：使能中断

在trap.c里面的trap函数

中断服务例程的入口地址在vector.s里面（定义了255个中断号的起始地址）

入口：__alltraps(在trapentry.s里面)

保存pushl
call trap->（trap.c）trap_dispatch（判断中断号）
struct trapframe：打断时的重要信息
内核态：0；用户态：3


```
