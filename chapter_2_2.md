### 进程同步与信号量
**临界区**：一次只允许一个进程进入的该进程的那一段代码。读写信号量的代码一定在临界区
一个非常重要的工作：找出进程中的临界区代码

**基本原则：互斥进入**：如果一个进程在临界区中执行，则其他进程不允许进入
* 这些进程间的约束关系成为**互斥**
* 这保证了是临界区

好的临界区保护原则：
**有空让进**：若干进程要求进入空闲临界区时，应尽快使一进程进入临界区
**优先等待**：从进程发出进入请求到允许进入，不能无限等待

### 轮换法
```c
// 进程P0
/*剩余区*/
while(turn !=0);
/*临界区*/
turn = 1;
/*剩余区*/

// 进程P1
/*剩余区*/
while(turn !=1);
/*临界区*/
turn = 0;
/*剩余区*/
```
满足互斥进入，
但不满足有空让进：如果此时turn=0,turn在上面的剩余区一直进行，空闲临界区就不能被P1使用
### 标记法
```c
/*剩余区*/
flag[0]=true;
while(flag[1]);
/*临界区*/
flag[0]=false;
/*剩余区*/

/*剩余区*/
flag[1]=true;
while(flag[0]);
/*临界区*/
flag[1]=false;
/*剩余区*/
```
满足有空就进：因为标记法是临界区空闲，自己决定是否进入临界区，所以临界区空闲情况下可以直接进入
存在死锁现象

### 结合了标记和轮转两种思想
Peterson's Algorithm（彼得森算法）
```c
//进程P0
/*剩余区*/
flag[0] = true;  // 进程 P0 表示自己想进入临界区
turn = 1;        // 让出优先权给 P1
while(flag[1] && turn == 1);  // 如果 P1 也想进入，并且它的优先权更高，则等待
/* 临界区 */
flag[0] = false; // P0 退出临界区
/* 剩余区 */

//进程P1
/*剩余区*/
flag[1] = true;  // 进程 P1 表示自己想进入临界区
turn = 0;        // 让出优先权给 P0
while(flag[0] && turn == 0);  // 如果 P0 也想进入，并且它的优先权更高，则等待
/* 临界区 */
flag[1] = false; // P1 退出临界区
/* 剩余区 */
```
结合了轮转和标记的方法，交替切换优先级，但是由自己决定是否进入临界区，若临界区空闲，随时都可以进入

flag[0] 和 flag[1]：
flag[i] = true 表示进程Pi想进入临界区。
flag[i] = false 表示进程Pi退出临界区。
turn：
这个变量的作用是控制优先级，让两个进程交替执行。
turn = i 表示让 P_i 先执行，即如果两个进程同时想进入临界区，就让出优先权给 i 号进程。

**互斥性分析**
（1）进入临界区
P0 想进入临界区

设置 `flag[0] = true`，表示自己想进入。
让 turn = 1，让 P1 先执行（如果它也想进）。
`while(flag[1] && turn == 1);`
如果 `flag[1] == false`（P1 没有想进），P0 直接进入临界区。
如果 `flag[1] == true`（P1 也想进），但 turn == 1，P0 需要等待，直到 P1 释放临界区。
P1 想进入临界区

设置 `flag[1] = true`，表示自己想进入。
让 turn = 0，让 P0 先执行（如果它也想进）。
`while(flag[0] && turn == 0)`;
如果 `flag[0] == false`（P0 没有想进），P1 直接进入临界区。
如果 `flag[0] == true`（P0 也想进），但 turn == 0，P1 需要等待，直到 P0 释放临界区。
（2）退出临界区
进程在退出临界区时，会将自己的 flag[i] = false，让对方有机会进入。

### 多进程--面包店算法


### 临界区保护的硬件方法
关闭中断，对于多CPU不合适，因为中断在CPU的INTR寄存器上

### 原子指令法
硬件原子指令Mutex,类似1的信号量:
```c
// 原子指令
boolean TestAndSet(boolean &x) {
    boolean rv = x; // 读取 x 的当前值
    x = true;       // 设置 x 为 true（上锁）
    return rv;      // 返回原始值
}
```
如果 lock 是 false，意味着锁是空闲的，TestAndSet(lock) 返回 false，进而成功获取锁。
如果 lock 是 true，意味着锁已被占用，TestAndSet(lock) 返回 true，进而进入循环等待（自旋）。
**TestAndSet 必须是一个原子操作**
多CPU情况：
多处理机会锁住内存总线，等一个进程执行完，另一个进程才能读lock。所以tsl支持多cpu。

### 内核中的信号量
```c
//sem.c
typedef struct{
    char name[20];
    int value;
    task_struct *queue;
}semtable[20];
sys_sem_open(char *name){
    //semtable中寻找到name，没有就创建，返回下标
}
sys_sem_wait(int sd){
    cli();
    if(semtable[sd].value -- <0){
        //设置自己阻塞；加入semtable[sd].queue中；
        schedule();
        sti;
    }
    sti();
}
main(){
    sd = sem_open("empty");
    for(i=1 to 5){
        sem_wait(sd);
        write(fd, &i, 4);
    }
}
```
操作系统中的信号量在内核态，

磁盘读写的信号量：
buffer_head *bh 是 Linux 内核用于管理磁盘缓冲区（buffer cache）的数据结构。它主要用于文件系统和块设备驱动，用于缓存磁盘块，提高磁盘 I/O 访问效率，减少直接访问磁盘的次数。
结构体包括：磁盘块大小，磁盘块号，等待队列等
```c
//实现一个隐蔽的队列，自己阻塞然后调度让出cpu
void sleep_on(struct task_struct **p){
    // tmp保存队首，然后将自己放到队首，通过栈再找到保存的tmp,幸成链表
    struct task_struct *tmp; 
    // *p表示队列第一个，保存等待队列第一个
    tmp=*p;
    // 将自己放入队首
    *p=current;
    // 设置为阻塞态
    current->state=TASK_UNINTERRUPTIBLE;
    // 进行调度
    schedule();
    // 被阻塞的程序激活后从这里开始，将上一个进程激活，一直到tmp的第一个进程
    if(tmp)
        tmp->state=0;
}
// 一个加锁调度函数，每个进程在读取磁盘前先进行此操作获取锁，如果其余进程正在使用，将自己堵塞加入队列
lock_buffer(buffer_head *bh){
    cli(); // 单核CPU实现原子指令
    // 如果磁盘块已经上锁的话：
    while(bh->b_lock)
    // bh->b_wait是一个等待队列，这里为什么是while，当从整个队列中唤醒后，schedule会先从激活的进程中选取优先级高的执行，优先级高的进程执行完之后将lock置为1，其余进程继续阻塞
        sleep_on(&bh->b_wait);
    // 如果没有进程占用锁，将磁盘上锁
    bh->b_lock = 1;
    sti();
}
//磁盘中断唤醒ublock
void unlock_buffer(struct buffer_head *bh) {
    bh->b_lock = 0;          // 释放锁
    wake_up(&bh->b_wait);    // 唤醒等待的进程
}
//唤醒
wake_up(struct task_struct **p){
    if(p&&*p){
        //激活队首进程，删掉队首指针
        (**p).state=0;
        *p=NULL;
    }
}
bread(int dev, int block){
    struct buffer_head *bh;
    ll_rw_block(READ,bh);
    wait_on_buffer(bh);
}
```
缺点：sleep_on() 可能在 唤醒者（wake_up）执行前 调用 schedule()，从而导致进程无法被正确唤醒。

### 死锁处理
用户如果写出出现死锁，操作系统需要作出处理
 * 资源互斥使用，一旦占有别人无法使用
 * 进程占有了一些资源，又不释放，再去申请其他资源
 * 各自占有的资源和互相申请的资源形成了环路等待
