## chapter 1
关键：取址执行
![alt text](pic/ch1_1.png)

**打开电脑，计算机执行的第一个指令是什么**
![alt text](pic/ch1_2.png)
CS：段地址 IP：偏移地址
BIOS的意思：Basic Input Output System
开机时自动加载BIOS，获取基本的输入输出写入内存，作为冯诺依曼的取址的内容;
之后检查RAM,键盘显示器等
0磁道0扇区为操作系统引导扇区
引导扇区代码：
bootsect.s(.s汇编代码)