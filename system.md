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