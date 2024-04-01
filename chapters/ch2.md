# 0x02. 初始化

在上一章结尾, 我们提到: xv6 内核在执行完 `kernel/entry.S` 中最初的几行代码后, 就跳转到 `kernel/start.c` 中定义的 `start()` 函数执行. 这一章我们就来阅读 `start()` 函数的代码. 这部分代码如下面所示: (`kernel/start.c[19:55]`)

```c
// entry.S jumps here in machine mode on stack0.
void
start()
{
  // set M Previous Privilege mode to Supervisor, for mret.
  unsigned long x = r_mstatus();
  x &= ~MSTATUS_MPP_MASK;
  x |= MSTATUS_MPP_S;
  w_mstatus(x);

  // set M Exception Program Counter to main, for mret.
  // requires gcc -mcmodel=medany
  w_mepc((uint64)main);

  // disable paging for now.
  w_satp(0);

  // delegate all interrupts and exceptions to supervisor mode.
  w_medeleg(0xffff);
  w_mideleg(0xffff);
  w_sie(r_sie() | SIE_SEIE | SIE_STIE | SIE_SSIE);

  // configure Physical Memory Protection to give supervisor mode
  // access to all of physical memory.
  w_pmpaddr0(0x3fffffffffffffull);
  w_pmpcfg0(0xf);

  // ask for clock interrupts.
  timerinit();

  // keep each CPU's hartid in its tp register, for cpuid().
  int id = r_mhartid();
  w_tp(id);

  // switch to supervisor mode and jump to main().
  asm volatile("mret");
}
```

## 内联汇编

上面的代码中充斥着大量名为 `w_xxxx()` 或 `r_xxxx()` 的函数. (`xxxx` 是某个寄存器名.) 如果查看一下这些函数的定义 (在 `kernel/riscv.h` 中), 会发现是这样的:
```c
static inline void w_xxxx(uint64 x)
{
  asm volatile("csrw xxxx, %0" : : "r" (x));
}

static inline uint64 r_xxxx()
{
  uint64 x;
  asm volatile("csrr %0, xxxx" : "=r" (x) );
  return x;
}
```
或者是这样的:
```c
static inline void w_yyyy(uint64 x)
{
  asm volatile("mv yyyy, %0" : : "r" (x));
}

static inline uint64 r_yyyy()
{
  uint64 x;
  asm volatile("mv %0, yyyy" : "=r" (x) );
  return x;
}
```
这些函数是所谓的 **内联汇编**. 不难看出 `w_xxxx()` 的作用是将 8 字节的参数 `x` 写入寄存器 `xxxx`, 而 `r_xxxx()` 的作用是读寄存器 `xxxx` 的值并返回.

> 关于内联汇编的语法, [这里](https://www.xmos.com/download/AN10022:-How-to-use-inline-assembly(1.0.0rc4).pdf)有个很简短但自洽的介绍. 阅读完这个介绍 (只需约 2 分钟时间) 对于理解上面的这些内联汇编已经够用了.

同样是执行寄存器读写, 却需要使用两种不同的指令: `mv` 或者 `csrw`/`csrr`. 这是因为 `csrw`/`csrr` 是 RISC-V 专门用来写/读 CSR (control and status registers) 的指令, 它不同于读写通用寄存器的指令 `mv`. 

> CSR 是一系列存储有 RISC-V 机器运行关键信息的寄存器. 为了理解 xv6 内核干了什么, 我们需要对常见的 CSR 的功能有所了解. 因此, 我们将在讲解代码的同时介绍有关的背景知识. 这些背景知识均可以在 RISC-V 特权指令集文档 ([https://riscv.org/wp-content/uploads/2017/05/riscv-privileged-v1.10.pdf](https://riscv.org/wp-content/uploads/2017/05/riscv-privileged-v1.10.pdf)) 中找到.

## `mstatus` 寄存器

`start()` 函数的最开始几行访问了 `mstatus` 寄存器:
```c
  // set M Previous Privilege mode to Supervisor, for mret.
  unsigned long x = r_mstatus();
  x &= ~MSTATUS_MPP_MASK;   // MSTATUS_MPP_MASK = 3L << 11;
  x |= MSTATUS_MPP_S;       // MSTATUS_MPP_S = 1L << 11;
  w_mstatus(x);
```

`mstatus` 寄存器存储了有关机器状态的重要信息. RISC-V 特权指令集文档给出的 `mstatus` 寄存器结构如下图所示:

![mstatus 寄存器](/chapters/ch2-figs/mstatus.png "mstatus 寄存器")

上面的几行代码的效果是 **把 `mstatus` 的 11~12 两位设置成了 `01`**. 查一下 `mstatus` 的结构, 会发现这两位对应于 MPP. RISC-V 特权指令集文档告诉我们:

* `mstatus` 中的 MPP/SPP/UPP 表示 "(M/S/U) previous privilege". 假如当前特权级别是 x (x 为 M/S/U), 则 `xPP` 存储了 **trap 到当前特权级别 x 之前, 机器处在哪一个特权级别**.

* RISC-V 用两个比特位来编码特权级别, 具体如下:

  |特权级别|编码|
  |:---:|:---:|
  | U (user) | `00` |
  | S (supervisor) | `01`|
  | (未使用, 保留) | `10` |
  | M (machine) | `11` |

也就是说, 上面的代码把 `mstatus` 中的 MPP 位设置成了 S 模式. 换言之, xv6 内核试图营造出这样的假象——**仿佛之前发生过一次由 S 模式到 M 模式的陷入**. 这样, 当陷入返回之后 (通过一条 `mret` 指令), 处理器就会自动将特权级别 "设置回" 我们设定的 MPP 的值, 即 S 模式.

> 因为处理器只能从低特权级别陷入到高特权级别, 所以 xPP 编码的特权级别不会高于 x. 从而, MPP 需要两个比特位, 而 SPP/UPP 只需要一个比特位.

接下来的一行代码
```c
  // set M Exception Program Counter to main, for mret.
  // requires gcc -mcmodel=medany
  w_mepc((uint64)main);
```
把 `mepc` 设置为 `main()` 函数 (定义在 `kernel/main.c` 中) 的地址. 寄存器 `xepc` (`x=m/s`) 表示: 当我们从一次陷入到特权级别 x 的过程返回时 (通过 `xret` 指令), 应该把 PC 设置成什么值? 通过这条指令, 当我们随后执行 `mret` 时, PC 就被设置成 `main()` 函数的地址.

继续往下看,
```c
  // disable paging for now.
  w_satp(0);
```
`satp` 是一个在 S 模式管理地址翻译和保护 (address translation and protection) 的寄存器. 当它的最高位 (对于 RV32) 或最高 4 位 (对于 RV64) 全置 0 时, 代表 **在 S 模式下禁用地址翻译**, 即机器指令中所有的内存地址就是物理地址.

> RISC-V 规定: 在 M 模式下, 永远不启用地址翻译. 因此 M 模式下的访存, 使用的地址都是物理地址, 也不存在一个类似于 `satp` 的 `matp` 寄存器.
>
> 我们这里禁用 S 模式下的地址翻译, 只是暂时的. 在初始化虚拟内存的时候 (文件 `kernel/vm.c` 中定义的 `kvminithart()` 函数) 我们会重新设置 `satp` 寄存器打开地址翻译.
>
> 由于通常 S 模式下我们会使用虚拟地址访存, 而 M 模式下却必须使用物理地址, 当我们有时不得不从 S 模式切换到 M 模式的时候, 上述区别会给我们带来麻烦. 之后会看到, xv6 处理 **时钟中断** 的时候面临的正是这样一种情形.

## 中断控制与中断代理

随后的几行代码
```c
  // delegate all interrupts and exceptions to supervisor mode.
  w_medeleg(0xffff);
  w_mideleg(0xffff);
  w_sie(r_sie() | SIE_SEIE | SIE_STIE | SIE_SSIE);
```
与中断控制有关. **中断** 是一种硬件事件, 它要求 CPU 暂时离开 "取指令-执行" 的工作循环以进行一定的处理. 在 RISC-V 中, **中断是引起 CPU 特权级别从低级别变为高级别的唯一机制**. RISC-V 进一步地把中断分为 **interrupt** (外中断, 由 CPU 外产生) 和 **exception** (内中断, 由 CPU "取指令-执行" 循环自身引发). 下图展示了 RISC-V 支持的所有中断 (包括 interrupt 和 exception) 以及它们各自的中断码:

![RISC-V 支持的中断](/chapters/ch2-figs/intr_table.png "RISC-V 支持的中断")

RISC-V 中的每一种中断都有一个相应的特权级别. 中断 i 的特权级别为 x, 意味着中断 i 需要由特权级别 x 处理. 因此, 假如当前特权级别低于 x, 那么为了处理中断 i, 就必须 **陷入** 到特权级别 x. 默认情况下, **RISC-V 所有中断的特权级别都是 M**, 即: 为了处理任何中断, 都必须陷入到机器模式.

### 中断开关

有时, 我们希望 CPU 不要被中断打断. 通过 **中断开关** 可以实现中断机制的开启和关闭. 有两种中断开关:

* **全局中断开关.** 这些开关位于 `mstatus` 寄存器中:
  + `mstatus` 寄存器中的 MIE/SIE/UIE 比特位表示 "(M/S/U) interrupt enabled". 如果当前特权级别是 x, 那么 `xIE=1` 表示当前 **允许特权级为 x 的中断**, `xIE=0` 表示当前 **不允许特权级为 x 的中断**.
  + `mstatus` 寄存器中的 MPIE/SPIE/UPIE 表示 "(M/S/U) previously interrupt enabled". 如果当前特权级别是 x, 那么 `xPIE` 用来保存 **陷入到当前特权级别 x 之前 `xIE` 的值**.
> 请注意: 设置 `xIE=0` 只是在 **处理器特权级等于 x 时, 禁用特权级为 x 的中断**. 假如处理器特权级低于 x, 那么 RISC-V 规定 **总是要响应** 特权级为 x 的中断 (即中断发生时总会 trap 到 x 特权级), 而不受 `xIE` 比特的影响. 因此, 假如 S 模式或 U 模式下发生了一次特权级为 M 的中断, RISC-V 总会陷入到 M 模式, 即便 `MIE=0`.
>
> 更完整的解释可以参见 RISC-V 特权指令集文档:
>
> An interrupt traps to M-mode whenever all of the following are true: 
> * either the current privilege mode is M-mode and machine-level interrupts are enabled by the MIE bit of `mstatus`, or the current privilege mode has less privilege than M-mode
> * matching bits in `mip` and `mie` are both 1
> * if `mideleg` exists, the corresponding bit in `mideleg` is 0
>

* **局部中断开关.** `mie` 寄存器包含一系列比特位, 每一个比特位对应一种 **外中断**. 假如将外中断 i 对应的 `mie` 比特位置 0, 那么就禁止响应外中断 i. 

> 在不同的处理器特权级别下, 对 CSR 有着不同的访问权限. 例如, S 模式仍然允许访问 `mstatus` 的部分比特位; 这些在 S 模式下仍可访问的比特位在 ISA 中被封装到了 `sstatus` 寄存器中. 下面展示了 `sstatus` 的寄存器结构:
>
![sstatus 寄存器](/chapters/ch2-figs/sstatus.png "sstatus 寄存器")
>
> 可以看到, `sstatus` 将 `mstatus` 中那些只有 M 模式才可访问的比特位封装为不可见的. 在 S 模式下, 只有 SPP/UPP, SIE/UIE, SPIE/UPIE 这些比特位是可访问的. 
> 
> 类似地, 在 S 模式可以访问 `sie` 寄存器, 它是 `mie` 的子集.

### 中断代理

如上所述, RISC-V 默认要求所有的中断都必须陷入到 M 模式处理. 那么看上去, `mstatus` 中的 SIE/UIE, SPIE/UPIE, 以及 `sie` 寄存器都是没用的了. (因为这些比特只有当处理器特权模式为 S/U 时才生效, 但是当一个 M 级别中断发生的时候, 无论这些比特取何值都必须陷入 M 模式.)

此外, 操作系统通常运行在 S 模式下. 这意味着操作系统不能直接处理中断. 为解决这个问题, RISC-V 提供了 **中断代理** 机制. 

RISC-V 提供了 `mideleg` 和 `medeleg` 寄存器, 分别代表 interrupt delegation 和 exception delegation. 如果将 `mideleg`/`medeleg` 寄存器的第 i 位设为 1, 那么就说明将中断码为 i 的外中断/内中断 **代理给 S 模式处理**.

> 当然, 并不是所有类型的中断都允许代理给 S 模式. 假如 RISC-V 不允许中断 i 被代理给 S 模式, 那么即使我们用一条指令把 `mideleg`/`medeleg` 的第 i 位设为 1, 硬件也会直接忽略这个命令, 下次读这俩寄存器的第 i 位时, 读出的值还是 0.
>
> 查阅一下 RISC-V 文档就会发现, 寄存器 `mideleg`/`medeleg` 的每一位都是 WARL (write anything, read legal). 也就是说, 程序员可以随便写这两个寄存器, 但是硬件会保证从这两个寄存器读出的值总是合法的. 换言之, 如果硬件不支持修改这两个寄存器中的某些比特位, 那么程序员写它们就不会生效.
>
> 在我们这里, **时钟中断** 就是这样的一个例子. 时钟中断会产生一种叫做 machine timer interrupt (MTI) 的外中断 (中断码为 7). 这种中断不允许代理给 S 模式. 即使把 `mideleg` 的第 7 位改成 1, 下次读的时候得到的还是 0.

现在我们可以来看 `start()` 中相关的代码:

```c
  w_medeleg(0xffff);
  w_mideleg(0xffff);
```
将 `mideleg` 和 `medeleg` 的低 16 位设为 1. 也就是说, 我们试图 **把所有内中断/外中断全部代理给 S 模式处理**.

> 根据上面的讨论, 我们并不应该期望所有这些代理都生效. 事实上, 可以用 `gdb` 给这两条指令之后的位置打断点. 待程序执行完这两条指令后, 监测一下 `mideleg` 和 `medeleg` 的值:
> ```
> (gdb) info registers mideleg
> mideleg        0x666    1638
> (gdb) info registers medeleg
> medeleg        0xbfff   49151
> ```
> 会发现许多位并没有被设为 1. 特别地, 我们会发现 `mideleg` 中, 只有 supervisor external interrupt (SEI), supervisor timer interrupt (STI), supervisor software interrupt (SSI) 三种外中断被真正代理给了 S 模式处理. 而 **时钟中断** (MTI) 并没有代理给 S 模式. 这就意味着 xv6 在 S 模式运行过程中, 每逢时钟中断还是会 trap 到 M 模式去处理.

```c
  w_sie(r_sie() | SIE_SEIE | SIE_STIE | SIE_SSIE);
```
将 `sie` 中对应 SEI, STI 和 SSI 的三个比特设为 1. 

> 同样地, 开启这三个比特位并不代表 S 模式已经可以处理所有的中断. 此外, 要真正启用中断, 还需要把 `mstatus` 中的 SIE 置为 1.

## 从 M 模式到 S 模式

现在我们接着来看 `start()` 函数的剩余部分:

```c
  // configure Physical Memory Protection to give supervisor mode
  // access to all of physical memory.
  w_pmpaddr0(0x3fffffffffffffull);
  w_pmpcfg0(0xf);
```
这两条指令管理物理内存保护 (physical memory protection). 它们的效果是把物理地址 $[0,2^{56}-1]$ 的范围设为所有模式 (M/S/U) 都可读、可写、可执行的. 

> RISC-V 特权指令集文档告诉我们: 
> * `pmpcfg0` 的 0, 1, 2 位分别代表一段物理内存的可读、可写、可执行权限. 这段物理内存的地址范围由 `pmpcfg0` 的 3~4 位, 以及 `pmpaddr0` 共同决定
> * 当 `pmpcfg0` 的 3~4 位为 `01` 时, 上述的地址范围就是低于 `(pmpaddr0 + 1) * 4` 字节的全部物理地址, 因此也就是低于 `(0x3fffffffffffff + 1) * 4`, 即 $2^{56}$ 字节的全部地址.

接下来的一行代码是
```c
  // ask for clock interrupts.
  timerinit();
```
`timerinit()` 函数完成了 **时钟中断处理** 的初始化. 执行完这个函数之后, 我们就能够正确应付时钟中断. 这是我们首次接触 **中断处理** 的代码. 我们将在下一章阅读 `timerinit()` 函数的细节.

> 为什么要把时钟中断处理放在最开始就完成呢? 这是因为时钟中断是 xv6 系统中唯一的 **必须陷入 M 模式才能处理** 的中断. 因此, 对时钟中断处理的初始化必须在特权级为 M 模式时完成, 也就是现在.

继续往下看:
```c
  // keep each CPU's hartid in its tp register, for cpuid().
  int id = r_mhartid();
  w_tp(id);
```
把核心号 (`mhartid`) 存入寄存器 `tp`. `tp` 就是所谓的 "线程指针" (thread pointer).

```c
  // switch to supervisor mode and jump to main().
  asm volatile("mret");
```
执行 `mret` 指令. 这条指令主要的效果是:
* 把 `mstatus` 中的 MIE 设为 MPIE, 而 MPIE 设为 1
* 特权级别被设为 `mstatus` 中 MPP 的值, 也就是 **S 模式**, 而 MPP 设为 `00` (U 模式)
* PC 被设为 `mepc` 中的值, 也就是 `main()` 函数的地址

从现在开始, 处理器特权级别就由 M 模式变成了 S 模式. 并且处理器接下来将跳转到 `main()` 函数开始执行.
