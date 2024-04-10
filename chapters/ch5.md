# 5. Hello, World!

在本章中, 我们正式开始研究 `main()` 函数的代码: (`kernel/main.c[10:45]`)
```c
void main()
{
  if(cpuid() == 0){
    consoleinit();
    printfinit();
    printf("\n");
    printf("xv6 kernel is booting\n");
    printf("\n");
    kinit();         // physical page allocator
    kvminit();       // create kernel page table
    kvminithart();   // turn on paging
    procinit();      // process table
    trapinit();      // trap vectors
    trapinithart();  // install kernel trap vector
    plicinit();      // set up interrupt controller
    plicinithart();  // ask PLIC for device interrupts
    binit();         // buffer cache
    iinit();         // inode table
    fileinit();      // file table
    virtio_disk_init(); // emulated hard disk
    userinit();      // first user process
    __sync_synchronize();
    started = 1;
  } else {
    while(started == 0)
      ;
    __sync_synchronize();
    printf("hart %d starting\n", cpuid());
    kvminithart();    // turn on paging
    trapinithart();   // install kernel trap vector
    plicinithart();   // ask PLIC for device interrupts
  }

  scheduler();        
}
```
初看起来, 一大堆函数调用让人眼花缭乱. 不过稍微观察一下, 就会发现 `main()` 函数的结构如下:
```c
void main()
{
  if(cpuid() == 0){
    /* 
     * Tons of initialization ......
     */
    __sync_synchronize();
    started = 1;
  } else {
    while(started == 0)
      ;
    __sync_synchronize();
    printf("hart %d starting\n", cpuid());
    /*
     * Some more initialization ......
     */
  }

  scheduler();        
}
```
也就是说, xv6 维护了一个 flag: (`kernel/main.c[7]`)
```c
volatile static int started = 0;
```
当 `started == 0` 时, 除了核心号为 0 的 CPU 在执行, 其余 CPU 都在空转. 直到核心号为 0 的 CPU 把该做的事情都做完了, 它才把 `started` 设为 1, 让其他核心启动并完成它们各自的初始化工作.

> 注意这里也有两次 `__sync_synchronize()` 调用:
> * 第一个 `__sync_synchronize()` 保证了 `started = 1;` 语句严格地发生在核心 0 执行完所有初始化工作之后
> * 第二个 `__sync_synchronize()` 保证了其他核心的初始化工作严格地发生在退出 `while` 循环之后
> 
> 这两条同步指令都是为了避免其他 CPU 过早地开始初始化而破坏共享数据结构的情况.
>

这一章我们来阅读下面五行代码的实现: (`kernel/main.c[14:18]`)
```c
    consoleinit();
    printfinit();
    printf("\n");
    printf("xv6 kernel is booting\n");
    printf("\n");
```
正如我们的标题所暗示的, 本章的目标是让内核能够向控制台输出字符, 从而我们能在屏幕上打印出 `"Hello World!"`.

> xv6 内核中定义了 `printf()` 函数用于让内核向控制台输出格式化字符串. 虽然它与 C 的标准库函数 `printf()` 同名, 但两者的实现是完全不同的. 事实上, xv6 内核中的 `printf()` 远比 C 标准库中的版本简单; 它仅仅为内核代码提供最基础的输出功能. 对于用户程序, xv6 提供了另外的系统调用 `write()`, 其实现也不同于这里的 `printf()`.
>

## UART 硬件

函数 `consoleinit()` 的代码如下: (`kernel/console.c[181:192]`)

```c
void consoleinit(void)
{
  initlock(&cons.lock, "cons");

  uartinit();

  // connect read and write system calls
  // to consoleread and consolewrite.
  devsw[CONSOLE].read = consoleread;
  devsw[CONSOLE].write = consolewrite;
}
```

我们这一章的目标是实现内核中的 `printf()` 函数. 对于这个目的来说, 真正有用的只有第二行代码: 对 `uartinit()` 的调用.

> `consoleinit()` 的剩余部分, 我们将在之后回过头来讨论. 简要来说, `consoleinit()` 的第一行代码初始化了一个互斥锁 `cons.lock`, 用于保护控制台缓冲区; 最后两行代码将 `read()` 和 `write()` 系统调用分别绑定到 `consoleread()` 和 `consolewrite()`. 

文件 `uart.c` 中的一系列函数提供了 **UART 串口硬件** 的驱动程序. 特别地, `uartinit()` 函数初始化了 UART 硬件. UART 硬件是一块 16550 芯片; `qemu` 模拟了 UART 硬件, 以实现从键盘读取输入及向屏幕输出. 

UART 硬件提供了一系列寄存器, 来让软件控制它的行为. 这些寄存器都是 1 字节长的, 并且都被 **内存映射** 到了物理地址空间中 `[0x10000000, 0x10001000)` 的范围 (参考第 1 章中列出的 `memlayout.h` 中的注释).

为了方便操控这些寄存器, 我们定义以下宏: (`kernel/memlayout.h[21]`, `kernel/uart.c[16, 38:39]`)
```c
#define UART0 0x10000000L
#define Reg(reg) ((volatile unsigned char *)(UART0 + reg))

#define ReadReg(reg) (*(Reg(reg)))
#define WriteReg(reg, v) (*(Reg(reg)) = (v))
```
其中 `UART0` 是所有 UART 控制寄存器在物理内存中的映像的起始地址. `ReadReg(reg)` 和 `WriteReg(reg, v)` 分别用来读写以 `UART0` 为基址, `reg` 为偏移量的 UART 控制寄存器.

在 `kernel/uart.c[22:36]` 中, 列出了一些 UART 控制寄存器的信息 (包括它们的名字, 在物理内存中的偏移量, 以及一些设定值). 这些信息都可以从网站 [http://byterunner.com/16550.html](http://byterunner.com/16550.html) 中找到. 由于这些寄存器的细节相当繁琐, 且没有普适的意义, 我们不在这里一一详述, 而是在讲解代码的过程中提供必要的背景知识.

### `uartinit()` 函数

现在我们可以来看 `uartinit()` 函数的实现. 该函数定义在 `kernel/uart.c[52:78]`:
```c
void uartinit(void)
{
  // disable interrupts.
  WriteReg(IER, 0x00);

  // special mode to set baud rate.
  WriteReg(LCR, LCR_BAUD_LATCH);

  // LSB for baud rate of 38.4K.
  WriteReg(0, 0x03);

  // MSB for baud rate of 38.4K.
  WriteReg(1, 0x00);

  // leave set-baud mode,
  // and set word length to 8 bits, no parity.
  WriteReg(LCR, LCR_EIGHT_BITS);

  // reset and enable FIFOs.
  WriteReg(FCR, FCR_FIFO_ENABLE | FCR_FIFO_CLEAR);

  // enable transmit and receive interrupts.
  WriteReg(IER, IER_TX_ENABLE | IER_RX_ENABLE);

  initlock(&uart_tx_lock, "uart");
}
```
`uartinit()` 函数操纵了一系列 UART 控制寄存器:
* `IER` 寄存器 (偏移量 1 字节) 是 UART 硬件的中断开关. 它的低 4 位分别控制四种中断的开启 (1) 或关闭 (0). 特别地, 
  + 第 0 位 (最低位) 控制 "receiver ready interrupt", 即收到一个输入字节时是否产生中断
  + 第 1 位控制 "transmitter ready interrupt", 即输出缓冲区为空 (从而允许输出新的字节) 时是否产生中断

  所以, 第一行代码 `WriteReg(IER, 0x00);` 禁用了 UART 硬件的所有中断. 
* 接下来三行代码
  ```c
  WriteReg(LCR, LCR_BAUD_LATCH);  // LCR_BAUD_LATCH = (1 << 7);
  WriteReg(0, 0x03);
  WriteReg(1, 0x00);
  ```
  的作用是设置波特率. 
  
> 读者可能会有些奇怪: 如果查看 `uart.c[22:36]` 中的宏定义, 或者在线文档 [http://byterunner.com/16550.html](http://byterunner.com/16550.html), 会发现从 `UART0` 出发偏移量为 0 的字节对应于 `RHR`/`THR` 寄存器, 偏移量为 1 的字节对应于 `IER` 寄存器. 那么这里
> ```c
> WriteReg(0, 0x03);
> WriteReg(1, 0x00);
> ```
> 是不是覆写了这些寄存器呢? 其实不是这样的. 问题的关键在于 `LCR` 寄存器 (偏移量 3 字节). 当 `LCR` 的最高位 (第 7 位) 等于 1 时, 说明 UART 硬件进入 "divisor latch enable" 模式. 此时写偏移量 0、1 字节处的寄存器, 分别代表着修改 "LSB of divisor latch" 和 "MSB of divisor latch". 也就是说, 这两个字节现在不再代表 `RHR`/`THR` 和 `IER` 寄存器, 而是在控制 UART 内部的波特率生成器.
>

* 下一行代码 `WriteReg(LCR, LCR_EIGHT_BITS);` 把 `LCR` 的最高位设为 0 (意味着退出 "divisor latch enable" 模式), 并且规定一次读/写的字长为 8 位 (1 字节).
* 接下来的代码
  ```c
  // FCR_FIFO_ENABLE = (1 << 0);
  // FCR_FIFO_CLEAR = (3 << 1);
  WriteReg(FCR, FCR_FIFO_ENABLE | FCR_FIFO_CLEAR);
  ```
  启用 UART 硬件中的输入输出队列 (将 `FCR` 的第 0 位设 1), 并且重置输入输出队列 (将 `FCR` 的第 1、2 位设 1).
* 最后, 
  ```c
  // IER_TX_ENABLE = (1 << 1);
  // IER_RX_ENABLE = (1 << 0);
  WriteReg(IER, IER_TX_ENABLE | IER_RX_ENABLE);
  ```
  开启 receiver ready interrupt 和 transmitter ready interrupt. (当然, 在我们设置好中断处理程序并打开 `sstatus` 中的中断开关之前, 我们还没法响应这两个中断.)

做完这些之后, `uartinit()` 还初始化了一个互斥锁 `uart_tx_lock`, 用来保护 UART 的输出缓冲区.

### 最简单的输出: `uartputc_sync()` 函数

现在, UART 硬件已经初始化完毕; 是时候看看它能不能做点什么了. (比如, 能不能打印出 `"Hello World!"` 呢?) 下面我们就来看看怎样操纵 UART 硬件, 使之往屏幕上打印字符.

为实现最基本的输出功能, `kernel/uart.c[111:127]` 定义了 `uartputc_sync()` 函数:
```c
void uartputc_sync(int c)
{
  push_off();

  if(panicked){
    for(;;)
      ;
  }

  // wait for Transmit Holding Empty to be set in LSR.
  while((ReadReg(LSR) & LSR_TX_IDLE) == 0)
    ;
  WriteReg(THR, c);

  pop_off();
}
```
这个函数相当简单:
* 打印之前, 首先用 `push_off()` 关中断.
> 为什么要关中断呢? 原因在于, 下面会看到, `uartputc_sync()` 不对 UART 输出缓冲区做任何并发保护措施. 所以如果函数 `uartputc_sync()` 被中断打断, 且该中断的处理程序也要调用 `uartputc_sync()`, 那么输出就会混乱.
* 检查 `panicked` 字段. 这个字段定义在 `kernel/printf.c[18]`:
  ```c
  volatile int panicked = 0;
  ```
  如果 `panicked` 非零, 那么意味着操作系统发生了意料之外的严重错误. 此时 `uartputc_sync()` 函数会进入死循环.
* 循环地检查 `LSR` 寄存器的第 5 位 (用掩码 `LSR_TX_IDLE` 获得) 是否为 1. 当这一位为 1 时, 说明 UART 的输出缓冲区变空了, 从而可以接收新的输出字符.
* 一旦发现 `LSR` 的第 5 位为 1, 就往 `THR` 写入要输出的字符 `c`.
> 寄存器 `THR`/`RHR` 都被内存映射到从地址 `UART0` 出发, 偏移量为 0 的字节. 当软件要向屏幕输出字符时, 就把字符写到 `THR` 寄存器中; 当软件要从 UART 读取输入时, 就读 `RHR` 寄存器的值. 
* 恢复中断状态.

可能会有一些惊讶: `THR` 寄存器是被多个处理器 **共享** 的, 但是 `uartputc_sync()` 函数竟然没有对它做任何保护措施就直接写入了. 假如有多个 CPU 同时调用 `uartputc_sync()` 的话, 肯定会发生输出混乱. 为了避免这种情况发生, 我们要求 `uartputc_sync()` 的调用者承担保护 `THR` 寄存器的责任.

### 读取输入: `uartgetc()` 函数

类似地, 在 `kernel/uart.c[161:170]` 中定义了 `uartgetc()` 函数, 用于从 UART 硬件读取字符输入:
```c
int uartgetc(void)
{
  if(ReadReg(LSR) & 0x01){
    // input data is ready.
    return ReadReg(RHR);
  } else {
    return -1;
  }
}
```
这个函数检查 `LSR` 寄存器的第 0 位是否为 1. 当这一位为 1 时, 说明输入的字符已经就绪, 可以通过 UART 硬件读取它了. 在这种情况下, `uartgetc()` 函数就从 `RHR` 寄存器读取该字符, 并返回. 如果输入字符尚未就绪 (`LSR` 的第 0 位为 0), 那么 `uartgetc()` 函数就返回 -1.

> 对于所有的 CPU 来说, `LSR` 和 `RHR` 寄存器都是只读的. 因此 `uartgetc()` 函数是并发安全的.

## `printf()` 函数

现在我们回到 `main()` 函数的代码, 看看 xv6 内核如何利用 UART 硬件提供的服务, 实现简单的格式化字符串输出, 即 `printf()` 函数. 这对应于下面四行代码:
```c
    printfinit();
    printf("\n");
    printf("xv6 kernel is booting\n");
    printf("\n");
```

`printf()` 函数在底层调用了 `uartputc_sync()`. 根据我们上面的讨论, `printf()` 有责任对 UART 硬件的 `THR` 寄存器作并发保护. 为此, 在 `kernel/printf.c[21:24]` 中定义了下列结构体:
```c
static struct {
  struct spinlock lock;
  int locking;
} pr;
```
然后使用 `printfinit()` 函数初始化这个结构体: (`kernel/printf.c[130:135]`)
```c
void printfinit(void)
{
  initlock(&pr.lock, "pr");
  pr.locking = 1;
}
```
`printf()` 的实现保证: 当 `pr.locking == 1` 时, 都会首先获取互斥锁 `pr.lock`, 然后才调用 `uartputc_sync()`. 这样, 两个并发的 `printf()` 调用不会导致输出的交错. 

下面我们来看看 `printf()` 的定义: (`kernel/printf.c[63:116]`)
```c
void printf(char *fmt, ...)
{
  va_list ap;
  int i, c, locking;
  char *s;

  locking = pr.locking;
  if(locking) acquire(&pr.lock);

  if (fmt == 0) panic("null fmt");

  va_start(ap, fmt);
  for(i = 0; (c = fmt[i] & 0xff) != 0; i++){
    if(c != '%')  { 
      consputc(c); 
      continue; 
    }
    c = fmt[++i] & 0xff;
    if(c == 0)
      break;
    switch (c) {
      case 'd':  printint(va_arg(ap, int), 10, 1); break;
      case 'x':  printint(va_arg(ap, int), 16, 1); break;
      case 'p':  printptr(va_arg(ap, uint64)); break;
      case 's':
        if((s = va_arg(ap, char*)) == 0)
          s = "(null)";
        for(; *s; s++)
          consputc(*s);
        break;
      case '%':  consputc('%'); break;
      default:
        // Print unknown % sequence to draw attention.
        consputc('%');
        consputc(c);
        break;
    }
  }
  va_end(ap);

  if(locking)
    release(&pr.lock);
}
```
这段代码很容易理解:
* 首先检查 `pr.locking` 的值. 如果为 1, 那么就获取互斥锁 `pr.lock`, 并在全部输出结束后释放锁.
* 逐字符解析格式串 `fmt`. 对于 `%d`, `%x`, `%p` 或 `%s`, 从参数列表中取出相应参数并选择合适的打印方法.
* 对每个字符调用 `consputc()` 函数. 这个函数定义在 `kernel/console.c[33:42]`:
  ```c
  void consputc(int c)
  {
    if(c == BACKSPACE){ // BACKSPACE = 0x100;
      // if the user typed backspace, overwrite with a space.
      uartputc_sync('\b'); uartputc_sync(' '); uartputc_sync('\b');
    } else {
      uartputc_sync(c);
    }
  }
  ```
  它就是对 `uartputc_sync()` 函数的简单包装. (现在无需关心 `c == BACKSPACE` 的情况, 因为在 `printf()` 中, 给 `consputc()` 传递的参数不会超过 `0xff`.)

至此, xv6 内核已经可以打印任意字符串. 为了庆祝这一标志性事件, xv6 内核会输出
```
xv6 kernel is booting
```

## `panic()` 函数

在 `kernel/printf.c[118:128]` 中, 还定义了一个函数 `panic()`:
```c
void panic(char *s)
{
  pr.locking = 0;
  printf("panic: ");
  printf(s);
  printf("\n");
  panicked = 1; // freeze uart output from other CPUs
  for(;;)
    ;
}
```
当 xv6 内核出现某种意料之外的错误时 (通常是因为写操作系统的人没有正确实现某些逻辑), 会调用 `panic()`. 这个函数会输出一条错误信息, 然后把全局变量 `panicked` 赋值为 1; 此后, 函数陷入死循环. 

> 全局变量 `panicked` 的作用是: 当底层的输出函数 `uartputc_sync()` 或 `uartputc()` 检测到 `panicked == 1` 时, 会停止输出并陷入死循环. (此时也没必要用互斥锁保护 UART 输出缓冲区了, 因此 `panic()` 还会设置 `pr.locking = 0`.) 
>
> 换言之, 一旦 `panicked == 1`, 内核实际上已经不再工作.
>


