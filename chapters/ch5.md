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
