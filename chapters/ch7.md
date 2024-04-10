# 7. 内核态虚拟内存

从本章开始, 我们的主题是 **虚拟内存**. 简单来讲, 虚拟内存是一种对物理内存进行 **抽象** 的机制. 一旦启用这种机制后, 当 CPU 执行一条访问地址 X 的访存指令时, 并不会直接访问物理地址为 X 的存储单元, 而是先通过一种特殊的硬件 (**内存管理单元**, 或 **MMU**) 将地址 X 翻译为另一个地址 Y, 然后访问物理地址为 Y 的存储单元. 在这里, 地址 X 就叫做 **虚拟地址**.

虚拟内存机制为程序员提供了一个简洁的编程模型: 从应用程序的角度看来, 程序中的访存指令是在访问一个虚构的 **虚拟地址空间**, 而非真正的物理内存. 此外, 操作系统对内存的管理和保护也依赖于虚拟内存机制.

## RISC-V 地址翻译硬件

虚拟内存机制依赖于硬件提供的 **地址翻译** 功能. 所谓地址翻译, 是指把一个虚拟地址 X 翻译成物理地址 Y. 在 RISC-V 特权指令集手册 ([https://riscv.org/wp-content/uploads/2017/05/riscv-privileged-v1.10.pdf](https://riscv.org/wp-content/uploads/2017/05/riscv-privileged-v1.10.pdf), 4.3 至 4.5 节) 中, 介绍了 RISC-V 支持的三种地址翻译模式: Sv32, Sv39 和 Sv48.

xv6 操作系统使用 RISC-V 的 **Sv39** 地址翻译模式作为虚拟内存机制的硬件支持. 在接下来的几段中, 我们将介绍 Sv39 模式下的地址翻译流程. 

Sv39 模式下, 虚拟地址的长度为 39 位, 物理地址的长度为 56 位. 这意味着一个合法的 64 位虚拟地址中的高 25 位都必须为 0. 为了支持地址翻译, 虚拟地址空间和物理地址空间都被划分为一系列 4096 ($2^{12}$) 字节的 **页**. 相应地, 虚拟地址和物理地址都被划分为 **页号** (page number, PN) 和 **页偏移** (page offset), 如下图所示:

![Sv39 模式下的虚拟地址和物理地址](/chapters/ch7-figs/address.png "Sv39 模式下的虚拟地址和物理地址")

虚拟地址和物理地址的低 12 位代表页偏移, 其余位代表页号. 页号 (VPN 或 PPN) 又进一步分成三个部分.

### 地址翻译过程

**页表** 记录了虚拟页号 (VPN) 到物理页号 (PPN) 的映射. 粗略地来说, 地址翻译过程就是:
* 给定一个虚拟地址 X, 取出它的 VPN
* 在页表中, 找到序号为 VPN 的条目, 从中读出 PPN
* 把虚拟地址 X 的 VPN 改成 PPN, 即得物理地址 Y

页表本身也需要存放在内存中. 在 RISC-V 中, 使用 `satp` 寄存器保存页表的起始物理地址. 在 32 位和 64 位机器上, `satp` 寄存器的结构分别如下面两张图所示:

![RV32 下的 satp 寄存器](/chapters/ch7-figs/satp32.png "RV32 下的 satp 寄存器")

![RV64 下的 satp 寄存器](/chapters/ch7-figs/satp64.png "RV64 下的 satp 寄存器")

我们只需要关心第二张图, 因为只有 64 位 RISC-V 机器才支持 Sv39 翻译模式. 在 64 位机器上, `satp` 的低 44 位代表 **页表所在的物理页号 (PPN)**. 将这个页号乘以一页的大小 (4096 字节), 就得到 **页表的起始物理地址**.

此外, `satp` 的头 4 位表示地址翻译模式. 当这 4 位的值为 8 时, 表示 Sv39 模式.

![RV64 支持的地址翻译模式](/chapters/ch7-figs/modes.png "RV64 支持的地址翻译模式")

当然, 实际的地址翻译过程比这要复杂一些. 主要原因是: Sv39 模式下 VPN 一共有 27 位; 如果按照上面的方式来做地址翻译, 那么我们需要 $2^{27}$ 个页表项. 为了减少页表项的内存占用, 实际的 Sv39 模式使用 **多级页表** (共有 3 级) 来帮助地址翻译:

* 给定一个虚拟地址 X, 取出它的 `VPN[2]`
* 在 **一级页表** 中 (物理基址为 `satp[43:0] * 4096` 字节), 找到序号为 `VPN[2]` 的条目, 读出该条目的 PPN, 设为 `p0`
* 取出地址 X 的 `VPN[1]`, 然后在 **二级页表** 中 (物理基址为 `p0 * 4096` 字节), 找到序号为 `VPN[1]` 的条目, 读出该条目的 PPN, 设为 `p1`
* 取出地址 X 的 `VPN[0]`, 然后在 **三级页表** 中 (物理基址为 `p1 * 4096` 字节), 找到序号为 `VPN[0]` 的条目, 读出该条目的 PPN, 设为 `p2`
* 把虚拟地址 X 的 VPN 改为 `p2`, 即得物理地址 Y

> 以上地址翻译过程中, 只有 `p2` 是虚拟地址 X 真正对应的物理页号. 物理页号 `p0` 或 `p1` 表达的含义是: 在 `p0` 或 `p1` 所指示的物理页中, 存放着下一级 (二级或三级) 页表. 通常把对应于某个虚拟页的物理页 (例如 PPN 为 `p2` 的物理页) 叫做 **叶子页**, 而把对应于某个二级/三级页表的物理页 (例如 PPN 为 `p0` 或 `p1` 的物理页) 叫做 **页表页**.

### 页表的结构

在 Sv39 模式下, 无论是一级、二级还是三级页表, 都是由一系列 8 字节的表项组成的. 每个表项的内容如下图所示:

![Sv39 模式下的页表项](/chapters/ch7-figs/pte.png "Sv39 模式下的页表项")

> 因为 `VPN[0]`, `VPN[1]` 和 `VPN[2]` 都是 9 位长的, 所以每一级页表都有 $2^9$ 个表项, 其大小正好是 $2^{12}$ 字节, 即一个页的大小.

页表项中除了地址翻译过程所需要的 PPN (长度 44 位) 之外, 还有一系列 **权限检查位**:
* `V` 位表示该页表项是 (1) / 否 (0) 有效. 如果硬件在地址翻译过程中查询到了一条无效的页表项, 则会终止翻译, 并触发一个缺页异常.
* `R`, `W`, `X` 位分别表示该页表项指向的物理页是 (1) / 否 (0) 是 **可读的**, **可写的** 或 **可执行的**. 如果该页表项的 PPN 对应于一个 **叶子页**, 那么 `R`, `W`, `X` 三位至少有一位是 1. 如果这三位都为 0, 那么表示该页表项的 PPN 是某个 **页表页** 的物理页号. 具体如下图所示:

![R, W, X 位的含义](/chapters/ch7-figs/rwx.png "R, W, X 位的含义")

* `U` 位表示该页表项指向的物理页是 (1) / 否 (0) 可在用户模式 (U 模式) 访问.
* `A` 位又叫 **访问位** (access bit). 它表示: 自 `A` 位上一次被清零以来, 该页表项对应的物理页是 (1) / 否 (0) 被访问过. (访问包括读、写或执行.)
* `D` 位又叫 **脏位** (dirty bit). 它表示: 自 `D` 位上一次被清零以来, 该页表项对应的物理页是 (1) / 否 (0) 被写过.

> `U`, `A`, `D` 位都只对 **叶子页** 生效. 页表页的 `U`, `A`, `D` 位必须总是清零.

有了这些权限位之后, 地址翻译流程还需要增加对这些权限位的检查:

* 访问一个叶子页之前, 必须根据处理器特权级别、访存指令的类型, 检查此次访问是否符合指向该页的页表项中 `R`, `W`, `X`, `U` 位的限制.
* 成功访问一个 `A=0` 的叶子页后, 须令指向该页的页表项中 `A` 位为 1.
* 成功写一个 `D=0` 的叶子页后, 须令指向该页的页表项中 `D` 位为 1.

> 硬件保证: 同一次地址翻译过程中, 读取叶子页页表项, 与写页表项中的 `A`, `D` 位, 这两步操作是原子的. 也就是说, 不会在两步之间插入不属于本次地址翻译过程的读或写.

## 内核态虚拟内存的初始化

现在我们来看 xv6 内核如何 **为自身** 启用虚拟内存机制. 这部分代码位于 `kernel/main.c[20:21]`:
```c
    kvminit();       // create kernel page table
    kvminithart();   // turn on paging
```
一旦完成这部分工作, 我们就可以使用 **虚拟地址** (而非物理地址) 来访问内核中的数据结构.

从上一节可以看到, 地址翻译过程并不需要操作系统的过多参与——硬件会自动地完成这一切. 操作系统要做的其实只有三件事:
* 在内存中建立并维护页表
* 设置 `satp` 寄存器以控制地址翻译
* 当缺页中断发生时, 做出处理

在这三条当中, 第三条涉及中断处理, 因而我们在此不做赘述. (第 10~11 章会更仔细地介绍中断处理这个主题.) 本章的剩余部分将介绍 xv6 内核如何完成前两条的工作.

### xv6 内核态虚拟地址空间

xv6 内核通过 `kvminit()` 函数为内核建立页表. 这个函数定义在 `kernel/vm.c[52:57]`:
```c
// Initialize the one kernel_pagetable
void kvminit(void)
{
  kernel_pagetable = kvmmake();
}
```
它调用 `kvmmake()` 生成内核页表, 然后将一级页表的物理基址保存在全局变量 `kernel_pagetable` 中. 全局变量 `kernel_pagetable` 定义在 `kernel/vm.c[12]`:
```c
pagetable_t kernel_pagetable;
```
它的数据类型定义在 `kernel/riscv.h[330:331]`:
```c
typedef uint64 pte_t;
typedef uint64 *pagetable_t;
```
在这里, 类型 `pte_t` 代表一个 8 字节长的页表项, `pagetable_t` 代表一个页表 (页表项的数组).

> 为什么不需要保护共享数据 `kernel_pagetable` 呢? 这是因为 `kvminit()` 函数仅被编号为 0 的 CPU 独自调用一次. 除了 `kvminit()` 初始化了 `kernel_pagetable` 的值之外, 没有任何函数会修改 `kernel_pagetable`. 即 `kernel_pagetable` 实际上是只读常量.

在开始研究建立页表的细节之前, 描绘一些 big picture 是有帮助的. 从逻辑上讲, "建立页表" 的过程, 实质上是建立了一个 **虚拟地址空间**, 然后按照地址翻译的规则建立从虚拟地址空间到物理地址空间的 **映射**. 怎么样建立虚拟地址空间, 从原则上来说是任意的; 此外, 虚拟地址空间可能远比物理地址空间大, 因此我们也允许多个虚拟地址对应同一个物理地址. 

xv6 内核采用一种非常简单的方式来构建虚拟地址空间:
* 对于虚拟地址范围 `[0, 0x88000000)`, 采用 **直接映射**, 也就是令 **物理地址 = 虚拟地址**.
* 将虚拟地址空间最顶部的一页 (称为 **trampoline 页**) 映射到物理内存中内核代码段地址最高的一页. 这一页中包含了从用户态陷入内核态, 以及从内核态返回用户态时必须执行的代码. (之所以做这样的映射, 是为了方便中断处理.)
* 在 trampoline 页下方, 为每个进程分配一个虚拟页作为 **内核栈**. 这些内核栈对应的物理页通过 `kalloc()` 从空闲物理内存中分配出来.

下面这张图取自 xv6 book. 它很好地展示了 xv6 内核态虚拟内存的布局, 以及虚拟地址空间到物理地址空间的映射关系.

![xv6 内核态虚拟地址空间布局](/chapters/ch7-figs/addrspace.png "xv6 内核态虚拟地址空间布局")

### 页表的建立

现在我们来阅读 xv6 内核建立页表的代码细节. 一个核心的函数是 `walk()` (定义在 `kernel/vm.c[85:103]`):
```c
pte_t *walk(pagetable_t pagetable, uint64 va, int alloc)
{
  if(va >= MAXVA) // MAXVA = 1L << 38;
    panic("walk");

  for(int level = 2; level > 0; level--) {
    pte_t *pte = &pagetable[PX(level, va)];
    if(*pte & PTE_V) {
      pagetable = (pagetable_t)PTE2PA(*pte);
    } else {
      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0)
        return 0;
      memset(pagetable, 0, PGSIZE);
      *pte = PA2PTE(pagetable) | PTE_V;
    }
  }
  return &pagetable[PX(0, va)];
}
```
`walk()` 函数的功能是: 以参数 `pagetable` 为一级页表基址, 模拟硬件查询虚拟地址 `va` 对应的物理地址, 并且 (如果设置参数 `alloc` 为 1 的话) 初始化翻译 `va` 的过程中所需的页表项. 具体来说:
* 首先, 检查 `va` 是否超出了 xv6 虚拟地址空间的最大范围 (`MAXVA = 1L << 38`), 如果超出则 panic.
> 虽然 Sv39 模式支持最长 39 位的虚拟地址, 但 xv6 只使用其中的低 38 位.
* 查询一级、二级页表 (分别对应 `level == 2` 和 `level == 1`):
  + 首先, 取出 `va` 中的 `VPN[level]`:
    ```c
    PX(level, va) = (va >> (12+9*level)) & 0x1FF;
    ```
  + 查询以 `pagetable` 为基址的页表中, 序号为 `VPN[level]` 的条目, 即
    ```c
    pte_t *pte = &pagetable[PX(level, va)];
    ```
  + 如果页表条目有效 (`V` 位为 1, 即 `*pte & PTE_V != 0`), 就取出 `pte` 中的 PPN 所对应的物理页基址:
    ```c
    // PTE2PA(x) = (x >> 10) << 12;
    pagetable = (pagetable_t)PTE2PA(*pte);
    ```
    这个基址也就是下一级页表的物理基址. 因此将它赋值给 `pagetable`, 并进行下一级页表的查询.
  + 如果页表条目无效 (`*pte & PTE_V == 0`), 意味着下一级页表还不存在于物理内存中. 此时我们查看参数 `alloc` 的值. 如果 `alloc != 0`, 意味着我们希望新建下一级页表, 于是就做下面几步:
    * 尝试用 `kalloc()` 分配一个物理页, 并把它的基址赋值给 `pagetable`, 作为下一级页表的页表页.
      ```c
      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0)
        return 0;
      ```
    > 这里 `pde_t` 和 `pte_t` 一样, 都是 `uint64` 的别名.
    * 将新的页表页初始化为全零.
      ```c
      memset(pagetable, 0, PGSIZE);
      ```
    * 把新的页表页 PPN 填入当前 `pte`, 并设置 `V` 位为 1, 其他所有权限位 (`R`, `W`, `X`, `U`, `A`, `D`) 全为 0.
      ```c
      // PA2PTE(x) = (x >> 12) << 10;
      *pte = PA2PTE(pagetable) | PTE_V;
      ```
    如果 `alloc == 0`, 函数就返回空指针.
* 上一步结束时, `pagetable` 中存储了最后一级页表 (即三级页表) 的基址. `walk()` 函数返回三级页表中序号为 `VPN[0]` 的条目 (`&pagetable[PX(0, va)]`). 这个条目应当记录 `va` 对应的物理页号.

有了 `walk()` 函数, 我们就可以为任意的虚拟地址 **创建其翻译过程所需的全部页表项**. 但是, 我们还未填写最后一级页表中相应于该虚拟地址的 PPN 和权限位 (`R`, `W`, `X`, `U`). 这件事情由 `mappages()` 函数来完成: (`kernel/vm.c[142:165]`)
```c
int mappages(pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm)
{
  uint64 a, last;
  pte_t *pte;

  if(size == 0)
    panic("mappages: size");
  
  a = PGROUNDDOWN(va);
  last = PGROUNDDOWN(va + size - 1);
  for(;;){
    if((pte = walk(pagetable, a, 1)) == 0)
      return -1;
    if(*pte & PTE_V)
      panic("mappages: remap");
    *pte = PA2PTE(pa) | perm | PTE_V;
    if(a == last)
      break;
    a += PGSIZE;
    pa += PGSIZE;
  }
  return 0;
}
```
`mappages()` 将连续的虚拟地址范围 `[va, va + size)` 映射到物理地址范围 `[pa, pa + size)`. 对于地址范围 `[va, va + size)` 所经过的每一个虚拟页, `mappages()` 函数
* 调用 `walk()` 初始化中间所需的全部页表项
* 将此虚拟页对应的 PPN、权限位填入相应的三级页表项 `pte`, 并设置 `V` 位为 1
```c
// PA2PTE(pa) ignores physical page offset in "pa"
// making "pa" effectively page-aligned
*pte = PA2PTE(pa) | perm | PTE_V;
```

如果操作全部成功, `mappages()` 会返回 0. xv6 中还定义了一个很无聊的函数 `kvmmap()` 来包装 `mappages()`: (`kernel/vm.c[131:136]`)
```c
void kvmmap(pagetable_t kpgtbl, uint64 va, uint64 pa, uint64 sz, int perm)
{
  if(mappages(kpgtbl, va, sz, pa, perm) != 0)
    panic("kvmmap");
}
```

### 虚拟地址空间的建立

有了 `mappages()` (或 `kvmmap()`) 函数, 我们已掌握足够的工具来实现内核态虚拟地址空间到物理内存的映射 (也就是如图 "xv6 内核态虚拟地址空间布局" 所示的那样). 真正做这件事情的是 `kvmmake()` 函数: (`kernel/vm.c[19:50]`)
```c
pagetable_t
kvmmake(void)
{
  pagetable_t kpgtbl;

  kpgtbl = (pagetable_t) kalloc();
  memset(kpgtbl, 0, PGSIZE);

  // uart registers
  kvmmap(kpgtbl, UART0, UART0, PGSIZE, PTE_R | PTE_W);

  // virtio mmio disk interface
  kvmmap(kpgtbl, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

  // PLIC
  kvmmap(kpgtbl, PLIC, PLIC, 0x400000, PTE_R | PTE_W);

  // map kernel text executable and read-only.
  kvmmap(kpgtbl, KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);

  // map kernel data and the physical RAM we'll make use of.
  kvmmap(kpgtbl, (uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);

  // map the trampoline for trap entry/exit to
  // the highest virtual address in the kernel.
  kvmmap(kpgtbl, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);

  // allocate and map a kernel stack for each process.
  proc_mapstacks(kpgtbl);
  
  return kpgtbl;
}
```
`kvmmake()` 函数首先调用 `kalloc()` 分配一个物理页作为 **一级页表页**, 记为 `kpgtbl`. 然后, 它反复调用 `kvmmap` 来进行虚拟内存到物理内存的映射:
* 将虚拟地址范围 `[0, PHYSTOP)` **直接映射** 到物理内存的相同地址范围. 其中
  + 外部设备 (UART, VIRTIO, PLIC) 寄存器的内存映像被设置为 **具有读、写权限, 但无执行权限**
  + 内核代码段 (地址范围 `[KERNBASE, etext)`) 被设置为 **具有读、执行权限, 但无写权限**
  + 内核数据段以及内核以外的内存 (地址范围 `[etext, PHYSTOP)`) 被设置为 **具有读、写权限, 但无执行权限**
  > 常量 `KERNBASE` 等于 `0x80000000`. 符号 `etext` 是在 linker script 中定义的 (`kernel/kernel.ld[19]`). 下面会看到, 它代表内核代码段的结束.
* 将虚拟地址 `TRAMPOLINE` 出发的一页映射到符号 `trampoline` 所在的物理页.
  + 常量 `TRAMPOLINE` 等于 `MAXVA - PGSIZE`. 也就是说, 它指向 xv6 虚拟地址空间最顶部的一页.
  + 符号 `trampoline` 定义在 `trampoline.S` 中:
    ```asm
    .section trampsec
    .globl trampoline
    trampoline:

    .align 4
    .globl uservec
    uservec:   
      ... 

    .globl userret
    userret:
      ...
    ```
    在这个文件中, 汇编指示符 `.section trampsec` 告诉汇编器, 把接下来的代码放在目标文件 `trampoline.o` 的 `trampsec` 一节中. 这一节包含了 `trampoline`, `uservec` 和 `userret` 三个全局符号.

    那么经过链接之后, 符号 `trampoline` 位于可执行文件的哪里呢? 这需要看 linker script 如何指示: (`kernel/kernel.ld[4:20]`)
    ```
    SECTIONS
    {
      . = 0x80000000;
    
      .text : {
        *(.text .text.*)
        . = ALIGN(0x1000);
        _trampoline = .;
        *(trampsec)
        . = ALIGN(0x1000);
        ASSERT(. - _trampoline == 0x1000, 
               "error: trampoline larger than one page");
        PROVIDE(etext = .);
      }
    
      /* ... */
    }
    ```
    语句块 `.text : { ... }` 告诉链接器, 把大括号里的内容拼起来放到可执行文件的 `.text` 节. 看看大括号里有什么:
    + `*(.text .text.*)` 表示: 查找所有输入目标文件 (通配符 `*`) 中以 `.text` 为前缀的节 (通配符 `.text` 或 `.text.*`), 将所有这些节合并起来放入可执行文件的 `.text` 节.
    + `. = ALIGN(0x1000);` 表示: 接下来适当地 zero padding, 找到第一个 `0x1000` 字节对齐 (也就是 **页对齐**) 的位置. 在该位置定义一个符号 `_trampoline`.
    + `*(trampsec)` 表示: 查找所有输入目标文件中的 `trampsec` 节, 合并起来放到此处. (实际上, 只有 `trampoline.o` 中有 `trampsec` 节.)
    + 又一条 `. = ALIGN(0x1000);` 要求找到下一个页对齐的位置. 我们希望该位置距离符号 `_trampoline` 正好等于一页. 这就要求 **所有目标文件中 (实际上仅 `trampoline.o` 中) 的 `trampsec` 节大小加起来不能超过一页**.
    + 最后定义符号 `etext`, 令它指向内核代码段的终点.

    至此我们已经破案了: `trampsec` 节被链接到内核代码段中的 **最后一页**. 因此, 内核加载到内存后, 符号 `trampoline` 就指向内核代码段地址最高的一页.

> 这样做的原因将在第 9 章和第 11 章中揭晓.

* 最后, `proc_mapstacks()` 函数为每个进程号分配 **内核栈**. 这个函数定义在 `kernel/proc.c[32:44]`:
```c
void proc_mapstacks(pagetable_t kpgtbl)
{
  struct proc *p;
  
  for(p = proc; p < &proc[NPROC]; p++) {
    char *pa = kalloc();
    if(pa == 0)
      panic("kalloc");
    uint64 va = KSTACK((int) (p - proc));
    kvmmap(kpgtbl, va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
  }
}
```
在这里, `NPROC = 64` 表示 xv6 中允许的最大进程数. 对于进程号为 `i` 的进程, `proc_mapstacks()` 调用 `kalloc()` 分配一个物理页, 并且把虚拟地址
```c
KSTACK(i) = TRAMPOLINE - (i + 1) * 2*PGSIZE;
```
映射到该物理页的基址. 这样, 在内核的虚拟地址空间中, 每个进程的内核栈各占一个虚拟页, 且两两之间相隔一个无效的虚拟页. 所有这些内核栈都位于 trampoline 页的下方, 且按照进程号 0, 1, 2, ... 在虚拟地址空间中由高地址向低地址排列.

综上所述, `kvmmake()` 函数完成了内核态虚拟地址空间的定义, 并且为所有有效的虚拟页建立了页表项. 现在我们可以回到 `main()` 函数,

```c
    kvminit();       // create kernel page table
```
前面已经提到, `kvminit()` 调用了 `kvmmake()`, 并把一级页表基址存入 `kernel_pagetable`.

```c
    kvminithart();   // turn on paging
```
`kvminithart()` 函数定义如下: (`kernel/vm.c[61:71]`)
```c
void kvminithart()
{
  // wait for any previous writes to the page table memory to finish.
  sfence_vma();

  w_satp(MAKE_SATP(kernel_pagetable));

  // flush stale entries from the TLB.
  sfence_vma();
}
```
`kvminithart()` 的中间一条语句把数值
```c
MAKE_SATP(kernel_pagetable) = (8L << 60) | (kernel_pagetable >> 12);
```
写入寄存器 `satp`. 根据前面的介绍, 这表示启用 Sv39 地址翻译模式, 并规定了一级页表的 PPN 为 `kernel_pagetable` 指向的物理页号.

在 `w_satp()` 语句前后, 都插入了一条 `sfence_vma()`. `sfence_vma()` 代表下列内联汇编:
```asm
sfence.vma zero, zero
```
这条指令也是 fence instruction. 它要求: **所有对地址翻译数据结构 (即各级页表) 的读写都不允许从该指令的一侧移到另一侧**. 此外, 这条指令还会清空 TLB 中缓存的页表项. 这样, 可以保证 `w_satp()` 前、后的内存访问分别按照正确的地址翻译模式进行 (通过第一次 `sfence_vma()`), 并且启用地址翻译后, 所有的 TLB 都是空的 (通过第二次 `sfence_vma()`).

一旦完成 `kvminit()` 和 `kvminithart()`, 我们就为 xv6 内核配置好了虚拟内存. 此后, xv6 内核就可以通过虚拟地址访存了.

