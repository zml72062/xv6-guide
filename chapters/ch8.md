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

> 通常也会为内核维护一个虚拟地址空间 ([上一章](ch7.md)中我们看到 xv6 正是这么做的). 这样, 用户态和内核态都可以从虚拟内存机制中获得便利.

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
  可能有些奇怪, `TRAMPOLINE` 指向的虚拟页位于 **用户进程虚拟地址空间** 中, 但它却被映射到 **内核** 代码段中的一个物理页 (trampoline 页)! 很明显, 我们不希望用户进程拥有对 `TRAMPOLINE` 虚拟页的访问权限. 为此, 我们把这一页的权限位设置成 `R=1`, `X=1`, **但 `U=0`**.

* 最后, 将虚拟地址空间中 `TRAPFRAME` 指向的一页映射到进程 `p` 的 `proc` 结构体中的 `trapframe` 字段所指物理页:
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
  虚拟地址 `TRAPFRAME` 正好位于 `TRAMPOLINE` 下方一页的位置. 由于 `TRAPFRAME` 对应的物理页也属于内核, 我们同样将 `U` 权限位设置成 0.

> xv6 内核会为每个刚创建的进程都分配一个物理页, 称为 **陷入帧** (trap frame). 假如进程的 `proc` 结构体为 `p`, 则 `p.trapframe` 字段就是指向陷入帧的指针. 陷入帧的内容定义为下列结构体: (`kernel/proc.h[31:80]`)
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
> 结构体 `trapframe` 中, 除了前 5 个字段外, 剩余 31 个字段正好对应于 RISC-V 的 31 个通用寄存器 (除了恒为零的 `zero` 寄存器外).
>
> 在[第 11 章](ch11.md)中将会看到, 陷入帧的功能是 **服务当前进程与内核之间的切换**. 具体来讲, 从内核态返回任何进程之前, 都必须先从该进程的陷入帧中恢复所有通用寄存器的值 (通过调用 `userret()` 函数, 见 `kernel/trampoline.S[100:151]`). 反过来, 任何进程陷入内核的第一步都是将所有通用寄存器保存到它自己的陷入帧中 (通过调用 `uservec()` 函数, 见 `kernel/trampoline.S[20:98]`).
  
下图摘自 xv6 book: 它展示了用户进程虚拟地址空间的结构.

![xv6 用户进程虚拟地址空间](/chapters/ch8-figs/useraddr.png "xv6 用户进程虚拟地址空间")

`proc_pagetable()` 函数实际上只把 trampoline 和 trapframe 这两个虚拟页映射到了物理页; 进程虚拟地址空间的其他部分 (heap, stack, data, text 等等) 暂时都是无效的.

正如上面提到的那样, 虽然 trampoline 页和 trapframe 页是用户态虚拟地址空间的一部分, 但它们对应的物理页 **在内核中**; 用户程序不可访问这两个虚拟页. 除去这两页之外, 进程虚拟地址空间的其他部分都是用户可访问的.

### `proc_freepagetable()` 函数

`proc_freepagetable()` 函数可以回收一个进程的页表. 这包括两部分工作:
* 回收进程自身所使用的全部物理内存
* 回收进程页表占据的物理内存

`proc_freepagetable()` 的代码参见 `kernel/proc.c[208:216]`:
```c
// Free a process's page table, and free the
// physical memory it refers to.
void proc_freepagetable(pagetable_t pagetable, uint64 sz)
{
  uvmunmap(pagetable, TRAMPOLINE, 1, 0);
  uvmunmap(pagetable, TRAPFRAME, 1, 0);
  uvmfree(pagetable, sz);
}
```
这里, 参数 `pagetable` 表示 **进程一级页表的物理基址**, `sz` 表示 **进程自身占用的物理内存大小**.

在 `proc_freepagetable()` 中涉及两个辅助函数: `uvmunmap()` 与 `uvmfree()`. 我们首先来看 `uvmunmap()`: (`kernel/vm.c[167:192]`)
```c
void uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)
      panic("uvmunmap: not mapped");
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
```
这个函数把 (页对齐的) 虚拟地址 `va` 出发的 `npages` 个虚拟页设为无效的, 并且当 `do_free == 1` 时, 还会回收相应的物理页. 具体来说, 对于虚拟地址范围 `[va, va + npages * PGSIZE)` 中的每个虚拟页 `a`,
* 找到该虚拟页对应的三级页表条目, 设为 `pte`
  ```c
  if((pte = walk(pagetable, a, 0)) == 0)
    panic("uvmunmap: walk");
  ```
* 确认该页表条目的有效位 `V` 为 1
  ```c
  if((*pte & PTE_V) == 0)
    panic("uvmunmap: not mapped");
  ```
* 确认该页表条目对应一个叶子页 (权限位 `R`, `W`, `X` 不全为零)
  ```c
  // PTE_FLAGS(*pte) = (*pte) & 0x3FF;
  // selects last 10 bits (permission bits) of (*pte).
  if(PTE_FLAGS(*pte) == PTE_V)
    panic("uvmunmap: not a leaf");
  ```
* 如果参数 `do_free == 1`, 就读出 `pte` 中的物理页号, 把相应的物理页回收
  ```c
  if(do_free) {
    uint64 pa = PTE2PA(*pte);
    kfree((void*)pa);
  }
  ```
* 重置 `pte` 的内容为全零
  ```c
  *pte = 0;
  ```

从而, `proc_freetable()` 的前两行代码
```c
  uvmunmap(pagetable, TRAMPOLINE, 1, 0);
  uvmunmap(pagetable, TRAPFRAME, 1, 0);
```
取消了用户进程页表中 `TRAMPOLINE` 和 `TRAPFRAME` 页到物理内存的映射. 

> `proc_freetable()` 并不回收 `TRAMPOLINE` 和 `TRAPFRAME` 页对应的物理内存 (即上述调用中 `do_free == 0`). 理由是:
> * `TRAMPOLINE` 页对应的物理页位于内核代码段, 应当常驻内存, 且它根本不能用 `kfree()` 回收 
> * `TRAPFRAME` 页在物理内存中被映射到进程的陷入帧; 当进程消亡时, 陷入帧会在 `freeproc()` 函数 (`kernel/proc.c[152:172]`) 中回收, 而不是在这里回收

再来看 `uvmfree()` 函数. 它定义在 `kernel/vm.c[289:297]`:
```c
void uvmfree(pagetable_t pagetable, uint64 sz)
{
  if(sz > 0)
    uvmunmap(pagetable, 0, PGROUNDUP(sz)/PGSIZE, 1);
  freewalk(pagetable);
}
```
它首先调用 `uvmunmap()`, 将进程虚拟地址空间中 **零地址出发的 `PGROUNDUP(sz)/PGSIZE` 页** 取消映射, 并 **回收相应的物理页** (`do_free == 1`). 

> 前面已经提到, 进程虚拟地址空间中除最顶部的两页 (trampoline 和 trapframe) 外, 其余部分都是用户进程可访问的. 也就是说, 用户进程自身占用的内存对应于虚拟地址空间中零地址出发的若干页. 上述操作把这些内存全部清除.

接下来, `uvmfree()` 调用 `freewalk()`, 将各级页表占用的物理内存回收. `freewalk()` 函数定义在 `kernel/vm.c[269:287]`:
```c
void freewalk(pagetable_t pagetable)
{
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      freewalk((pagetable_t)child);
      pagetable[i] = 0;
    } else if(pte & PTE_V){
      panic("freewalk: leaf");
    }
  }
  kfree((void*)pagetable);
}
```
这个函数遍历页表 `pagetable` 中的 512 个页表项, 并检查每一个页表项 `pte` 是否指向下一级页表:
```c
(pte & PTE_V) != 0 && (pte & (PTE_R | PTE_W | PTE_X)) == 0
```
如果是的话, 就递归地对下一级页表调用 `freewalk()`:
```c
  uint64 child = PTE2PA(pte);
  freewalk((pagetable_t)child);
  pagetable[i] = 0;
```
直到全部 512 个页表项都被清除之后, 回收 `pagetable` 所使用的页表页:
```c
  kfree((void*)pagetable);
```
> 在调用 `freewalk()` 之前, 需确认 **已取消页表中所有叶子页到物理内存的映射**. 换言之, 页表中的每个条目要么是无效的, 要么须指向一个页表页. 如果 `freewalk()` 发现了指向叶子页的有效条目, 那么内核就会 panic.

这样, `proc_freetable()` 的第三行调用
```c
  uvmfree(pagetable, sz);
```
把进程自身占用的物理内存 (大小为 `sz` 字节) 以及进程的各级页表占用的物理内存全部回收.

## 进程虚拟内存工具函数

在 `kernel/vm.c` 中, 还定义了许多用于操纵进程虚拟内存的工具函数 (utility functions). 内核会调用这些函数实现一些常用的功能. 这些工具函数的源代码比较简单, 而且挺无聊的, 所以我们只列出它们的名称, 并描述它们的功能.

| 名称 | 源代码位置 | 功能 |
|:---:|:---:|:---|
|`uvmfirst()` | `vm.c[207:221]`| 新建一个物理页, 将内核地址空间中 `[src, src + sz)` 的范围拷贝到该物理页中 (`sz` 大小不超过一页), 然后把用户进程虚拟地址空间中地址最低的一页 (即零地址出发的一页) 映射到该物理页. |
|`uvmalloc()`| `vm.c[223:249]` | 分配新的物理页并新建地址映射, 使用户进程占用的虚拟地址范围由 `[0, oldsz)` 扩大到 `[0, newsz)`. 新建的每一个虚拟页都有权限位 `PTE_R \| PTE_U \| xperm`, 其中 `xperm` 由参数指定.|
|`uvmdealloc()` | `vm.c[251:267]` | 回收一些物理页, 使用户进程占用的虚拟地址范围由 `[0, oldsz)` 缩小到 `[0, newsz)`.|
|`uvmcopy()` | `vm.c[299:333]` | 将旧进程的页表 `old` 原样复制到新进程的页表 `new`. 页表 `new` 中的第 `i` 个虚拟页对应的物理页, 是页表 `old` 中的第 `i` 个虚拟页所对应物理页的副本. 这个函数的语义与 Unix 系统中的 `fork()` 相似 (无 copy-on-write).|
| `uvmclear()` |`vm.c[335:346]` | 将虚拟地址 `va` 所在的虚拟页设为用户不可访问 (`U=0`). |
| `copyout()` | `vm.c[348:371]` | 将内核地址空间中 `[src, src + len)` 的范围拷贝到进程虚拟地址空间中 `dstva` 出发的内存区域. 要求虚拟地址范围 `[dstva, dstva + len)` 经过的每一个虚拟页都是有效的 (`V=1`)、用户可访问的 (`U=1`). |
| `copyin()` | `vm.c[373:396]` | 将进程虚拟地址空间中 `[srcva, srcva + len)` 的范围拷贝到内核地址空间中 `dst` 出发的区域. 要求虚拟地址范围 `[srcva, srcva + len)` 经过的每一个虚拟页都是有效的 (`V=1`)、用户可访问的 (`U=1`). |
| `copyinstr()` | `vm.c[398:439]`| 同 `copyin()`, 只不过在拷贝过程中一旦检测到零字节, 就停止拷贝. 这适用于拷贝字符串. |

至此, 我们成功为 **xv6 内核** ([上一章](ch7.md)) 和 **用户进程** (本章) 完成了虚拟内存的配置.

> 本章中介绍的所有函数, 包括 `uvmcreate()`, `uvmunmap()`, `uvmfree()` 以及上面所有工具函数, 都是关于多处理器并发访问安全的. 关键原因在于, 用户进程使用的全部物理页 (包括页表页) 都是用 `kalloc()` 分配出来的, 从而 (由 `kalloc()` 的实现保证) 这些页是 **私有地** 属于单一处理器核心的.
>
> 这里的情况与内核页表不同: 内核页表是被多个处理器 **共享** 的.
