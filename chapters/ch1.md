# 1. 启动操作系统

## Getting started

从仓库获取源代码后, 在源代码根目录执行 `make qemu`, 即可启动 xv6 操作系统. 系统启动后, 会自动运行一个 shell 程序 `sh`. 

看看 `Makefile` 中的相关代码: (`Makefile[159:165]`)

```makefile
QEMUOPTS = -machine virt -bios none -kernel $K/kernel -m 128M -smp $(CPUS) -nographic
QEMUOPTS += -global virtio-mmio.force-legacy=false
QEMUOPTS += -drive file=fs.img,if=none,format=raw,id=x0
QEMUOPTS += -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

qemu: $K/kernel fs.img
	$(QEMU) $(QEMUOPTS)
```

命令 `make qemu` 首先会把 `kernel/` 子目录下的源文件编译成可执行文件 `kernel/kernel`, 然后使用一系列命令行参数启动模拟器 `qemu`. 有一些参数指定了要模拟的硬件配置, 例如
* `-machine virt` 指定了模拟器的机器类型为 `virt` 
* `-bios none` 告知模拟器不启用任何 BIOS, 即由用户决定将什么内容加载到内存
* `-m 128M` 指定了我们给模拟器分配的内存大小 

> `qemu` 文档对于 `-machine` 选项的解释如下: 
>
> For QEMU's RISC-V system emulation, you must specify which board model you want to use with the `-M` or `--machine` option; there is no default.
>
> Because RISC-V systems differ so much and in fundamental ways, typically operating system or firmware images intended to run on one machine will not run at all on any other. 
>
> If you don't care about reproducing the idiosyncrasies of a particular bit of hardware, such as small amount of RAM, no PCI or other hard disk, etc., and just want to run Linux, the best option is to use the `virt` board. This is a platform which doesn't correspond to any real hardware and is designed for use in virtual machines. You'll need to compile Linux with a suitable configuration for running on the `virt` board. `virt` supports PCI, virtio, recent CPUs and large amounts of RAM. It also supports 64-bit CPUs.
>
> [https://www.qemu.org/docs/master/system/target-riscv.html](https://www.qemu.org/docs/master/system/target-riscv.html)
>

和我们这里比较相关的是 `-kernel $K/kernel` 这一参数, 它指定了模拟器所用的 OS 内核为我们刚编译得到的 `kernel/kernel`.

## Where xv6 starts

当我们启动 `qemu` 后, `qemu` 会把物理内存的起始地址设置为 `0x80000000`; 由于我们给模拟器分配了 128 MB 的内存, 我们拥有的 **物理地址空间** 为 `[0x80000000, 0x88000000)`. 

> 为什么物理内存的起始地址不是 0 呢? 这是因为, 当我们以命令行参数 `-machine virt` 启动 `qemu` 时, `qemu` 会把一些管理 I/O 设备的寄存器 **映射** 到物理地址为 `[0, 0x80000000)` 的区域. 真正的物理内存是从地址 `0x80000000` 开始的.
>
> 在 `kernel/memlayout.h` 中, 具体解释了 `qemu -machine virt` 如何进行 I/O 设备的地址映射. 参见下面的代码: (`kernel/memlayout.h[1:13]`)
> ```c
> // Physical memory layout
> 
> // qemu -machine virt is set up like this,
> // based on qemu's hw/riscv/virt.c:
> //
> // 00001000 -- boot ROM, provided by qemu
> // 02000000 -- CLINT
> // 0C000000 -- PLIC
> // 10000000 -- uart0 
> // 10001000 -- virtio disk 
> // 80000000 -- boot ROM jumps here in machine mode
> //             -kernel loads the kernel here
> // unused RAM after 80000000.
> ```

在我们提供的命令行参数下, `qemu` 开始运行之后, 会把可执行文件 `kernel/kernel` 加载到内存地址 `0x80000000` 开始的一片物理内存区域, 然后将 PC 设置为可执行文件的入口地址. `kernel/kernel` 的入口地址是在 linker script 前两行中规定的: (`kernel/kernel.ld[1:2]`)

```
OUTPUT_ARCH( "riscv" )
ENTRY( _entry )
```

也就是说, 入口地址就是符号 `_entry` 所在的地址.

> Linker script 是指示链接器 (如 `ld`) 如何工作的脚本文件. 在 Linux 上用 `gcc` 编译时, 像下面这样指定参数
> ```
> gcc -Wl,-T,<linker script> ...
> ```
> 可以让链接器 `ld` 按照自定义文件 `<linker script>` 的指示来链接. 在 Linux 上执行 `ld --verbose`, 可以输出 `ld` 默认使用的 linker script (如果用 `ld` 链接时没有 `-T` 选项, 就会按照默认 linker script 进行链接).
>
> 网站 [https://mcyoung.xyz/2021/06/01/linker-script/](https://mcyoung.xyz/2021/06/01/linker-script/) 是一个很好的 linker script 学习资源. 这里我们只是简要介绍一下: 一个 linker script (.ld 文件) 由一系列 **命令** 组成, 例如上面代码中 
> * `OUTPUT_ARCH(...)` 命令指定了输出可执行文件适配的体系结构 
> * `ENTRY(...)` 命令指定一个内存地址为可执行文件的入口地址, 也就是 `_entry` 的地址
>
> 事实上, 我们用来链接 xv6 内核的 linker script (`kernel/kernel.ld`) 是很简单的. 我们会在之后逐步解释这个文件中的内容.

符号 `_entry` 在汇编语言文件 `kernel/entry.S` 中定义. 这个文件很短, 内容如下:

```asm
        # qemu -kernel loads the kernel at 0x80000000
        # and causes each hart (i.e. CPU) to jump there.
        # kernel.ld causes the following code to
        # be placed at 0x80000000.
.section .text
.global _entry
_entry:
        # set up a stack for C.
        # stack0 is declared in start.c,
        # with a 4096-byte stack per CPU.
        # sp = stack0 + (hartid * 4096)
        la sp, stack0
        li a0, 1024*4
        csrr a1, mhartid
        addi a1, a1, 1
        mul a0, a0, a1
        add sp, sp, a0
        # jump to start() in start.c
        call start
spin:
        j spin
```

(以下, 为了方便, 我们有时候会省略目录前缀 `kernel/`, 比如直接用 `entry.S` 来指代 `kernel/entry.S`.) 

汇编器首先将 `entry.S` 文件汇编成目标文件 `entry.o`. 在 `entry.o` 中, `_entry` 是一个位于 `.text` 节的符号; 经过链接之后, `_entry` 会指向可执行文件 `kernel/kernel` 代码段中的某个位置. CPU 正是从这个位置开始执行代码. 最开始的几行代码的意义如下:

* `la sp, stack0`: 将栈指针寄存器 `sp` 初始化为 `stack0` 的地址. `stack0` 是 `start.c` 中定义的一个数组: (`kernel/start.c[10:11]`)
```c
// entry.S needs one stack per CPU.
__attribute__ ((aligned (16))) char stack0[4096 * NCPU];
``` 
其中 `NCPU = 8` 表示最大可支持的 CPU 数. 也就是说, 我们 **为每个 CPU 分配独立的 4096 字节的栈区**. 这些栈区占据一段连续的内存, 其起始地址为 `stack0`.
* 接下来 5 行代码
```asm
    li a0, 1024*4
    csrr a1, mhartid	# read register "mhartid" to register "a1"
    addi a1, a1, 1
    mul a0, a0, a1
    add sp, sp, a0
```
的意思等同于 `sp += 4096 * (mhartid + 1)`. 也就是说, 对于每个 CPU, 我们让 `sp` 指向他自己的栈顶. 第 n 个核心的栈区是左闭右开的 `4096 * [n, n+1]`.
> `csrr` 和 `csrw` 都是 RISC-V 特权指令. 它们能够读/写 CSR (Control and Status Registers). 寄存器 `mhartid` 就是一个 CSR, 它存储了 "这是第几个 CPU 核心?" (从 0 开始编号). 指令 `csrr a1, mhartid` 将寄存器 `mhartid` 的值读到 `a1`. 
>

* 接下来一行代码调用了 `start()` 函数. 它定义在 `start.c` 中. (见 `kernel/start.c[19:55]`, 我们将在下一章介绍这个函数的内容.)

正常来讲, `start()` 函数不会返回, 因此程序永远不会进入 `spin` 标记的死循环中.


## RISC-V 特权级别

RISC-V 处理器有三种 **特权级别** (privilege levels): **机器模式** (**machine mode**, M 模式), **监督模式** (**supervisor mode**, S 模式), **用户模式** (**user mode**, U 模式). 这三种特权级别是由高到低的. 

当 RISC-V 处理器启动时, **默认处于 M 模式**, 因此可以执行所有 RISC-V 指令. 在上面 `entry.S` 代码中, 有一条指令 `csrr a1, mhartid`, 它读了 CSR 寄存器 `mhartid` 的值. 这个读指令只有在 M 模式下才允许执行. 

> 如果读一下 RISC-V 特权指令集文档, 会发现 M/S 模式下可访问的 CSR 一般由 `m`/`s` 开头.
>
> [https://riscv.org/wp-content/uploads/2017/05/riscv-privileged-v1.10.pdf](https://riscv.org/wp-content/uploads/2017/05/riscv-privileged-v1.10.pdf)
>

## 如何配环境, 如何 debug 代码?

对于操作系统这样复杂的软件, 仅仅静态地阅读代码不足以掌握系统运行时的全貌. 在读代码遇到困惑时, 一个很好的自救方法就是真的上手去 **运行并调试代码**. 像 `gdb` 这样的调试器, 提供了设置断点、查看寄存器内容等功能, 能给我们理解操作系统的运行提供很大帮助.

### 环境配置

要运行或调试代码, 就不可避免的遇到配环境问题. 这里提供一种亲测有效的配环境方法 (基于 MIT 课程 6.828 官网 [https://pdos.csail.mit.edu/6.828/2023/xv6.html](https://pdos.csail.mit.edu/6.828/2023/xv6.html)), 希望能给读者带来方便.

**第一步:** 去 Docker 官网下载安装 Docker. 如果执行命令
```
docker --version
```
得到的输出类似于
```
Docker version 20.10.23, build 7155243
```
那么说明 Docker 已经安装好了.

**第二步:** 获取最新版 Ubuntu 官方镜像. 启动 Docker, 然后执行
```
docker pull ubuntu:latest
```
如果镜像获取成功, 那么执行
```
docker images
```
应该会看到镜像列表中有一个条目是 `ubuntu`, 其 tag 为 `latest`.

**第三步:** 启动一个容器. 新建一个文件 `docker-compose.yml`, 然后写入
```yml
services:
  xv6-ubuntu:
    image: ubuntu:latest
    container_name: xv6os
    ports:
      - 19999:22
    tty: true
```
文件中的 `xv6-ubuntu` 和 `xv6os` 是名字, 可以自己任意取. 另外 `ports` 一栏里的 `19999:22` 将本机的 19999 端口连向容器的 22 端口. 这样我们可以通过 `ssh` 访问容器. 端口号 19999 也可以是任选的无名端口号.

然后执行
```
docker-compose up -d
```
即可创建容器. 创建完成后, 执行
```
docker ps -a
```
应该能看到刚刚创建的容器在运行.

**第四步:** 进入容器配置环境. 执行
```
docker exec -it xv6os /bin/bash
```
在容器中开一个交互式 `bash`. (如果上一步的容器名不是取为 `xv6os`, 则这里也要相应的改变.) 

进入 `bash` 后, 可以先安装 `ssh` 服务:
```
apt update
apt install openssh-client openssh-server
```
然后安装必要的工具链:
```
apt-get install git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu 
```
至此, 我们已经可以克隆 xv6 源代码:
```
git clone https://github.com/mit-pdos/xv6-riscv.git
```
在目录 `xv6-riscv` 下执行 `make qemu`, 操作系统应该就可以正常启动.

**第五步: (可选)** 启动 `ssh` 服务. 首先还是用 `docker exec -it` 在容器中开一个 `bash`, 然后执行 `passwd` 设置一个 root 账户密码. 然后在 `/etc/ssh/sshd_config` 的末尾添加两行:
```
PermitRootLogin yes
PasswordAuthentication yes
```
(在当前环境下不能使用 `vi` 直接去编辑文件, 需要先 `cat` 出来, 复制到本机的 text editor 编辑好, 再 `echo` 进去.)

添加完成后, 运行
```
service ssh restart
```
此后在主机中使用 
```
ssh root@127.0.0.1 -p 19999
```
即可登录容器. 这里 19999 要改成上面设置的端口号.

### 如何调试

为了调试 xv6 代码, 首先要在目录 `xv6-riscv` 下执行
```
make qemu-gdb
```
事实上, QEMU 提供了 **远程调试** 的功能, 它允许我们在一个进程里运行 QEMU, 另一个进程里运行调试器, 然后通过这两个进程之间的 TCP 连接实现远程调试. 看看 `Makefile` 中关于目标 `qemu-gdb` 的代码: (`Makefile[149:154, 170:172]`)
```Makefile
# try to generate a unique GDB port
GDBPORT = $(shell expr `id -u` % 5000 + 25000)
# QEMU's gdb stub command line changed in 0.11
QEMUGDB = $(shell if $(QEMU) -help | grep -q '^-gdb'; \
	then echo "-gdb tcp::$(GDBPORT)"; \
	else echo "-s -p $(GDBPORT)"; fi)

qemu-gdb: $K/kernel .gdbinit fs.img
	@echo "*** Now run 'gdb' in another window." 1>&2
	$(QEMU) $(QEMUOPTS) -S $(QEMUGDB)
```
会发现我们向 `qemu` 提供了两个命令行参数:
* `-gdb tcp::$(GDBPORT)`: 它的含义是, 使用本地 `gdb` 向 `localhost:<GDBPORT>` 建立一个 TCP 连接, 就可以调试 QEMU 模拟的代码.
* `-S`: 告诉 QEMU 在我们输入调试指令之前不要启动程序, 这样我们就可以调试操作系统刚启动时执行的那些代码.

运行 `make qemu-gdb` 后, 会把端口号 `GDBPORT` 打印在屏幕上. 我们接着另开一个终端窗口, 在里面运行
```
gdb-multiarch kernel/kernel
```
启动 `gdb` 调试器. `gdb` 启动后, 我们先执行
```
target remote localhost:<GDBPORT>
```
使 `gdb` 和 QEMU 建立连接. 这里 `<GDBPORT>` 就是刚才打印出来的端口号. 接着就可以像正常使用 `gdb` 那样设断点、单步调试了.


