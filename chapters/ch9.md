# 9. 进程的初始化

**进程** 是操作系统中最重要的概念之一, xv6 也不例外. 上一章中, 我们已经从 **虚拟地址空间** 的角度初步了解了 xv6 中的进程. 在本章, 我们将更仔细地看看 xv6 如何初始化一个新的进程.

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
我们在上一章中已经接触了与虚拟地址空间有关的 `pagetable` 和 `trapframe` 字段. 在本章以及之后各章中, 我们还会逐步介绍 `proc` 结构体中的其他字段.

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
void
procinit(void)
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
    + 互斥锁 `wait_lock` 与进程的同步操作有关. 我们将在第 13 章介绍这个主题.
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


## 第一个用户进程
        




