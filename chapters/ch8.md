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
可以看到, 即使是 xv6 这样简单的系统, 进程概念的内涵也相当复杂. 在本章, 我们只讨论 xv6 中用户进程的一个侧面——**虚拟地址空间**. 

## 进程的页表

操作系统的一个很重要的任务, 就是实现 **隔离**: 用户进程绝不应该访问属于操作系统的内存, 也绝不应该访问属于其他进程的内存. 但是, 最好又不要让用户进程感知到隔离的存在; 也就是说, 要给每一个用户进程营造出这样的假象, 好像只有它自己在独占整个内存一样.

为了实现以上目的, 一个聪明的做法是: 为每个用户进程维护一个一模一样的虚拟地址空间, 但是让这些虚拟地址空间采用不同的方式映射到物理内存. 因为用户程序只会使用虚拟地址 (地址翻译是硬件默默完成的), 用户并不知道 (也无需知道) 他们的程序实际在使用物理内存的哪些部分. 他们只需要认为自己有一个 "不受打扰的" 虚拟地址空间可供使用就行了.

> 通常也会为内核维护一个虚拟地址空间 (上一章中我们看到 xv6 正是这么做的). 这样, 用户态和内核态都可以从虚拟内存机制中获得便利.

以上做法直接的后果就是: 同一个虚拟地址在内核中, 或者在不同的用户进程中, 可能对应着完全不同的物理地址. 因为虚拟地址到物理地址的翻译是基于页表进行的, 这就意味着我们需要 **为每一个用户进程维护一个独立的页表**. 

在 xv6 中, 每个进程的 `proc` 结构体中都有 `pagetable` 字段, 用来存储该进程的一级页表物理基址. 函数 `proc_pagetable()` 和 `proc_freepagetable()` (均定义在 `kernel/proc.c` 中) 分别用来 **初始化** 及 **回收** 某个进程的页表.

### `proc_pagetable()` 函数

`proc_pagetable()` 定义在 `kernel/proc.c[174:206]`:
```c
// Create a user page table for a given process, with no user memory,
// but with trampoline and trapframe pages.
pagetable_t proc_pagetable(struct proc *p)
{
  pagetable_t pagetable;

  // An empty page table.
  pagetable = uvmcreate();
  if(pagetable == 0)
    return 0;

  // map the trampoline code (for system call return)
  // at the highest user virtual address.
  // only the supervisor uses it, on the way
  // to/from user space, so not PTE_U.
  if(mappages(pagetable, TRAMPOLINE, PGSIZE,
              (uint64)trampoline, PTE_R | PTE_X) < 0){
    uvmfree(pagetable, 0);
    return 0;
  }

  // map the trapframe page just below the trampoline page, for
  // trampoline.S.
  if(mappages(pagetable, TRAPFRAME, PGSIZE,
              (uint64)(p->trapframe), PTE_R | PTE_W) < 0){
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
  }

  return pagetable;
}
```
它为进程 `p` 创建并初始化页表. 这个函数的流程如下:
* 首先调用 `uvmcreate()` (定义在 `kernel/vm.c[194:205]`) 创建一个全为零的物理页作为 **页表页**:
  ```c
  // create an empty user page table.
  // returns 0 if out of memory.
  pagetable_t uvmcreate()
  {
    pagetable_t pagetable;
    pagetable = (pagetable_t) kalloc();
    if(pagetable == 0)
      return 0;
    memset(pagetable, 0, PGSIZE);
    return pagetable;
  }
  ```
> 有人可能会疑虑 `kalloc()` 和 `kfree()` 的语义是否因为内核态启用了地址翻译机制而发生了变化, 因为这两个函数本来是直接操纵物理地址的. 好在 xv6 内核将虚拟地址范围 `[0, 0x88000000)` 直接映射到相同的物理地址, 因此只要我们是在内核态中, 仍可以使用物理地址正确地访存.

* 接着, 将用户虚拟地址空间中最顶部的一页 (以 `TRAMPOLINE` 作为虚拟基址) 也映射到 `trampoline` 所在的物理页:
  ```c
  if(mappages(pagetable, TRAMPOLINE, PGSIZE,
              (uint64)trampoline, PTE_R | PTE_X) < 0){
    uvmfree(pagetable, 0);
    return 0;
  }
  ```
> 值得注意的是, 无论对于 xv6 内核还是任一用户进程而言, 以 `TRAMPOLINE` 为基址的虚拟页总是被映射到相同的物理页 (这个物理页称为 trampoline 页). 这意味着, 即使页表被切换 (通常由于从用户态陷入内核, 或者从内核返回用户态), 我们也能通过相同的虚拟地址访问 trampoline 页的内容.
>
> 事实上, 之后会看到: trampoline 页中存放了 xv6 在用户态和内核态之间切换的关键代码.
>

* 最后, 将虚拟地址空间中 `TRAPFRAME` 指向的一页映射到 `proc` 结构体中的 `trapframe` 字段所指物理页:
  ```c
  // TRAPFRAME = TRAMPOLINE - PGSIZE;
  // "p->trapframe" has been allocated in physical memory
  if(mappages(pagetable, TRAPFRAME, PGSIZE,
              (uint64)(p->trapframe), PTE_R | PTE_W) < 0){
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
  }
  ```
  虚拟地址 `TRAPFRAME` 正好位于 `TRAMPOLINE` 下方一页的位置.

> xv6 内核为每个存活进程都分配一个物理页, 称为 **陷入帧** (trap frame). 在进程的 `proc` 结构体中, `trapframe` 字段就指向陷入帧. 陷入帧的内容定义为下列结构体: (`kernel/proc.h[31:80]`)
> ```c
> struct trapframe {
>   /*   0 */ uint64 kernel_satp;   // kernel page table
>   /*   8 */ uint64 kernel_sp;     // top of process's kernel stack
>   /*  16 */ uint64 kernel_trap;   // usertrap()
>   /*  24 */ uint64 epc;           // saved user program counter
>   /*  32 */ uint64 kernel_hartid; // saved kernel tp
>   /*  40 */ uint64 ra;
>   /*  48 */ uint64 sp;
>   /*  56 */ uint64 gp;
>   /*  64 */ uint64 tp;
>   /*  72 */ uint64 t0;
>   /*  80 */ uint64 t1;
>   /*  88 */ uint64 t2;
>   /*  96 */ uint64 s0;
>   /* 104 */ uint64 s1;
>   /* 112 */ uint64 a0;
>   /* 120 */ uint64 a1;
>   /* 128 */ uint64 a2;
>   /* 136 */ uint64 a3;
>   /* 144 */ uint64 a4;
>   /* 152 */ uint64 a5;
>   /* 160 */ uint64 a6;
>   /* 168 */ uint64 a7;
>   /* 176 */ uint64 s2;
>   /* 184 */ uint64 s3;
>   /* 192 */ uint64 s4;
>   /* 200 */ uint64 s5;
>   /* 208 */ uint64 s6;
>   /* 216 */ uint64 s7;
>   /* 224 */ uint64 s8;
>   /* 232 */ uint64 s9;
>   /* 240 */ uint64 s10;
>   /* 248 */ uint64 s11;
>   /* 256 */ uint64 t3;
>   /* 264 */ uint64 t4;
>   /* 272 */ uint64 t5;
>   /* 280 */ uint64 t6;
> };
> ```
> 可以设想, 当指令流从用户进程切换到内核时, 必须保存用户进程使用的寄存器, 并恢复出内核所需的关键寄存器; 从内核切换到用户进程时, 则需要相反的步骤. **给每个进程分配陷入帧, 是为了预留出内存空间存储这些寄存器, 以服务该进程与内核之间的切换.**
  
下图摘自 xv6 book: 它展示了用户进程虚拟地址空间的结构.

![xv6 用户进程虚拟地址空间](/chapters/ch8-figs/useraddr.png "xv6 用户进程虚拟地址空间")

上面的 `proc_pagetable()` 函数实际上只把 trampoline 和 trapframe 这两个虚拟页映射到了物理页; 进程虚拟地址空间的其他部分 (heap, stack, data, text 等等) 暂时都是无效的.

要指出一点: 虽然 trampoline 页和 trapframe 页是用户态虚拟地址空间的一部分, 但它们对应的物理页却 **在内核中**. 从而我们不希望用户进程访问这两个虚拟页. 好在 `proc_pagetable()` 函数充分考虑到了这个担忧——不妨查看一下 trampoline 页和 trapframe 页的权限: trampoline 页是 `PTE_R | PTE_X`, 而 trapframe 页是 `PTE_R | PTE_W`. **这两页的页表项都没有把 `U` 位设为 1.** 

> 除去进程虚拟地址空间顶部的两页之外, 剩余的虚拟页都将是用户可访问的.

### `proc_freepagetable()` 函数

