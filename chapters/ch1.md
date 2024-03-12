# 1. 启动操作系统

从仓库获取源代码后, 在源代码根目录执行 `make qemu`, 即可启动操作系统. 看看 `Makefile` 中的相关代码: (`Makefile[159:165]`)

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
* `-bios none` 表示不启用任何 BIOS, 
* `-m 128M` 指定了模拟器的内存大小 

和我们这里比较相关的是 `-kernel $K/kernel` 这一参数, 它指定了模拟器所用的 OS 内核为我们刚编译得到的 `kernel/kernel`.
