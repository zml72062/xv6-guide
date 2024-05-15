# 3. 处理时钟中断

在[上一章](ch2.md)我们读完了 `start()` 函数的代码. 这个函数完成了 M 模式下一切必要的准备工作, 然后回到 S 模式. 不过, 我们仍然遗留了一部分的代码到这一章, 也就是函数 `timerinit()` 的代码.

## 初识设备驱动程序

`timerinit()` 函数完成了 **时钟中断处理** 的初始化工作. 这个函数定义在 `kernel/start.c[57:89]`:
```c
// arrange to receive timer interrupts.
// they will arrive in machine mode at
// at timervec in kernelvec.S,
// which turns them into software interrupts for
// devintr() in trap.c.
void
timerinit()
{
  // each CPU has a separate source of timer interrupts.
  int id = r_mhartid();

  // ask the CLINT for a timer interrupt.
  int interval = 1000000; // cycles; about 1/10th second in qemu.
  *(uint64*)CLINT_MTIMECMP(id) = *(uint64*)CLINT_MTIME + interval;

  // prepare information in scratch[] for timervec.
  // scratch[0..2] : space for timervec to save registers.
  // scratch[3] : address of CLINT MTIMECMP register.
  // scratch[4] : desired interval (in cycles) between timer interrupts.
  uint64 *scratch = &timer_scratch[id][0];
  scratch[3] = CLINT_MTIMECMP(id);
  scratch[4] = interval;
  w_mscratch((uint64)scratch);

  // set the machine-mode trap handler.
  w_mtvec((uint64)timervec);

  // enable machine-mode interrupts.
  w_mstatus(r_mstatus() | MSTATUS_MIE);

  // enable machine-mode timer interrupts.
  w_mie(r_mie() | MIE_MTIE);
}
```
`timerinit()` 函数是我们接触的第一个 **设备驱动程序**. 总的来说, 设备驱动程序的职责是:
* 设置设备的有关寄存器, 完成设备硬件状态的初始化
* 提供操纵设备的软件接口
* 在底层处理设备中断

时钟设备为上层提供的接口是很简单的: 对于每个 CPU 核心, 它提供 **一对寄存器 `mtime` 和 `mtimecmp`**. 在命令行参数 `-machine virt` 下, `qemu` 把控制时钟设备 (core local interruptor, CLINT) 的寄存器映射到了从物理地址 `0x2000000` 开始的一片内存区域 (见[第 1 章](ch1.md)的物理内存布局图). 为了方便访问这些寄存器, 在 `kernel/memlayout.h[28:31]` 中定义了下列宏:
```c
// core local interruptor (CLINT), which contains the timer.
#define CLINT 0x2000000L
#define CLINT_MTIMECMP(hartid) (CLINT + 0x4000 + 8*(hartid))
#define CLINT_MTIME (CLINT + 0xBFF8) // cycles since boot.
```
寄存器 `mtime` 和 `mtimecmp` 分别被内存映射到地址 `CLINT_MTIME` 和 `CLINT_MTIMECMP(hartid)` (每个 CPU 核心有一个自己的 `mtimecmp` 寄存器). 具体来讲, 位于地址 `CLINT_MTIME` 的 8 字节整数中实时存储了自机器启动以来经历的周期数; 对于核心号为 `i` 的处理器, 当 
```c
*(uint64*)CLINT_MTIME > *(uint64*)CLINT_MTIMECMP(i)
```
时, 会触发一次时钟中断 (MTI) 给处理器 `i`. 那么 `timerinit()` 的前几行代码
```c
  // each CPU has a separate source of timer interrupts.
  int id = r_mhartid();

  // ask the CLINT for a timer interrupt.
  int interval = 1000000; // cycles; about 1/10th second in qemu.
  *(uint64*)CLINT_MTIMECMP(id) = *(uint64*)CLINT_MTIME + interval;
```
其实就是: 给每个处理器核心都设了一个 1000000 周期之后的闹钟.

接下来的几行代码
```c
  // prepare information in scratch[] for timervec.
  // scratch[0..2] : space for timervec to save registers.
  // scratch[3] : address of CLINT MTIMECMP register.
  // scratch[4] : desired interval (in cycles) between timer interrupts.
  uint64 *scratch = &timer_scratch[id][0];
  scratch[3] = CLINT_MTIMECMP(id);
  scratch[4] = interval;
  w_mscratch((uint64)scratch);
```
在内存中维护了一个数据结构 `timer_scratch`, 它定义在 `kernel/start.c[13:14]`:
```c
// a scratch area per CPU for machine-mode timer interrupts.
uint64 timer_scratch[NCPU][5];
```
`timer_scratch` 中给每个 CPU 核心分配了 40 字节的内存. 对于核心号为 `i` 的 CPU, 分配给它的这 40 字节的起始地址就是 `&timer_scratch[i][0]`, 也就是上面代码中的 `scratch`.

这 40 字节内存的结构如下图所示:
```
0           8           16          24          32      (bytes)  
|-----------|-----------|-----------|-----------|-----------|  
|   save    |   save    |   save    |  MTIMECMP |  interval |
|    a1     |    a2     |    a3     | for CPU i | (1000000) |
|-----------|-----------|-----------|-----------|-----------|
```
* 前 24 字节是预留出来用于保存寄存器 `a1`, `a2`, `a3` 的
* 第 24~32 字节记录了 `CLINT_MTIMECMP(i)`, 回忆一下, 这是存储 "闹钟预定时间" 的地址
* 第 32~40 字节记录了每隔多少周期触发一次时钟中断 (在这里是 1000000)

设置完 `scratch` 指向的内存区域的内容后, 
```c
  w_mscratch((uint64)scratch);
```
把 `scratch` 的地址存入 `mscratch` 寄存器.

> `mscratch` 是个 M 模式下可访问的 CSR, 它的功能是用来储存一些临时值. 
>
>我们这里使用 `mscratch` 存放 `scratch` 的物理地址. 这样, 当处理器陷入 M 模式处理时钟中断时 (注意 M 模式总是使用物理地址访存), 不需要进行复杂的地址翻译就可以直接通过物理地址访问所需要的数据结构.
>

接下来的一条指令 
```c
  // set the machine-mode trap handler.
  w_mtvec((uint64)timervec);
```
把 `mtvec` 寄存器设置为函数 `timervec()` 的首地址. 这个 `timervec()` 函数是用汇编语言写的, 它定义在 `kernel/kernelvec.S` 中. 

寄存器 `mtvec` 存储了 M 模式下 **中断向量表** 的基址, 具体来说:
* 当 `mtvec` 的低 2 位为 `00` 时, 代表着在 M 模式下, 无论发生什么中断, 硬件都跳转到地址 `mtvec` (它整体是一个 4 字节对齐的地址) 去执行.
* 当 `mtvec` 的低 2 位为 `01` 时, 代表在 M 模式下发生中断时, 按下列方式跳转:
  + 如果是内中断, 跳转到 `mtvec - 1` (4 字节对齐) 执行.
  + 如果是外中断, 且中断码为 `i` (见[第 2 章](ch2.md)的表格 "RISC-V 支持的中断" 中 exception code 一列), 就跳转到以 `mtvec - 1` (4 字节对齐) 为基址, 偏移量 `4 * i` 字节的地址去执行.

`timervec()` 函数的代码如下所示: (`kernel/kernelvec.S[93:124]`)
```asm
.globl timervec
.align 4
timervec:
    # start.c has set up the memory that mscratch points to:
    # scratch[0,8,16] : register save area.
    # scratch[24] : address of CLINT's MTIMECMP register.
    # scratch[32] : desired interval between interrupts.
    
    csrrw a0, mscratch, a0
    sd a1, 0(a0)
    sd a2, 8(a0)
    sd a3, 16(a0)

    # schedule the next timer interrupt
    # by adding interval to mtimecmp.
    ld a1, 24(a0) # CLINT_MTIMECMP(hart)
    ld a2, 32(a0) # interval
    ld a3, 0(a1)
    add a3, a3, a2
    sd a3, 0(a1)

    # arrange for a supervisor software interrupt
    # after this handler returns.
    li a1, 2
    csrw sip, a1

    ld a3, 16(a0)
    ld a2, 8(a0)
    ld a1, 0(a0)
    csrrw a0, mscratch, a0

    mret
```
汇编器引导符 `.align 4` 规定了 `timervec()` 的首地址须是 4 字节对齐的. 因此, 把 `mtvec` 设为 `timervec()` 的首地址, 就意味着在 M 模式无论发生什么中断, 处理器都会跳转到 `timervec()` 执行.

> 这看起来不甚合理. 不过好在, 时钟中断是 **唯一一个** 我们需要在 M 模式下 handle 的中断. 其他所有中断都可以在 S 模式中处理 (通过中断代理机制), 而 S 模式下中断向量表的基址存储在一个不同的寄存器 `stvec` 中.

在具体研究 `timervec()` 的代码之前, 我们先看完 `timerinit()` 最后的几行代码:
```c
  // enable machine-mode interrupts.
  w_mstatus(r_mstatus() | MSTATUS_MIE);

  // enable machine-mode timer interrupts.
  w_mie(r_mie() | MIE_MTIE);
```
这两条代码开启了 M 模式下的全局中断开关 (`mstatus` 中的 MIE), 以及时钟中断 (MTI) 对应的局部中断开关. 从现在开始, M 模式下就可以接收并响应 MTI 中断了.

> 看起来, M 模式下还可以响应 SEI, STI 和 SSI 这三种中断. 因为回忆一下, 在[上一章](ch2.md)中, 调用 `timervec()` 之前有一条指令
> ```c
>   w_sie(r_sie() | SIE_SEIE | SIE_STIE | SIE_SSIE);
> ```
> 把 `sie` 中对应于 SEI, STI 和 SSI 的中断开关打开了. 而 `mie` 是 `sie` 的超集, 所以 `mie` 中现在总共有 4 个比特位被置 1. 使用调试器观察一下, 也会发现:
> ```
> (gdb) info registers mie
> mie            0x2a2    674
> ```
> `mie` 中置 1 的 4 个比特正好对应 MTI, SEI, STI 和 SSI.
>
> 不过, 由于我们已经把 SEI, STI 和 SSI 代理给 S 模式处理, RISC-V 文档对于这种情况有如下解释:
>
> An interrupt i will be taken if bit i is set in both `mip` and `mie`, and if interrupts are globally enabled. By default, M-mode interrupts are globally enabled if the hart’s current privilege mode is less than M, or if the current privilege mode is M and the MIE bit in the `mstatus` register is set. If bit i in `mideleg` is set, however, interrupts are considered to be globally enabled if the hart’s current privilege mode equals the delegated privilege mode (S or U) and that mode’s interrupt enable bit (SIE or UIE in `mstatus`) is set, or if the current privilege mode is less than the delegated privilege mode.
>
> 由于 M 模式的特权级别高于 S 模式, 所以在 M 模式下会直接忽略 SEI, STI 或 SSI 中断.
>

## 时钟中断处理函数 `timervec()`

我们现在回过头来研究 `timervec()` 函数的代码. 这个函数是一个 interrupt handler; 也就是说, 发生时钟中断的时候硬件会自动跳转到这个函数去执行.

```asm
    csrrw a0, mscratch, a0      # now a0 holds address "scratch"
    sd a1, 0(a0)
    sd a2, 8(a0)
    sd a3, 16(a0)
```
* 第一条 `csrrw` 指令原子地 (atomically) 交换寄存器 `mscratch` 和 `a0` 的值. 回忆一下, `mscratch` 中存储了地址 `scratch` (也就是那 40 字节的数据结构的首地址); 于是现在 `a0` 就持有地址 `scratch`
* 接下来 3 条指令把 `a1`, `a2`, `a3` 的值保存在了以 `scratch` 为首地址的前 24 字节 (因为我们之后会破坏这三个寄存器的值)

```asm
    ld a1, 24(a0) # CLINT_MTIMECMP(hart)
    ld a2, 32(a0) # interval
    ld a3, 0(a1)
    add a3, a3, a2
    sd a3, 0(a1)
```
这几条指令干了这些事情:
* 把 `*(uint64*)(scratch + 24)` 存入寄存器 `a1`. 对于核心号为 `i` 的 CPU, 这就是 `CLINT_MTIMECMP(i)` 的值
* 把 `*(uint64*)(scratch + 32)` 存入寄存器 `a2`. 这是 `interval` 的值
* 执行 C 语句 `*a1 += a2`. 也就是说, 
    ```c
    *(uint64*)CLINT_MTIMECMP(i) += interval;
    ```

至此, 我们为当前正在执行指令的 CPU 设置了一个新的闹钟, 即再过 1000000 个周期触发一个新的时钟中断.

```asm
    li a1, 2
    csrw sip, a1
```
我们把 `sip` 的值设置为 `0x2`.

`mip`/`sip` 分别是 M 模式或 S 模式下的 **外中断待处理 (pending)** 寄存器. `sip` 是 `mip` 的子集. 假如 `mip` 的第 i 位被置 1, 那么硬件就会认为有一个中断码为 i 的外中断到来了. 如果此时全局中断开关是开着的, 且相应于外中断 i 的局部中断开关也开着, 那么硬件就会跳转到中断向量表中的相应条目去执行.

> 可以用 `gdb` 调试这段代码:
> * 执行到 `sd a3, 0(a1)` 语句之前, 查看 `mip` 寄存器,
> ```
> (gdb) info registers mip
> mip            0x80     128
> ```
> 我们发现 `mip` 的第 7 位被置 1. 它对应的正好是 machine timer interrupt, 即时钟中断. 这表明: **当前有一个时钟中断处于 pending 状态.** 当然, 确实如此.
> * 执行完 `sd a3, 0(a1)` 语句, 我们再次查看 `mip` 寄存器, 发现
> ```
> (gdb) info registers mip
> mip            0x0      0
> ```
> 这是因为, 这条语句把 `*(uint64*)CLINT_MTIMECMP(i)` 增加了 1000000. 于是, 时钟中断的触发条件 `*(uint64*)CLINT_MTIME > *(uint64*)CLINT_MTIMECMP(i)` 不再满足, 从而 `mip` 的第 7 位被清零.
> * 接着到执行完 `csrw sip, a1` 语句, 我们发现
> ```
> (gdb) info registers mip
> mip            0x2      2
> ```
> 对应于我们手动设置的值.
>

通过把 `sip` 的第 1 位置 1, 我们手动触发了一个 supervisor software interrupt (SSI). 为什么要这么做呢? 这是因为, 操作系统可能要根据时钟中断进行一些相当复杂的处理 (例如, 进程调度). 在 M 模式下完成所有这些工作, 是相当麻烦的. (例如, M 模式禁用地址翻译机制, 导致我们必须非常小心地使用物理地址访存, 而不是虚拟地址.) 因此, 我们的解决方案是: **在 M 模式下手动触发一个 S 模式可处理的中断 (SSI), 然后回到 S 模式**. 

这样做的效果是: 当我们使用一条 `mret` 指令回到 S 模式时, 硬件会检查 `sip`, 发现有一个 SSI 中断在 pending, 于是自动跳转到 SSI 对应的 interrupt handler 执行. 这个 handler 运行在 S 模式下, 从而可以完成一些比较复杂的处理.

> 当然, 在我们还未回到 S 模式的时候, 硬件也会发现有一个 SSI 在 pending, 但硬件并不会响应它. 其原因是我们已经把 SSI 代理给了 S 模式, 从而在 M 模式下, 硬件会直接忽略 SSI. (参见前面粘贴的 RISC-V 文档节选)

`timervec()` 剩下的几行代码是:
```asm
    ld a3, 16(a0)
    ld a2, 8(a0)
    ld a1, 0(a0)
    csrrw a0, mscratch, a0

    mret
```
恢复了寄存器 `a0`, `a1`, `a2`, `a3` 及 `mscratch` 的值, 然后执行 `mret` 回到 S 模式. 一旦进入 S 模式, 硬件就会检测到 pending 的 SSI 中断, 然后转而执行 SSI 中断的 handler.

> 通过阅读以上代码, 我们发现时钟中断的间隔 `interval` 不能设的太小. 比如说, 假如还没等 xv6 初始化好 S 模式的中断向量表 (即设置 `stvec` 寄存器) 就触发了一次时钟中断, 那么这个中断就没法被正确 handle. 因此, 这里取 `interval` 为 1000000, 它足够操作系统执行完所有的初始化代码.

设置完时钟中断处理程序, M 模式下需要做的全部工作都已经完成.

