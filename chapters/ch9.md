# 9. 进程的初始化

**进程** 是操作系统中最重要的概念之一, xv6 也不例外. [上一章](ch8.md)中, 我们已经从 **虚拟地址空间** 的角度初步了解了 xv6 中的进程. 在本章, 我们将更仔细地看看 xv6 如何初始化一个新的进程.

回忆一下, xv6 使用 **`proc` 结构体** 来描述一个进程, 具体内容为
```c
// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // wait_lock must be held when using this:
  struct proc *parent;         // Parent process

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```
我们在[上一章](ch8.md)中已经接触了与虚拟地址空间有关的 `pagetable` 和 `trapframe` 字段. 在本章以及之后各章中, 我们还会逐步介绍 `proc` 结构体中的其他字段.

## 进程表

xv6 维护一个 `proc` 结构体数组, 作为存放进程状态的 "容器": (`kernel/proc.c[11]`)
```c
struct proc proc[NPROC];    // NPROC = 64;
```
这里 `NPROC` 表示 xv6 最多允许多少个进程存活. 这个数组就叫做 **进程表**.

在 `main()` 函数中, 通过调用 `procinit()` 函数初始化进程表: (`kernel/main.c[22]`)
```c
    procinit();      // process table
```
`procinit()` 函数定义在 `kernel/proc.c[46:59]`:
```c
// initialize the proc table.
void procinit(void)
{
  struct proc *p;
  
  initlock(&pid_lock, "nextpid");
  initlock(&wait_lock, "wait_lock");
  for(p = proc; p < &proc[NPROC]; p++) {
      initlock(&p->lock, "proc");
      p->state = UNUSED;
      p->kstack = KSTACK((int) (p - proc));
  }
}
```
它做了以下工作:
* 初始化两个全局互斥锁 `pid_lock` 和 `wait_lock`.
    + xv6 中的每个进程都有一个唯一的 **进程号** (PID). xv6 通过查看递增的计数器 `nextpid` 来决定给新进程分配什么 PID. 互斥锁 `pid_lock` 用来保护全局变量 `nextpid`.
    + 互斥锁 `wait_lock` 与进程的同步操作有关. 我们将在[第 14 章](ch14.md)介绍这个主题.
* 对于进程表中的每一个进程 `p`,
    + 初始化一个互斥锁 `p->lock`. 这个锁用于保护进程 `p` 的状态信息 (PID, 是否存活, 是否睡眠, 等等).
    + 将进程 `p` 的状态 (`p->state`) 标为 "非存活" (`UNUSED`).
    > xv6 的进程一共有 6 种状态: (`kernel/proc.h[82]`)
    > ```c
    > enum procstate { 
    >   UNUSED, USED, SLEEPING, RUNNABLE, RUNNING, ZOMBIE
    > };
    > ```
    > 它们的具体含义将在之后逐步介绍.
    + 将 `p->kstack` 指向内核虚拟地址空间中, 进程 `p` 的内核栈的起始虚拟地址
        ```c
        KSTACK((int) (p - proc)) = TRAMPOLINE - ((int) (p - proc) + 1) * 2*PGSIZE;
        ```

## 进程的创建与回收

xv6 通过调用 `allocproc()` 函数创建一个 **"干净的"** 新进程: (`kernel/proc.c[105:150]`)
```c
static struct proc* allocproc(void)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == UNUSED) {
      goto found;
    } else {
      release(&p->lock);
    }
  }
  return 0;

found:
  p->pid = allocpid();
  p->state = USED;

  // Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  return p;
}
```
这个函数分为以下几步:
* 查阅整个进程表, 寻找是否有状态为 `UNUSED` 的进程. 在访问进程 `p` 的 `p->state` 字段之前, 需获得互斥锁 `p->lock`.
* 如果找到了状态为 `UNUSED` 的进程 `p`, 就对它做初始化工作: (注意此时 `p->lock` 没有被释放, 因此我们可以修改 `p` 的状态字段)
    + 使用 `allocpid()` 为 `p` 分配一个 PID:
        ```c
        p->pid = allocpid();
        ```
        `allocpid()` 函数定义在 `kernel/proc.c[92:103]`:
        ```c
        int allocpid()
        {
          int pid;
          
          acquire(&pid_lock);
          pid = nextpid;
          nextpid = nextpid + 1;
          release(&pid_lock);
          return pid;
        }
        ```
        它取出全局变量 `nextpid` 的值作为 PID, 并给 `nextpid` 加 1.
    + 将 `p` 的状态 (`p->state`) 改为 `USED`.
        ```c
        p->state = USED;
        ```
    + 为 `p` 分配一个物理页, 作为陷入帧:
        ```c
        if((p->trapframe = (struct trapframe *)kalloc()) == 0){
          freeproc(p);
          release(&p->lock);
          return 0;
        }
        ```
    + 为 `p` 构建页表:
        ```c
        p->pagetable = proc_pagetable(p);
        if(p->pagetable == 0){
          freeproc(p);
          release(&p->lock);
          return 0;
        }
        ```
        我们在[上一章](ch8.md)已经详述过这一步的细节. 特别地, 虚拟地址 `TRAPFRAME` 出发的虚拟页会被映射到上一步刚刚创建的 **陷入帧**.
    + 初始化 `p` 的 **上下文**. 在 xv6 中, 进程的上下文被定义为如下结构体: (`kernel/proc.h[1:19]`)
        ```c
        // Saved registers for kernel context switches.
        struct context {
          uint64 ra;
          uint64 sp;

          // callee-saved
          uint64 s0;
          uint64 s1;
          uint64 s2;
          uint64 s3;
          uint64 s4;
          uint64 s5;
          uint64 s6;
          uint64 s7;
          uint64 s8;
          uint64 s9;
          uint64 s10;
          uint64 s11;
        };   
        ```
        这个结构体保存进程执行过程中的 `ra` (返回地址), `sp` (栈指针) 寄存器, 以及 `s0` ~ `s11` 这 12 个 **被调用者保存** (callee-saved) 的寄存器. `proc` 结构体中的 `context` 字段就是 **指向进程上下文的指针**.

        对于一个新进程, xv6 按如下方式初始化它的上下文:
        ```c
        memset(&p->context, 0, sizeof(p->context));
        p->context.ra = (uint64)forkret;
        p->context.sp = p->kstack + PGSIZE;
        ```
        也就是说, 它把进程 `p` 的上下文结构体中的 `ra` 字段设为 `forkret()` 函数的地址 (该函数定义在 `kernel/proc.c[512:531]`), 把 `sp` 字段设为 `p` 的内核栈栈顶, 其余字段全设为零.

        > 读者可能会注意到 `context` 结构体是 `trapframe` 结构体的真子集. 而且, 这两种数据结构具有相近的含义——它们都 (在某种意义上) 囊括了进程的 **寄存器上下文**. 为什么需要这两种结构体同时存在呢? 事实上, xv6 给这两种结构体赋予了不同的功能:
        > * `context` 结构体仅用于 **进程调度**. 所谓进程调度, 就是把执行权从一个进程转移给另一个进程. 在[第 12 章](ch12.md)中我们将会看到, xv6 通过两个函数 `scheduler()` (`kernel/proc.c[437:472]`) 和 `sched()`(`kernel/proc.c[474:499]`) 的相互配合, 来完成进程调度. 其中
        >   + `scheduler()` 遍历进程表, 选择下一个可调度的进程 (不妨假设进程 `p` 被选中执行). 紧接着, `scheduler()` 会调用 `swtch()` 函数. `swtch()` 将当前上下文保存到 `mycpu()->context` 中, 然后从 `p.context` 恢复出 `ra`, `sp` 和所有 callee-saved 寄存器. 当 `swtch()` 通过 `ret` 指令返回时, `pc` 会被设置为 `ra` 的值. 换言之, 程序接下来跳转到进程 `p` 上一次停下来的地方执行.
        >   + 当进程 `p` 出于某些原因 (例如调用 `yield()`, `sleep()`, `exit()`) 决定放弃 CPU 时, 会调用 `sched()` 函数. `sched()` 也会调用 `swtch()`, 将当前的 `ra`, `sp` 和 callee-saved 寄存器保存到 `p.context` 中, 然后从 `mycpu()->context` 恢复出调度器的上下文, 把控制权移交给 `scheduler()`, 让它选择下一个进程执行.
        > * `trapframe` 结构体用于中断引起的 **陷入** 及 **陷入返回**. 从用户态陷入内核态时, 用户进程 `p` 所使用的全部寄存器会被保存到 `p.trapframe` 指向的页 (也就是进程 `p` 的陷入帧) 中; 返回用户态时, 又会从 `p` 的陷入帧中恢复寄存器的值.
        >
        > 和 `trapframe` 相比, 可以认为 `context` 是相对 "轻量级" 的上下文. 一个很自然的问题是: 为什么对于进程调度而言, 保存和恢复 "轻量级" 的上下文就足够了, 但是对于陷入及陷入返回, 则需要保存和恢复 "完整" 的上下文呢? 关键的区别在于, 进程调度总是由操作系统 **主动发起** 的, 而陷入是由 **中断** 引起的. 具体来说,
        > * 引发调度的唯一途径是调用 `swtch()` 函数. 因此, 编译器会保证: 在开始调度之前, `swtch()` 的调用者已经把 caller-saved 寄存器都保存在了栈上. 当控制流下一次回到 `swtch()` 的调用者时, 只需要恢复 `sp`, 就可以从栈上正确地恢复所有 caller-saved 寄存器. 这就是为什么 `swtch()` 只需要保存和恢复 **`ra`, `sp` 和 callee-saved 寄存器**.
        > * 与进程调度不同, 陷入是由硬件的中断机制引起的. 中断具有 **不可预测性**: 处理器每执行完一条指令都有可能被中断打断. 由于中断的这一特点, 编译器不可能也无需为将要发生的中断保存 caller-saved 寄存器. 因此, 必须由中断处理函数 (`uservec()` 和 `userret()`) 来保证中断发生前后 **所有通用寄存器** 的完整性.
        
* 如果进程创建成功, 就返回其 `proc` 结构体的首地址, 并且 **不释放 `p->lock`**; 否则, 返回 0 并释放 `p->lock`.

从 `allocproc()` 创建的进程是 "干净的", 因为它的虚拟地址空间中用户可访问的部分是空的; 此外, 还没有为它进行文件系统的初始化工作. 直到我们把用户编写的程序 **加载** 到进程的地址空间中 (这意味着分配一些物理页, 并把程序中的代码、数据复制到物理页中), 进程的状态才会变成 **可运行的** (`RUNNABLE`, 而不仅仅是 `USED`).

与创建进程相对应的操作是 **回收进程**. xv6 使用函数 `freeproc()` 回收一个进程: (`kernel/proc.c[152:172]`)
```c
static void freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  p->pagetable = 0;
  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;
  p->state = UNUSED;
}
```
该函数回收进程的陷入帧 (`p->trapframe`), 通过 `proc_freepagetable()` 清除用户程序占用的全部物理内存并回收页表, 重置 `proc` 结构体字段, 最后把进程状态 (`p->state`) 设为 `UNUSED`. 在 `p` 上调用 `freeproc()` 时必须持有 `p->lock`.

## 第一个用户进程

虽然 `allocproc()` 已经实现了进程的创建, 但这样创建出来的进程还不能运行. 为了让一个进程变为可运行的, 通常需要把磁盘上的可执行文件映射到内存的地址空间. (我们暂不展开具体的过程, 因为它涉及文件系统.) 

假如磁盘上还没有任何可执行程序怎么办呢? xv6 的解决办法是: 把一个预先写好的可执行程序 (`user/initcode`) 的二进制代码放到内核中, 随内核加载到内存. 待 xv6 启动后, 内核代码会自动创建第一个用户进程, 并且把 `user/initcode` 的代码放到该进程的虚拟地址空间中.

在 `kernel/proc.c[218:229]`, 就展示了 `user/initcode` 的二进制代码:
```c
uchar initcode[] = {
  0x17, 0x05, 0x00, 0x00, 0x13, 0x05, 0x45, 0x02,
  0x97, 0x05, 0x00, 0x00, 0x93, 0x85, 0x35, 0x02,
  0x93, 0x08, 0x70, 0x00, 0x73, 0x00, 0x00, 0x00,
  0x93, 0x08, 0x20, 0x00, 0x73, 0x00, 0x00, 0x00,
  0xef, 0xf0, 0x9f, 0xff, 0x2f, 0x69, 0x6e, 0x69,
  0x74, 0x00, 0x00, 0x24, 0x00, 0x00, 0x00, 0x00,
  0x00, 0x00, 0x00, 0x00
};
```
在 `main()` 函数中, 当 0 号 CPU 完成了全部必要的初始化工作后, 它就调用 `userinit()` 创建出第一个用户进程: (`kernel/main.c[31]`)
```c
    userinit();      // first user process
```
`userinit()` 函数定义在 `kernel/proc.c[231:255]`:
```c
void userinit(void)
{
  struct proc *p;

  p = allocproc();
  initproc = p;
  
  // allocate one user page and copy initcode's instructions
  // and data into it.
  uvmfirst(p->pagetable, initcode, sizeof(initcode));
  p->sz = PGSIZE;

  // prepare for the very first "return" from kernel to user.
  p->trapframe->epc = 0;      // user program counter
  p->trapframe->sp = PGSIZE;  // user stack pointer

  safestrcpy(p->name, "initcode", sizeof(p->name));
  p->cwd = namei("/");

  p->state = RUNNABLE;

  release(&p->lock);
}
```
这个函数做以下工作:
* 调用 `allocproc()` 创建一个新进程, 并把其 `proc` 结构体的首地址放到全局变量 `initproc` 中. 这个全局变量定义在 `kernel/proc.c[13]`:
    ```c
    struct proc *initproc;
    ```
    注意, 这一步结束之后, 我们仍然持有锁 `initproc->lock`.
* 调用 `uvmfirst()`, 将 `initcode` 数组中的二进制代码拷贝到进程 `initproc` 虚拟地址空间的第一页 (零地址出发的一页).
* 将 `initproc` 自身占用的内存大小 (仅包括用户代码、数据的大小) 设置为一页.
    ```c
    p->sz = PGSIZE;
    ```
* 设置陷入帧中的 `epc` 与 `sp` 字段.
    ```c
    p->trapframe->epc = 0;      // user program counter
    p->trapframe->sp = PGSIZE;  // user stack pointer
    ```
    我们在之后会看到, 当 xv6 从内核态回到用户态之前, 会把 `sepc` 寄存器设置为 `p->trapframe->epc` 保存的值, 把栈指针 `sp` 设置为 `p->trapframe->sp` 保存的值. (其中 `p` 是返回用户态后将要运行的进程.)
* 将 `initproc` 的进程名设置为 `initcode`.
    ```c
    safestrcpy(p->name, "initcode", sizeof(p->name));
    ```
* 初始化 `initproc` 的工作目录. (我们会在介绍文件系统时解释细节.)
    ```c
    p->cwd = namei("/");
    ```
* 将 `initproc` 的状态设置为 `RUNNABLE`.
* 释放 `initproc->lock`.

调用 `userinit()` 后, `initproc` 就成为 xv6 中第一个 **可运行** 的进程. 所谓 "可运行", 就是指内核中的调度器 (`scheduler()` 函数) 可以把控制权转交给这个进程.

到现在为止, 我们已经知道了 xv6 怎样创建并初始化一个进程. 所有这些工作都是在 **内核态** 中完成的. 一个很自然的问题是: **什么时候才会离开内核, 真正去执行用户进程呢?** 为了回答这个问题, 我们需要了解 xv6 在 **内核态** 和 **用户态** 之间切换的机制. 这将是我们接下来两章的主题.

