# 2. 初始化

在上一章结尾, 我们提到: xv6 内核在执行完 `kernel/entry.S` 中最初的几行代码后, 就跳转到 `kernel/start.c` 中定义的 `start()` 函数执行. 这一章我们就来阅读 `start()` 函数的代码. 这部分代码如下面所示: (`kernal/start.c[19:55]`)

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

CSR 是一系列存储有 RISC-V 机器运行关键信息的寄存器. 为了理解 xv6 内核干了什么, 我们需要对常见的 CSR 的功能有所了解. 因此, 我们将在讲解代码的同时介绍有关的背景知识. 这些背景知识均可以在 RISC-V 特权指令集文档 ([https://riscv.org/wp-content/uploads/2017/05/riscv-privileged-v1.10.pdf](https://riscv.org/wp-content/uploads/2017/05/riscv-privileged-v1.10.pdf)) 中找到.

## `mstatus` 寄存器

现在我们回到 `start()` 函数的代码. `start()` 函数的第一步访问了 `mstatus` 寄存器:
```c
  // set M Previous Privilege mode to Supervisor, for mret.
  unsigned long x = r_mstatus();
  x &= ~MSTATUS_MPP_MASK;   // MSTATUS_MPP_MASK = 3L << 11;
  x |= MSTATUS_MPP_S;       // MSTATUS_MPP_S = 1L << 11;
  w_mstatus(x);
```
****

`mstatus` 寄存器存储了有关机器状态的重要信息. RISC-V 特权指令集文档给出的 `mstatus` 寄存器结构如下图所示:

![mstatus 寄存器](/chapters/ch2-figs/mstatus.png "mstatus 寄存器")

* **WPRI** 是 "writes preserve, reads ignore" 的缩写. 也就是说, 程序员应当保证: 写寄存器时, 不要改变那些被 **WPRI** 标记的比特位的值; 读寄存器时, 应当忽略那些被 **WPRI** 标记的比特位的值. 这些比特位通常是保留给将来使用的, 因此希望程序员不要在现在改动它们或使用它们的值.
* 在 `mstatus` 中有一些成三个出现的比特位, 如 MPP/SPP/UPP, MIE/SIE/UIE, MPIE/SPIE/UPIE 等. 当 RISC-V 处理器特权级别为 x 时 (x 代表 M/S/U), 实际上 **只会使用相应于 x 级别的 `xPP`/`xIE`/`xPIE`**. 尽管如此, 高特权级别仍然拥有对低特权级别比特位的访问权, 例如 M 模式可以访问所有的 MPP/SPP/UPP, ... 等比特位.

> 在不同的处理器特权级别下, 对 CSR 有着不同的访问权限. S 模式仍然允许访问 `mstatus` 的部分比特位; 这些在 S 模式下仍可访问的比特位在 ISA 中被封装到了 `sstatus` 寄存器中. 下面展示了 `sstatus` 的寄存器结构:
>
> ![sstatus 寄存器](/chapters/ch2-figs/sstatus.png "sstatus 寄存器")
>
> 可以看到, `sstatus` 将 `mstatus` 中那些只有 M 模式才可访问的比特位封装为 **WPRI**. 在 S 模式下, 只有 SPP/UPP, SIE/UIE, SPIE/UPIE 这些比特位是可访问的.

* MPP/SPP/UPP 表示 "(M/S/U) previous privilege". 假如当前特权级别是 x, 则 `xPP` 表示 **trap 到当前特权级别 x 之前, 机器处在哪一个特权级别**.

> RISC-V 用两个比特位来编码特权级别, 具体如下:
>
> |特权级别|编码|
> |:---:|:---:|
> | U (user) | `00` |
> | S (supervisor) | `01`|
> | (未使用, 保留) | `10` |
> | M (machine) | `11` |

因为处理器只能从低特权级别陷入到高特权级别, 所以 xPP 编码的特权级别不会高于 x. 从而, MPP 需要两个比特位, 而 SPP/UPP 只需要一个比特位.

* 比特位 MIE/SIE/UIE 和 MPIE/SPIE/UPIE 与中断控制密切相关, 我们将在下一节 "中断控制与中断代理" 中继续介绍.

****

回过头来, `start()` 函数开头的这几行代码干了什么呢? 它读取了 `mstatus` 的值, 将 11~12 两位的值设置成 `01`, 然后写回 `mstatus`. 这样做的效果就是: **将 MPP 设置成 S 模式**. 也就是说, xv6 内核营造出这样的假象——仿佛之前发生过一次由 S 模式到 M 模式的陷入. 这样, 当陷入返回之后 (通过一条 `mret` 指令), 处理器就会自动将特权级别 "设置回" 我们设定的 MPP 的值, 即 S 模式.


## 中断控制与中断代理

**中断** 是一种硬件事件, 它要求 CPU 暂时离开 "取指令-执行" 的工作循环以进行一定的处理. 在 RISC-V 中, **中断是引起 CPU 特权级别从低级别变为高级别的唯一机制**. RISC-V 进一步地把中断分为 **interrupt** (外中断, 由 CPU 外产生) 和 **exception** (内中断, 由 CPU "取指令-执行" 循环自身引发). 

RISC-V 中的每一种中断都有一个相应的特权级别. 中断 i 的特权级别为 x, 意味着中断 i 是由特权级别 x 处理的. 因此, 假如当前特权级别低于 x, 那么为了处理中断 i, 就必须 **陷入** 到特权级别 x. 默认情况下, RISC-V 所有中断的特权级别都是 M.

### 中断开关

* MIE/SIE/UIE 表示 "(M/S/U) interrupt enabled". 如果当前特权级别是 x, 那么 `xIE=1` 表示当前 **允许特权级为 x 的中断**, `xIE=0` 表示当前 **不允许特权级为 x 的中断**.

* MPIE/SPIE/UPIE 表示 "(M/S/U) previously interrupt enabled". 如果当前特权级别是 x, 那么 `xPIE` 用来保存 **陷入到当前特权级别 x 之前 `xIE` 的值**.

> 假如发生了一次从特权级别 y 到特权级别 x 的陷入, 那么
> * `xPIE` 被设为 `xIE`
> * `xIE` 被设为 0 (默认关中断)
> * `xPP` 被设为 y
> * 特权级别被设为 x
>
> 使用指令 `xret` (`x=m/s/u`) 从陷入返回时, 会发生
> * `xIE` 被设为 `xPIE`
> * `xPIE` 被设为 1
> * 特权级别被设为 `xPP`
> * `xPP` 被设为 U (用户模式)
