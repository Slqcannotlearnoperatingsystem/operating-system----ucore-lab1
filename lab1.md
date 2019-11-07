#  **实验一：系统软件启动过程**

#### **练习1：理解通过make生成执行文件的过程。**

#### **1.1** **操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)**

**1.1.0 执行make V=命令查看具体执行关于生成kernel和bootblock的相关命令。**

![img](file:///./ksohtml108256\wps1.jpg) 

![img](file:///./ksohtml108256\wps2.jpg) 

 

**1.1.1** **kernel的生成**

![img](file:///./ksohtml108256\wps3.jpg) 

**·**$(call add_files_cc,$(call listf_cc,$(LIBDIR)),libs,)

寻找libs目录下的所有具有.c, .s后缀的文件，并生成相应的.o文件，放置在obj/libs/文件夹下。

**·**$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS)

用于生成kernel的所有子目录下包含的CTYPE文件（.s, .c文件）所对应的.o文件以及.d文件。

**·**$(kernel): tools/kernel.ld

/bin/kernel文件依赖于tools/kernel.ld文件，并且没有指定生成规则。

·$(kernel): $(KOBJS)

kernel文件的生成还依赖于上述生成的obj/libs, obj/kernels下的.o文件，并且生成规则为使用ld链接器将这些.o文件连接成kernel文件，其中ld的-T表示指定使用kernel.ld来替代默认的链接器脚本。

 

**1.1.2** **bootblock的生成**

![img](file:///./ksohtml108256\wps4.jpg) 

·$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc)

将boot/文件夹下的bootasm.S, bootmain.c两个文件编译成相应的.o文件，并且生成依赖文件.d。其中：

-nostdinc: 不搜索默认路径头文件；

-0s: 针对生成代码的大小进行优化，这是因为bootloader的总大小被限制为不大于512-2=510字节。

·$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign

bootblock依赖于bootasm.o, bootmain.o文件与sign文件。

·$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)

使用ld链接器将依赖的.o文件链接成bootblock.o文件。其中：

-N：将代码段和数据段设置为可读可写；

-e：设置入口；

-Ttext：设置起始地址为0X7C00。

·@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)

使用objdump将编译结果反汇编出来，保存在bootclock.asm中，-S表示将源代码与汇编代码混合表示。

·@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)

使用objcopy将bootblock.o二进制拷贝到bootblock.out。

·@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

使用sign程序, 利用bootblock.out生成bootblock。

·$(call add_files_host,tools/sign.c,sign,sign

利用tools/sing.c生成sign.o。

·$(call create_target_host,sign,sign)

利用sign.o生成sign，至此bootblock所依赖的文件均生成完毕。

 

**1.1.3** **使用bootblock, kernel文件来生成ucore.img文件**

![img](file:///./ksohtml108256\wps5.jpg) 

**·$(V)dd if=/dev/zero of=$@ count=10000** 

从/dev/zero文件中获取10000个block，每一个block为512字节，并且均为空字符，并且输出到目标文件ucore.img中。

**·$(V)dd if=$(bootblock) of=$@ conv=notrunc** 

从bootblock文件中获取数据，并且输出到目标文件ucore.img中，-notruct选项表示不要对数据进行删减。

**·$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc** 

从kernel文件中获取数据，并且输出到目标文件ucore.img中, 并且seek = 1表示跳过第一个block，输出到第二个块。

 

**1.1.4 其他部分的代码分析**

**·对各种常量进行初始化。**

![img](file:///./ksohtml108256\wps6.jpg) 

**·推断环境中调用所安装的gcc。**

![img](file:///./ksohtml108256\wps7.jpg) 

如果为定义GCCPREFIX变量，则利用bash中的技巧来推断所使用的gcc命令是什么。

在本部分首先猜测gcc命令的前缀是i386-elf-，因此执行i386-elf-objdump -i命令，2>&1表示将错误输出一起输出到标准输出里，然后通过管道的方式传递给下一条bash命令grep '^elf32-i386$$' >/dev/null 2>&1；>/dev/null这部分表示将标准输出输出到一个空设备里，而输入上一条命令发送给grep的标准输出中可以匹配到'^elf32-i386$$'的话，则说明i386-elf-objdump这一命令是存在的，那么条件满足，由echo输出'i386-elf-'。由于是在$()里的bash命令，这个输出会作为值被赋给GCCPREFIX变量。

如果i386-elf-objdump命令不存在，则猜测使用的gcc命令不包含其他前缀，则继续按照上述方法，测试objdump这条命令是否存在，如果存在则GCCPREFIX为空串，否则之间报错，要求显示地提供gcc的前缀作为GCCPREFIX变量的数值。

**·与上述方法一致，利用bash命令来推断qemu的命令。**

![img](file:///./ksohtml108256\wps8.jpg) 

**·其他涉及到的命令及解释。**

-g：在编译中加入调试信息，便于之后使用gdb进行调试。

-Wall：使能所有编译警告，便于发现潜在的错误。

-O2: 开启O2编译优化。

-fno-builtin: 不承认所有不是以builtin为开头的内建函数。

-ggdb 产生gdb所需要的调试信息（与-g的区别是ggdb的调试信息是专门为gdb而生成的）。

-m32: 32位模式。

-gstabs：以stabs格式输出调试信息，不包括gdb拓展。

-nostdinc: 不搜索默认路径头文件。

-fno-stack-protector: 禁用堆栈保护。

-nostdlib: 该链接器选项表示不链接任何系统标准启动文件和标准库文件，这是因为编译操作系统内核和bootloader是不需要这些启动文件和库就应该能够执行的。

mkdir -p: 允许创建嵌套子目录。

touch -c: 不创建已经存在的文件。

rm -f: 无视任何确认提示。

 

**1.1.5 make V=操作的结果分析**

生成ucore.img需要先生成kernel、bootblock。

生成ucore.img的运行部分为：

 

![img](file:///./ksohtml108256\wps9.jpg) 

其中一些指令的含义如下：

**-dd：用指定大小的块拷贝一个文件，并在拷贝的同时进行指定的转换。** 

**-if=文件名：输入文件名，缺省为标准输入。即指定源文件。< if=input file >** 

**-of=文件名：输出文件名，缺省为标准输出。即指定目的文件。< of=output file >** 

**-count=blocks：仅拷贝blocks个块，块大小等于ibs指定的字节数。** 

**-conv=conversion：用指定的参数转换文件。** 

**-conv=notrunc:不截短输出文件。**

由此可知，ucore生成后创建一个大小为10000字节的块，然后利用设备级转换与拷贝工具 dd进行bootblock写入。

 

#### **1.2** **一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？**

![img](file:///./ksohtml108256\wps10.jpg) 

sign.c中的源代码

·磁盘主引导扇区只有512字节。

·磁盘最后两个字节为0x55、0xAA。

 

#### **练习2：使用qemu执行并调试lab1中的软件。（要求在报告中简要写出练习过程）**

#### **2.1 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。**

由于BIOS是在实模式下运行的，因此需要在tools/gdbinit里进行相应设置：

![img](file:///./ksohtml108256\wps11.jpg) 

在lab1目录下执行make debug，就可以使用gdb单步追踪BIOS的指令执行了：

![img](file:///./ksohtml108256\wps12.jpg) 

 

在gdb界面下，可通过x /2i $pc命令来看BIOS的代码：

![img](file:///./ksohtml108256\wps13.jpg) 

#### **2.2 在初始化位置0x7c00设置实地址断点,测试断点正常。**

在tools/gdbinit结尾加上：

![img](file:///./ksohtml108256\wps14.jpg) 

 

得到如下结果，断点正常： 

 

![img](file:///./ksohtml108256\wps15.jpg) 

 

 

#### **2.3 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。**

在tools/gdbinit结尾加上：

![img](file:///./ksohtml108256\wps16.jpg) 

得到如下结果： 

![img](file:///./ksohtml108256\wps17.jpg) 

 

查看bootasm.S文件：

![img](file:///./ksohtml108256\wps18.jpg) 

查看 bootblock.asm文件：

![img](file:///./ksohtml108256\wps19.jpg) 

 

发现运行出来的结果与bootasm.S和bootblock.asm中的代码相同。

 

#### **练习3：分析bootloader进入保护模式的过程**

bootloader中从实模式进到保护模式的代码保存在lab1/boot/bootasm.S文件下，使用x86汇编语言编写。

bootloader入口为start, bootloader会被BIOS加载到内存的0x7c00处，此时cs=0, eip=0x7c00，在刚进入bootloader的时候，最先执行的操作分别为关闭中断、清除EFLAGS的DF位以及将ax, ds, es, ss寄存器初始化为0：

![img](file:///./ksohtml108256\wps20.jpg) 

接下来为了使得CPU进入保护模式之后能够充分使用32位的寻址能力，需要开启A20，关闭“回卷”机制。该过程主要分为等待8042控制器Inpute Buffer为空，发送P2命令到Input Buffer，等待Input Buffer为空，将P2得到的第二个位（A20选通）置为1，写回Input Buffer。接下来对应上述步骤分析bootasm中的汇编代码——

首先是从0x64内存地址中（映射到8042的status register）中读取8042的状态，直到读取到的该字节第二位（input buffer是否有数据）为0，此时input buffer中无数据：

![img](file:///./ksohtml108256\wps21.jpg) 

接下来往0x64写入0xd1命令，表示修改8042的P2 port：

![img](file:///./ksohtml108256\wps22.jpg) 

接下来继续等待input buffer为空：

![img](file:///./ksohtml108256\wps23.jpg) 

接下来往0x60端口写入0xDF，表示将P2 port的第二个位（A20）选通置为1：

![img](file:///./ksohtml108256\wps24.jpg) 

至此，A20开启，进入保护模式之后可以充分使用4G的寻址能力。

接下来需要设置GDT（全局描述符表），在bootasm.S中已经静态地描述了一个简单的GDT，如下所示; 值得注意的是GDT中将代码段和数据段的base均设置为了0，而limit设置为了2^32-1即4G，此时就使得逻辑地址等于线性地址，方便后续对于内存的操作：

![img](file:///./ksohtml108256\wps25.jpg) 

因此在完成A20开启之后，只需要使用命令lgdt gdtdesc即可载入全局描述符表；接下来只需要将cr0寄存器的PE位置1，即可从实模式切换到保护模式：![img](file:///./ksohtml108256\wps26.jpg)

接下来则使用一个长跳转指令，将cs修改为32位段寄存器，以及跳转到protcseg这一32位代码入口处，此时CPU进入32位模式：

![img](file:///./ksohtml108256\wps27.jpg) 

接下来执行的32位代码功能为：设置ds、es, fs, gs, ss这几个段寄存器，然后初始化栈的frame pointer和stack pointer，然后调用使用C语言编写的bootmain函数，进行操作系统内核的加载，至此，bootloader已经完成了从实模式进入到保护模式的任务。

![img](file:///./ksohtml108256\wps28.jpg) 

 

#### **练习4：分析bootloader加载ELF格式的OS的过程。**

·读取硬盘扇区

首先是waitdisk函数：

![img](file:///./ksohtml108256\wps29.jpg) 

该函数的作用是连续不断地从0x1F7地址读取磁盘的状态，直到磁盘不忙为止。

接下来是readsect函数：

![img](file:///./ksohtml108256\wps30.jpg) 

其基本功能为读取一个磁盘扇区。

代码的具体步骤：

**-等待磁盘直到其不忙；**

**-往0x1F2到0X1F6中设置读取扇区需要的参数，包括读取扇区的数量以及LBA参数；**

**-往0x1F7端口发送读命令0X20；**

**-等待磁盘完成读取操作；**

**-从数据端口0X1F0读取出数据到指定内存中。**

 

在bootmain.c中还有另外一个与读取磁盘相关的函数readseg：

![img](file:///./ksohtml108256\wps31.jpg) 

Readseg的功能为将readsect进行进一步封装，提供能够从磁盘第二个扇区起（kernel起始位置）offset个位置处，读取count个字节到指定内存中，由于上述readsect函数只能就整个扇区进行读取，因此在readseg中，不得不连不完全包括了指定数据的首尾扇区内容也要一起读取进来，此处还有一个小技巧就是将va减去了一个offset%512 Byte的偏移量，这使得就算是整个整个扇区读取，也可以使得要求的读取到的数据在内存中的起始位置恰好是指定的原始的va。

 

·bootmain函数（加载ELF格式的OS）

首先，从磁盘的第一个扇区（第零个扇区为bootloader）中读取OS kenerl最开始的4kB代码，然后判断其最开始四个字节是否等于指定的ELF_MAGIC，用于判断该ELF header是否合法：

![img](file:///./ksohtml108256\wps32.jpg) 

接下来从ELF头文件中获取program header表的位置，以及该表的入口数目，然后遍历该表的每一项，并且从每一个program header中获取到段应该被加载到内存中的位置（Load Address，虚拟地址），以及段的大小，然后调用readseg函数将每一个段加载到内存中，至此完成了将OS加载到内存中的操作：

![img](file:///./ksohtml108256\wps33.jpg) 

最后从ELF header中查询到OS kernel的入口地址，然后使用函数调用的方式跳转到该地址上去：

![img](file:///./ksohtml108256\wps34.jpg) 

 

#### **练习5：实现函数调用堆栈跟踪函数**

根据注释写出相应代码：

![img](file:///./ksohtml108256\wps35.jpg) 

代码的具体步骤：

**-使用read_ebp和read_eip函数获取当前stack frame的base pointer以及call read_eip这条指令下一条指令的地址，存入ebp, eip两个临时变量中；**

**-使用cprint函数打印出ebp, eip的数值；**

**-打印出当前栈帧对应的函数可能的参数；**

**-使用print_debuginfo打印出当前函数的函数名；**

**-根据动态链查找当前函数的调用者(caller)的栈帧；**

**-如果ebp非零并且没有达到规定的STACKFRAME DEPTH的上限，则跳转到（2），继续循环打印栈上栈帧和对应函数的信息。**

 

执行“make qemu”结果：

![img](file:///./ksohtml108256\wps36.jpg) 

 

bootloader设置的堆栈从0x7c00开始，使用”call bootmain”转入bootmain函数。 call指令压栈，所以bootmain中ebp为0x7bf8。

 

#### **练习6：完善中断初始化和处理**

#### **6.1** **中断向量表中一个表项占多少字节？其中哪几位代表中断处理代码的入口？**

IDT中的每一个表项均占8个字节。

其中最开始2个字节和最末尾2个字节定义了offset，第16-31位定义了处理代码入口地址的段选择子，使用其在GDT中查找到相应段的base address，加上offset就是中断处理代码的入口。

 

#### **6.2 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。**

根据注释写出相应代码：

![img](file:///./ksohtml108256\wps37.jpg) 

代码的具体步骤：

**-对整个idt数组进行初始化；**

**-把所有的中断都初始化为内核级的中断；**

**-使用lidt指令加载中断描述符表。**

 

执行“make qemu”结果：

![img](file:///./ksohtml108256\wps38.jpg) 

###  

#### **6.3** **请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数**。

根据注释写出相应代码：

![img](file:///./ksohtml108256\wps39.jpg) 

代码的具体步骤：

**-每次时钟中断之后ticks就会加一 当加到TICK_NUM次数时 打印并重新开始；**

**-打印字符串；**

 

执行“make qemu”结果：

![img](file:///./ksohtml108256\wps40.jpg) 