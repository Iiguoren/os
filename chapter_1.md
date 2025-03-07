## chapter 1
关键：取址执行
![alt text](pic/ch1_1.png)

**打开电脑，计算机执行的第一个指令是什么**
操作系统执行第一步骤：把操作系统从磁盘载入内存由操作系统的引导扇区完成BOOTSECT.S
![alt text](pic/ch1_2.png)
CS：段地址 IP：偏移地址
BIOS的意思：Basic Input Output System
开机时，CS=0xFFFF;IP=0x0000
寻址0xFFFF0(ROM BIOS映射区)
自动加载BIOS，获取基本的输入输出写入内存，作为冯诺依曼的取址的内容;
之后检查RAM,键盘显示器等
0磁道0扇区为**操作系统引导扇区**，引导扇区只需要执行一次来加载操作系统，读入一个扇区(512字节),写入0x7c00
设置cs=0x07c0, ip=0x0000
引导扇区代码：
bootsect.s(.s汇编代码)
```c
/*
BOOTSEG = 0x07c0
INITSEG = 0x9000
SETUPSEG = 0x9020
将0x07c0移动到0x9000
*/
// 告诉链接程序从start开始
entry start
start:
    // ax地址寄存器赋值0x07c0, ds数据段寄存器赋值ax
    mov ax, #BOOTSEG mov ds, ax 
    // ax地址寄存器赋值0x9000, 再赋值给es额外段寄存器
    mov ax, #INITSEG mov es, ax
    //设置 CX 为 256，表示接下来将执行 256 次操作
    mov cx, #256
    // 初始化段偏移si和di
    sub si, si       sub di, di
    // 重复移动,ds:si->es:di,执行256次，每次两个字节，将0x7c000-0x7c200->0x90000-0x90200
    rep movw
    // 跳转到INITSEG<<4+go，这里go是标号
    jmpi go, INITSEG
```
调用setup模块：
```c
//CS 是代码段寄存器，表示当前代码所在的段地址。这里将 CS 的值加载到 AX，ax为累加寄存器也可通用，即获取当前段的地址。
go: mov ax, cs      将当前代码段的段地址cs:0x9000加载到 ax
//这里的 DS 是数据段寄存器，它被设置为与当前代码段（CS）相同的段地址。通常，代码和数据是分开存放的，但在某些程序中，数据段也可能与代码段相同。
mov ds, ax          将 AX 的值加载到 DS
//同样，ES 是额外段寄存器，这里也被设置为 CS 的地址。这样，ES 和 DS 都指向了同一个段（当前代码段）。
mov es, ax          将 AX 的值加载到 ES
//SS 是堆栈段寄存器，设置为当前段地址。这意味着堆栈也会在当前代码段的地址范围内。
mov ss, ax          将 AX 的值加载到 SS
//SP是堆栈指针寄存器。这里将其设置为 0xFF00，这将确定堆栈的起始位置。堆栈通常从高地址向低地址增长，所以设置为 0xFF00 是一个常见的堆栈初始化方法。
mov sp, #0xff00     将堆栈指针（SP）设置为 0xFF00
load setup:         加载 setup 模块
/*
这些指令准备参数，用于后续的 BIOS 中断调用：
dx 设置为 0x0000。dh磁头号，dl驱动器号
cx 设置为 0x0002，从第二个扇区开始读，第一个是bootsect
bx 设置为 0x0200，是一个偏移值或参数，具体含义取决于上下文。
ax 设置为 0x0200 + SETUPLEN，SETUPLEN 是一个常量，表示 setup 模块的长度。
*/
    mov dx, #0x0000     将 0x0000 存入 DX
    mov cx, #0x0002     将 0x0002 存入 CX
    mov bx, #0x0200     将 0x0200 存入 BX
    // es:bx为内存地址
    // ah = 0x02 读磁盘 al = SETUPLEN=4=扇区数量,从第二个扇区后读四个扇区
    mov ax, #0x0200 + SETUPLEN  将 0x0200 加上 SETUPLEN 赋值给 AX
//int 0x13 是 BIOS 中断，这里调用 int 0x13，将第二个扇区后4个扇区读入0x90200
    int 0x13            调用 BIOS 中断 0x13（磁盘操作中断）
    jnc ok_load_setup   如果无进位（无错误），跳转到 ok_load_setup 标签
    mov dx, #0x0000     如果发生错误，重新初始化 DX 为 0
    mov ax, #0x0000     将 AX 设置为 0，通常用于复位
    int 0x13            再次调用 BIOS 中断，可能是为了复位或重试
    j    load setup          重读 setup 模块
```

setup.s完成OS启动前设置
10000~90000之间的数据全部往前平移了10000位
```c
start: mov ax, #INITSEG    mov ds, ax    mov ah,#0x03
    xor bh,bh int 0x10    mov [0], dx//取光标位置dx
    // 将ax高八位赋值0x88,调用15中断，将ax值写入内存中的第 2 个字节的位置
    mov ah, #0x88     int 0x15     mov [2] ax
    cli // 不允许中断
    mov ad #0x0000 cld
do move:mov es,ax    add ax,#0x1000
// 比较ax寄存器是否=0x9000，如果是跳转到end_move结束移动操作
    cmp ax, #0x9000     jz end_move
// ax的值加载到ds寄存器,将数据段设置为ax的值；清零di
    mov ds,ax   sub di, di
// 清零si
    sub si, si
// cx寄存器设置为0x8000作为计数器
    mov cx #0x8000
// 重复将ds:si移动到es:di，移动0x8000个字即0x10000字节即从0x10000~0x20000->0x00000~0x10000
// 下此循环就是从0x20000~0x30000->0x10000~0x20000
    rep 
    movsw
    jmp do_move
```
**之前将0x07c0段处移到0x9000段处就是为了腾出0地址往上空间**

### 进行保护模式
```c
call empty_8042
...
mov ax. #0x0001 mov cr0, ax
jmpi 0, 8
```
设置cr0寄存器进入**保护模式**
在实模式中，CPU通过段地址和段偏移量寻址。其中段地址保存到段寄存器，包含：CS、SS、DS、ES、FS、GS。段偏移量可以保存到IP、BX、SI、DI寄存器。在汇编代码mov ds:[si], ax中，会将AX寄存器的数据写入到物理内存地址DS * 16 + SI中。

而在保护模式下，也是通过段寄存器和段偏移量寻址，但是此时段寄存器保存的数据意义不同了。
此时的CS和SS寄存器后13位相当于GDT表中某个描述符的索引，即段选择子。第2位存储了TI值（0代表GDT，1代表LDT），第0、1位存储了当前的特权级（CPL）。
**特权级**：
0：内核态
3：用户态
当切换内核态和用户态时，系统通过改变 CS 的值来调整特权级别。
![段选择子](pic/ch1_3.png)
首先CPU需要查找GDT在内存中位置，GDT的位置从GDTR寄存器中直接获取
然后根据DS寄存器得到目标段描述符的物理地址
计算出描述符中的段基址的值加上SI寄存器存储的偏移量的结果，该结果为目标物理地址
将AX寄存器中的数据写入到目标物理地址
### gdt
在实模式下最大可以访问1M大小的地址，进入保护模式后，由于需要内存保护，必须一五一十的记录段的权限–哪个段是系统级别的权限，哪个段时应用级别的权限，这样就可以使得系统可以对所有硬件进行直接访问，限定应用程序对硬件的直接访问，以免怀有恶意的应用程序对计算机破坏。
负责记录段权限的就是GDT，全称全局描述符表，它由64位比特组成，也就是两个字。
cs:ip的寻址模式：
实模式下：cs左移4位+ip;
保护模式下:根据cs查表+ip
![gdt表](pic/ch1_4.png)

### 进入main 函数
**问题：为什么bios为什么只会从0磁道0扇区开始读**
在早期的计算机架构中，BIOS 是计算机开机时最先执行的程序。计算机的硬件设计和启动流程规定了 BIOS 从硬盘的第0磁道、第0扇区（也就是MBR所在的位置）开始读取引导信息。
bootsect->setup->system
操作系统编译后形成Image镜像，放入0磁道0扇区的system对应位置，操作系统就通过setup初始化
head.s(system的第一个文件)
after_page_tables
```
push1 $0 push1 $0 push $0 push1 $L6
push1 $main jmp set_paging
L6
```
函数跳转：C语言中函数调用函数：
```c
int foo(int a, int b) {
    return bar(a, b);
}
```
汇编上：
```
push b         ; 将参数b压栈
push a         ; 将参数a压栈
call bar       ; 调用bar函数
```
将参数a,b依次压入栈中，调用bar函数，同时CPU将foo函数的下一条指令的地址(返回地址)压入堆栈，当bar执行完后跳转回来
被调用函数 bar 会执行相应的操作，并通过返回值将计算结果返回给调用者。返回值通常存放在某个寄存器中（如x86中的 eax 寄存器）。
`mov eax, result  ; 将结果存入 eax 寄存器`
```c
void main(void){
    mem_init();
    trap_init();
    blk_dev_init();
    chr_dev_init();
    tty_init();
    time_init();
    sched_init();
    buffer_init();
    hd_init();
    floppy_init();
    sti();
}
// start_mem可用内存的起始地址
// end_mem可用内存的结尾地址
void mem_init(log start_mem, long end_mem){
    int i;
    for(i=0; i<PAGING_PAGES;i++)
        // 初始化一个mem_map数组
        mem_map[i]=USED;
    i=MAP_NR(start_mem);
    end_mem -= start_mem;
    end_mem>>=12;
    while(end_mem -- >0)
        mem_map[i++]=0;
}
```
两件事情：操作系统读入内存、初始化
![内存分布图](pic/ch1_5.png)