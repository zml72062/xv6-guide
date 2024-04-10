# 8. 用户态虚拟内存

从这一章开始, 我们要把目光转向 **用户程序**——毕竟, 操作系统自身只是计算机的管理者, 并不完成太多实际工作. 操作系统通常使用 **进程** 这一抽象概念表示一个 **运行中或待运行** 的用户程序. 

xv6 沿用了进程的概念. 对于每一个进程, xv6 内核维护一个 `proc` 结构体: (`kernel/proc.h[84:107]`)
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
可以看到, 即使是 xv6 这样简单的系统, 进程概念的内涵也相当复杂. 在本章, 我们只讨论 xv6 中用户进程的一个侧面——虚拟地址空间.


