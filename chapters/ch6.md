# 6. 物理内存管理

在本章中, 我们来研究 xv6 内核如何管理物理内存. 这部分代码全部位于源文件 `kernel/kalloc.c` 中. 虽然本章的代码比较简单, 但它为接下来两章的内容 (**虚拟内存**) 提供了基础设施.

## 数据结构

回忆一下, `qemu` 模拟器把 (大小为 128 MB 的) 物理地址范围 `[0x80000000, 0x88000000)` 作为可用的物理内存. 操作系统启动时, 可执行文件 `kernel/kernel` 中的代码和数据会被加载到起始物理地址为 `0x80000000` 的物理内存中. 

很容易想见, xv6 自身的代码和数据并不需要占据 128 MB 那么大的内存 (事实上, 查看一下编译链接后 `kernel/kernel` 的大小就知道了). 那么我们如何管理剩下来的这些 **空闲物理内存** 呢?

xv6 的做法如下:
* 将空闲物理内存划分成一系列 4096 字节大小的 **物理页**. (每个物理页都是 4096 字节对齐的.)
* 将物理页组织成 **链表**, 做法是在每个物理页的前 8 个字节记录下一页的起始地址.

为实现这种管理方式, xv6 维护了一个数据结构 `kmem`: (`kernel/kalloc.c[21:24]`)
```c
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem;
```
其中结构体 `run` 代表一个链表结点: (`kernel/kalloc.c[17:19]`)
```c
struct run {
  struct run *next;
};
```

在 `kmem` 中:
* `kmem.freelist` 是链表中第一个结点的起始地址 (链表为空时, `kmem.freelist == 0`)
* `kmem.lock` 是一个互斥锁, 保护共享数据 `kmem.freelist`

## 物理页的分配与释放

### `kalloc()` 函数

通过调用 `kalloc()`, 可以申请获得一个物理页: (`kernel/kalloc.c[68:82]`)
```c
void * kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r)
    kmem.freelist = r->next;
  release(&kmem.lock);

  if(r) // PGSIZE = 4096;
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```
这个函数干的事情如下:
* 首先, 取得互斥锁 `kmem.lock` 以访问共享数据 `kmem.freelist`.
* 如果链表非空, 取出链表中的第一个物理页 (以 `r` 为首地址), 并将链表头指针 `kmem.freelist` 指向 `r` 的下一个物理页. 这就相当于把空闲页链表中的第一个页拿出来使用.
* 释放互斥锁, 用重复字节填充刚获取的物理页, 并返回该页首地址 (如果成功获取的话).

> 如果已经没有空闲的物理页了, 那么 `kalloc()` 将返回零指针.

### `kfree()` 函数

相应地, 调用 `kfree()` 可以把一个物理页 "归还" 给空闲页链表: (`kernel/kalloc.c[46:63]`)
```c
void kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}
```
这个函数的作用是:
* 首先检查物理页的起始地址 `pa` 是否合法:
    + `pa` 需是 4096 字节对齐的
    + `pa` 不允许低于内核代码、数据段的最高地址 `end` (以防止内核自身的代码、数据被覆盖)
    + `pa` 不允许高于物理内存的最高地址 `PHYSTOP` (它等于 `0x88000000`)
    
    如果地址不合法, 那么内核就 `panic()`.
> `end` 指针出现在 `kernel/kalloc.c[14]`:
> ```c
> extern char end[]; // first address after kernel.
> ```
> 会发现它是一个仅有声明而未定义的外部符号. 事实上, 这个符号的定义出现在 linker script `kernel.ld` 中. 查看一下 `kernel.ld` 的内容, 会发现它的结构为
> ```
> SECTIONS
> {
>   . = 0x80000000;
> 
>   .text : {
>     /* ... */
>     PROVIDE(etext = .);
>   }
>   .rodata : { /* ... */ }
>   .data : { /* ... */ }
>   .bss : { /* ... */ }
> 
>   PROVIDE(end = .);
> }
> ```
> 如果读者在[第 1 章](ch1.md)中 (例如, 通过网站 [https://mcyoung.xyz/2021/06/01/linker-script/](https://mcyoung.xyz/2021/06/01/linker-script/)) 学习过 linker script, 会了解到:
> * `SECTIONS { ... }` 指令指定了链接器的输出文件中含有哪些 sections
> * 在 `SECTIONS { ... }` 指令块中, 
>   + `.` 是一个随着链接过程不断增长的指针. 在链接过程中, 链接器总是把 `.` 作为接下来生成的内容的起始地址. 所以, 上面 linker script 中的语句 `. = 0x80000000;` 表示: 把 `0x80000000` 作为 xv6 内核 `.text` 段的起始地址.
>   + `PROVIDE` 语句可以定义新的符号. 例如 `PROVIDE(end = .);` 的意思就是说, 假如链接器的输入中 (是一系列可重定位目标文件) 没有定义符号 `end`, 那么就新建一个符号 `end`, 其地址为当前位置下 `.` 的地址.
>
> 也就是说, `kernel/kalloc.c[14]` 中声明的 `end` 数组, 直到链接的那一刻才获得定义: `end` 的首地址就是 xv6 内核中 `.text`, `.rodata`, `.data` 和 `.bss` 节全部结束之后, 对应的 `.` 的地址. 当 xv6 被加载到物理内存后, `end` 的含义就是 xv6 自身代码、数据段所占据的最高物理地址加 1.
>

* 如果物理页起始地址合法, 就将物理页填满重复字节.
* 取得互斥锁 `kmem.lock` 以访问共享数据 `kmem.freelist`.
* 将待释放的物理页附加到链表头部, 并令头指针 `kmem.freelist` 指向这个新加入链表的页.
* 释放互斥锁 `kmem.lock`.

> 可能有人会注意到: 在 `kalloc()` 和 `kfree()` 中, `kmem.lock` 只保护了 `kmem.freelist`, 而不保护申请到的物理页 `r` (对于 `kalloc()`) 或待释放的物理页 `pa` (对于 `kfree()`). 
>
> 因此, 设想这样的情况: 一个 CPU 核心 A 通过 `kalloc()` 申请到了一个物理页 `p`; 过了一段时间 (此时核心 A 已经往 `p` 里写了一些数据了), 另一个 CPU 核心 B somehow 的获得了 `p` 的起始地址, 然后调用 `kfree()` 把它释放掉了. 此时, 对于 A 来说, 它往物理页 `p` 里写的数据就全被 B 破坏掉了; 更严重的是, A 仍然认为自己占有对物理页 `p` 的访问权, 但实际上页 `p` 已经被归还到空闲页链表中去了.
>
> 为了避免上述情况, 要求每个 `kalloc()` 和 `kfree()` 的调用者遵守如下君子协定: **谁 `kalloc()`, 谁 `kfree()`**. 也就是说, 一个核心只许 `kfree()` 自己之前通过 `kalloc()` 申请到的物理页, 而不许 `kfree()` 别人申请到的物理页.
>
> 只有初始化函数 `kinit()` 可以豁免遵守上述协定.
>

在 `kernel/kalloc.c[33:40]` 中, 还定义了一个函数 `freerange()`:
```c
void freerange(void *pa_start, void *pa_end)
{
  char *p;
  // PGROUNDUP(x) = (x + 4096 - 1) & ~(4096 - 1);
  // finds minimum y such that:
  //        y >= x && y % 4096 == 0
  p = (char*)PGROUNDUP((uint64)pa_start);
  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE)
    kfree(p);
}
```
它可以一次性释放完全包含在物理地址范围 `[pa_start, pa_end)` 内的多个物理页.

## `kmem` 的初始化

在 `main()` 函数中, 我们使用下列代码初始化 `kmem`: (`kernel/main.c[19]`)
```c
    kinit();         // physical page allocator
```
`kinit()` 的定义为: (`kernel/kalloc.c[26:31]`)
```c
void kinit()
{
  initlock(&kmem.lock, "kmem");
  freerange(end, (void*)PHYSTOP);
}
```
它初始化互斥锁 `kmem.lock`, 并且把地址范围 `[end, PHYSTOP)` 内包含的全部物理页加入空闲页链表. 这样, 当我们需要更多内存时, 就可以使用上面所述的 `kalloc()` 函数获得新的物理页了.

> 参见上面的介绍, `end` 是 xv6 内核代码、数据段在物理内存中的终点.

