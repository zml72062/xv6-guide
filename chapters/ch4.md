# 4. 互斥锁

## 多处理器并行问题

回忆一下, 迄今为止 xv6 系统已经完成了下述工作:
* 为每个 CPU 核心分配栈空间, 然后跳转到 `start()`
* 在 M 模式下, 初始化一系列寄存器, 并安装时钟中断处理程序
* 特权级别从 M 模式切换到 S 模式, 然后跳转到 `main()`

接下来我们就要阅读 `main()` 函数的代码. 不过, 在本章中我们暂时不具体展开 `main()` 函数的内容. 这是因为现在有必要指出 xv6 操作系统运行的整个过程中都存在的一个问题: **多处理器并行** 问题. 

具体来讲, 由于我们使用的机器有 **多个 CPU 核心**, 而物理内存是被所有这些核心 **共用** 的, 因此我们必须保证不同核心对内存的读写操作不会发生彼此冲突的情况.

到现在为止, 我们并没有严肃地考虑过多处理器并行的问题. 事实上, 如果仔细观察一下我们前几章读过的代码, 会发现所有的指令干的事情不外乎以下三种情况:

* 操作寄存器
    + 例如: 使用 `mv`/`csrr`/`csrw` 把一个寄存器的值复制到另一个寄存器, 或者使用 `mret` 指令
* 读一块 **所有 CPU 都不会修改** 的内存
    + 例如: 读取内存位置 `CLINT_MTIME` 处的 8 字节整数
* 读写一块 **仅当前 CPU 可访问** 的内存
    + 例 1: 使用栈. C 语言中的临时变量、函数嵌套调用都需要栈. 在 `entry.S` 中, 给每个核心都分配了一段 **独立的** 4096 字节内存作为栈. 每个核心都不会访问其他核心的栈 (只要不发生溢出).
    + 例 2: 访问内存位置 `CLINT_MTIMECMP(i)`. 从 `memlayout.h` 中可以看到, 每个核心 `i` 都有 **独立的** 8 字节内存 (以 `CLINT_MTIMECMP(i)` 为首地址).
    + 例 3: 使用数组 `timer_scratch`. 在 `start.c` 中定义的 `timer_scratch` 数组, 给每个核心都分配了 **独立的** 40 字节. 在处理时钟中断时, 每个核心都只会使用属于自己的 40 字节内存.

我们看到: 这三种情况都不会导致多个核心之间的相互干扰. (注意寄存器不是共享的, 每个核心都有一套自己的寄存器.) 这就是为什么我们之前无需认真考虑并行问题.

但是, 我们不可能保证多个核心永远不会发生冲突. 一个典型例子是 **物理内存分配**. 在 xv6 中, 内核把物理内存分割成一系列大小为 4096 字节的块, 并使用一个链表连接所有空闲的块. 当调用 `kalloc()` (`kernel/kalloc.c[65:82]`) 时, 链表头部的块被移出链表以供使用; 调用 `kfree()` (`kernel/kalloc.c[42:63]`) 时, 不再使用的块被重新附加到链表头部.

现在设想这样的情景: 有两个 CPU 核心同时调用了 `kalloc()` 申请获得一块物理内存. 这种场景下, 每个核心都要依次做下面两件事情:
* 把链表的头指针 `head` 存放到某个寄存器中
* 将 `head` 更新为链表头部块下一个块的起始地址

> 需要注意, `head` 本身也必须放在内存里. 可不可以把 `head` 放在一个专门的寄存器里呢? 看似可以, 但是这样会使得每个 CPU 都维护了一份 `head` 的副本. (回忆一下, 每个核心都有一套独立的寄存器.) 我们仍然需要操心如何同步这些副本. 所以, xv6 选择将 `head` 放在内存中.

上述情况会导致一个潜在的危险: 我们无法预知两个核心执行指令的次序, 因而完全有可能, 两个核心都先把 `head` 从内存加载到了它们各自的某个寄存器中, 然后再各自将 `head` 更新为头部块下一个块 (对于它们来说是同一个块!) 的地址. 此时, 它们都觉得自己已经申请到了原先 (更新 `head` 前) 链表中的第一个块, 并开始访问它. 很明显, 这将会带来严重的访问冲突. 

在上面描绘的情景中, 有两个数据是被多个核心 **共享** 的:
* 链表中所有的空闲块都是共享的 (特别地, 对于上面的例子, 链表中的第一个块是共享的)
* 头指针 `head` 本身也是共享的

在多核并行场景下, 我们必须对 **共享的数据结构** 采取某种保护措施.

## 互斥锁

解决这个问题的一种办法是: 当我们需要修改被多个核心共享的数据结构时, 采用某种机制保证 **同时只能有一个核心在访问这个数据结构**. 在这个核心宣布修改完成之前, 不允许其他的核 **读或写** 这个数据结构. 这种方法的具体实现即为 **互斥锁**.

在 xv6 系统中, 大量使用互斥锁保护内核中的共享数据结构. 在 `kernel/spinlock.h` 中, 定义了互斥锁的数据结构:
```c
// Mutual exclusion lock.
struct spinlock {
  uint locked;       // Is the lock held?

  // For debugging:
  char *name;        // Name of lock.
  struct cpu *cpu;   // The cpu holding the lock.
};
```
其中最重要的字段就是 `locked`. 具体来讲, 如果某个 CPU 核心想要访问共享数据结构 A, 它首先要检查 A 的互斥锁中 `locked` 字段是否为 1.
* 如果 `locked == 0`, 那么说明该核心可以访问 A. 在访问之前, 它须把 `locked` 字段设为 1.
* 如果 `locked == 1`, 那么说明有别的核心在访问 A, 该核心不能访问 A. 它就反复地检查 `locked` 的值, 直到发现 `locked` 变为 0, 它才能访问 A (同时把 `locked` 设为 1).

此外, 为了调试方便, 在 `spinlock` 结构中还记录了一个名字 `name` (指明该互斥锁的功能) 和一个指针 `cpu` (指明当前占有互斥锁的 CPU 核心的信息).

> 指针 `cpu` 指向一个 `cpu` 结构体. 该结构体定义在 `kernel/proc.h[21:27]` 中:
> ```c
> // Per-CPU state.
> struct cpu {
>   struct proc *proc;          // The process running on this cpu, or null.
>   struct context context;     // swtch() here to enter scheduler().
>   int noff;                   // Depth of push_off() nesting.
>   int intena;                 // Were interrupts enabled before push_off()?
> };
> ```
> 此外, 在 `kernel/proc.c[9]` 中维护了一个数组
> ```c
> struct cpu cpus[NCPU];
> ```
> 用于将 CPU 核心号映射到相应的 `cpu` 结构体. 使用函数 `mycpu()` 可以方便地获取当前核心的 `cpu` 结构体: (`kernel/proc.c[61:79]`)
>
> ```c
> int cpuid()
> {
>   int id = r_tp();    // read thread pointer "tp“
>   return id;
> }
> 
> // Return this CPU's cpu struct.
> // Interrupts must be disabled.
> struct cpu* mycpu(void)
> {
>   int id = cpuid();
>   struct cpu *c = &cpus[id];
>   return c;
> }
> ```
>
> 在调用 `mycpu()` 时要求 **关闭中断**. 这是因为中断处理程序可能会执行上下文切换, 使 `cpu` 结构体中的 `proc` 和 `context` 字段发生变化.
>

互斥锁的使用分为两步: **加锁** 和 **解锁**. 
* **加锁:** 循环地检查 `locked` 的值, 一旦发现 `locked == 0` (其他核心访问完成), 将 `locked` 设为 1 (宣布开始访问).
* **解锁:** 将 `locked` 设为 0 (宣布访问完成).

> 加锁和解锁操作通常分别被叫做 `P()` 和 `V()` 操作.

## 互斥锁: 实现

在 `kernel/spinlock.c` 中, 提供了实现互斥锁的代码.

### 初始化

参见 `kernel/spinlock.c[11:17]`:
```c
void initlock(struct spinlock *lk, char *name)
{
  lk->name = name;
  lk->locked = 0;
  lk->cpu = 0;
}
```
初始时刻, 互斥锁 `lk` 处于不上锁状态 (`lk->locked == 0`), 其 `cpu` 指针为空.

### 加锁

参见 `kernel/spinlock.c[19:43]`:
```c
// Acquire the lock.
// Loops (spins) until the lock is acquired.
void acquire(struct spinlock *lk)
{
  push_off(); // disable interrupts to avoid deadlock.
  if(holding(lk))
    panic("acquire");

  // On RISC-V, sync_lock_test_and_set turns into an atomic swap:
  //   a5 = 1
  //   s1 = &lk->locked
  //   amoswap.w.aq a5, a5, (s1)
  while(__sync_lock_test_and_set(&lk->locked, 1) != 0)
    ;

  // Tell the C compiler and the processor to not move loads or stores
  // past this point, to ensure that the critical section's memory
  // references happen strictly after the lock is acquired.
  // On RISC-V, this emits a fence instruction.
  __sync_synchronize();

  // Record info about lock acquisition for holding() and debugging.
  lk->cpu = mycpu();
}
```
可以看到, `acquire()` 函数分为四步.

#### **关中断**

加锁的第一步需要调用 `push_off()` **关中断**. 

> 为什么要这么做呢? 这是因为有些中断处理程序也会访问共享数据结构, 从而也需要使用互斥锁. 现在设想某个 CPU 核心 **没关中断** 就持有了数据结构 A 的互斥锁, 这时候来了一个中断 i, 然后硬件就自动跳转到中断 i 对应的中断向量去执行. 
>
> 问题在于: 假如中断 i 的处理程序也需要访问数据结构 A, 那么它必须等待 A 的互斥锁被解除才能继续; 然而, 想要解锁 A, 就得回到中断 i 到来之前的正常指令流去执行, 从而首先要等中断 i 的处理程序执行完. 我们发现这时候就出现了 **死锁**.
>
> 为避免这种问题发生, 我们必须保证: **当持有互斥锁 A 时, 关闭那些同样需要持有锁 A 才能处理的中断.** 在 xv6 中, 索性做的更简单粗暴一些: 在试图获取任何互斥锁之前, 都关闭所有中断.
>

函数 `push_off()` 的实现如下: (`kernel/spinlock.c[88:97]`)
```c
void push_off(void)
{
  int old = intr_get();

  intr_off();
  if(mycpu()->noff == 0)
    mycpu()->intena = old;
  mycpu()->noff += 1;
}
```
> 在 `kernel/riscv.h` 中提供了三个实用的函数:
> * `intr_get()`: 查看 `sstatus` 中全局中断开关 SIE 是 (1) / 否 (0) 开启
> * `intr_on()`: 打开 `sstatus` 中的全局中断开关 SIE
> * `intr_off()`: 关闭 `sstatus` 中的全局中断开关 SIE
>

为什么不直接用 `intr_off()` 关中断, 而是用这里的 `push_off()` 呢? 这是由于一个 CPU 可以同时持有多个互斥锁, 我们需要保证: **只要当前 CPU 还持有锁, 就应保持中断关闭.** 为此, 我们需要为每个 CPU 维护其持有锁的数目. 

在每个核心对应的 `cpu` 结构体中, 我们用 `noff` 字段来表示它持有锁的个数, 用 `intena` 字段表示这个 CPU 第一次获取锁之前, 中断是否开启. 每调用一次 `push_off()`, `noff` 会加一.

> 以上, 我们通过将 `sstatus` 中的 SIE 置 0 来关闭中断. 但是, 这样无法关闭时钟中断: 无论 SIE 为何值, 时钟中断都会使处理器从 S 模式 trap 到 M 模式. 这似乎违反了我们上面的原则 (持有锁之前关闭全部中断). 不过, xv6 的时钟中断处理程序保证了这是没问题的, 因为
> * M 模式的时钟中断处理程序 `timervec()` **本身不试图获取锁**, 仅仅简单地重置闹钟并向 S 模式转发一个 SSI 中断.
> * SSI 中断处理程序 **会试图获取锁**, 但可以通过设置 `SIE=0` 暂时禁用.
>
> 这样, 死锁仍然不会发生.
>

#### **获取锁**

现在我们可以尝试获取锁:
```c
  while(__sync_lock_test_and_set(&lk->locked, 1) != 0)
    ;
```
`__sync_lock_test_and_set()` 函数是 GCC 提供的原子化访存操作. 在 RISC-V 处理器上, 这个函数经过编译之后会含有一条指令:
```asm
    amoswap.w r, r, (a)
```
其中我们假定寄存器 `r` 持有立即数 1, 而寄存器 `a` 持有 `&lk->locked`. 这条指令会 **原子地** 交换寄存器 `r` 的值与 `a` 所指向的内存地址的值. `__sync_lock_test_and_set()` 的返回值就是交换之后 `r` 的值. 这样, 如果这个函数返回 1, 那么就说明交换前 `lk->locked == 1`, 即别的核心还在占用锁, 所以要继续循环; 如果函数返回 0, 那么就说明已成功获得锁, 可以退出循环继续执行了.

注意这里 `__sync_lock_test_and_set()` 函数的原子性对于保证互斥性是很重要的. 通常来说, 交换一个寄存器 `r` 和一个内存地址 `a` 的值需要两次访存: 首先读出内存地址 `a` 的值, 然后把新值写入内存. 也就是说, 
```asm
    mv temp, (a)
    mv (a), r
```
现在假设 `lk->locked == 0`, 且有两个核心都试图交换寄存器 `r` 与内存地址 `&lk->locked` 处的值. 假如我们用两条 `mv` 指令来实现交换操作的话, 那么有可能两个核心都先完成了 `mv temp, (a)` 这条指令, 读出了内存中 `lk->locked` 的值 (0); 然后它们再各自把数值 1 写入内存. 这种情况下, 两个核心都会认为自己获得了锁, 从而互斥性被破坏了. 为避免此问题, 我们必须使用原子的 `amoswap.w` 指令来实现交换操作.

#### **访存指令同步**

当上一步的循环退出时, 意味着当前 CPU 已经成功取到了锁. 接下来一条指令是
```c
  __sync_synchronize();
```
这是一条 **fence instruction**. 它要求编译器和 CPU 都 **不要把 fence 一侧的访存指令移到另一侧**. 也就是说, `__sync_synchronize()` 保证: 该指令之后的访存操作, 都必须等到该指令之前的所有访存操作都完成之后, 才会执行. 从而它承诺: 对于共享数据结构的内存访问严格地发生在 CPU 获得锁之后.

> 以下内联汇编
> ```c
> asm volatile("" ::: "memory");
> ```
> 可以起到 compiler-level fence instruction 的作用. 也就是说, 它可以阻止编译器交换该指令前后的访存指令. (但是如果处理器支持乱序执行, 则这条指令无法在 CPU-level 保证访存顺序.) 
>
> 可以看到: 这条内联汇编的主体是空的, 也就是它什么都不执行. 但是, 
> * `volatile` 限定符指示编译器, 这条指令有副作用, 不允许删除这条指令
> * 第三个冒号后面的 `"memory"` 指示编译器, 这条指令修改了内存中的内容, 从而不允许把一条访存指令同这条指令交换次序
>

#### **维护数据结构**

最后一步, 我们把当前 CPU 的信息填入互斥锁 `lk` 的数据结构:
```c
  lk->cpu = mycpu();
```

### 解锁

参见 `kernel/spinlock.c[45:72]`:
```c
// Release the lock.
void
release(struct spinlock *lk)
{
  if(!holding(lk))
    panic("release");

  lk->cpu = 0;

  // Tell the C compiler and the CPU to not move loads or stores
  // past this point, to ensure that all the stores in the critical
  // section are visible to other CPUs before the lock is released,
  // and that loads in the critical section occur strictly before
  // the lock is released.
  // On RISC-V, this emits a fence instruction.
  __sync_synchronize();

  // Release the lock, equivalent to lk->locked = 0.
  // This code doesn't use a C assignment, since the C standard
  // implies that an assignment might be implemented with
  // multiple store instructions.
  // On RISC-V, sync_lock_release turns into an atomic swap:
  //   s1 = &lk->locked
  //   amoswap.w zero, zero, (s1)
  __sync_lock_release(&lk->locked);

  pop_off();
}
```
解锁和加锁实际上是互逆操作, 它也分为四步.

#### **重置数据结构**

把 `lk` 中的 CPU 信息清零:
```c
  lk->cpu = 0;
```

#### **访存指令同步**

我们希望对共享数据结构的内存访问严格地发生在释放锁之前, 因此这里也需要一条 fence instruction:
```c
  __sync_synchronize();
```

#### **释放锁**

代码中的
```c
  __sync_lock_release(&lk->locked);
```
经过编译后会得到指令
```
    amoswap.w zero, zero, (a)
```
其中我们假定寄存器 `a` 持有地址 `&lk->locked`. 这条指令原子地交换寄存器 `zero` 和内存中的值 `lk->locked`. 由于 `zero` 是 RISC-V 中一个恒为零的寄存器, 上述指令相当于
```c
  lk->locked = 0;
```

#### **恢复中断状态**

最后, 我们调用 `pop_off()` 恢复中断状态. 从名字上容易看出 `pop_off()` 是 `push_off()` 的逆过程. 参见 `kernel/spinlock.c[99:110]`:
```c
void
pop_off(void)
{
  struct cpu *c = mycpu();
  if(intr_get())
    panic("pop_off - interruptible");
  if(c->noff < 1)
    panic("pop_off");
  c->noff -= 1;
  if(c->noff == 0 && c->intena)
    intr_on();
}
```
它把当前核心的 `cpu` 结构体中的 `noff` 减一; 然后, 如果发现当前 CPU 已经不持有任何锁 (`c->noff == 0`) 并且在第一次持有锁之前, 中断处于开启状态 (`c->intena == 1`), 那么就打开中断.

至此, 我们已经实现了互斥锁. 在接下来将会看到它的大量应用.

