# 附录

## 资源链接

* **RISC-V 用户级 ISA 手册:** [https://riscv.org/wp-content/uploads/2017/05/riscv-spec-v2.2.pdf](https://riscv.org/wp-content/uploads/2017/05/riscv-spec-v2.2.pdf)
* **RISC-V 特权级 ISA 手册:** [https://riscv.org/wp-content/uploads/2017/05/riscv-privileged-v1.10.pdf](https://riscv.org/wp-content/uploads/2017/05/riscv-privileged-v1.10.pdf)
* **RISC-V PLIC 手册:** [https://github.com/riscv/riscv-plic-spec](https://github.com/riscv/riscv-plic-spec)
* **UART 硬件编程接口:** [http://byterunner.com/16550.html](http://byterunner.com/16550.html)
* **VIRTIO 手册:** [https://docs.oasis-open.org/virtio/virtio/v1.1/virtio-v1.1.pdf](https://docs.oasis-open.org/virtio/virtio/v1.1/virtio-v1.1.pdf)

## RISC-V 控制与状态寄存器 (CSR) 简表

| CSR 名称 | 功能 |
| :---: | :---|
| `mhartid` | 仅在 M 模式可见. 存储核心号. |
| `mstatus`/`sstatus` | 分别在 M/S 模式可见. 存储一系列有关机器状态的信息, 如 <ul><li>**MPP/SPP:** 陷入 M/S 模式之前的特权级.</li><li>**MIE/SIE/UIE:** M/S/U 模式的全局中断开关.</li><li>**MPIE/SPIE/UPIE:** 陷入 M/S/U 模式之前的 MIE/SIE/UIE 值.</li></ul>`sstatus` 是 `mstatus` 的真子集.|
| `medeleg`/`mideleg` | 仅在 M 模式可见. 当 `medeleg`/`mideleg` 的第 i 位为 1 时, 表示将中断码为 i 的内中断/外中断代理到 S 模式处理. |
| `mie`/`sie` | 分别在 M/S 模式可见. 当 `mie`/`sie` 的第 i 位为 1 时, 表示在 M/S 模式启用中断码为 i 的外中断.|
| `mtvec`/`stvec` | 分别在 M/S 模式可见. 存储中断向量表基址. 具体地, 设处理器处于 M/S 模式, <ul><li>如果 `mtvec`/`stvec` 低 2 位为 `00`, 那么无论发生什么中断, 硬件都把 `pc` 设为 `mtvec`/`stvec`. 该地址是 4 字节对齐的.</li><li>如果 `mtvec`/`stvec` 低 2 位为 `01`, 那么根据中断的种类, 有<ul><li>如果是内中断, 把 `pc` 设为 `mtvec - 1`.</li><li>如果是外中断, 且中断码为 `i`, 把 `pc` 设为 `mtvec - 1 + (4 * i)`.</li></ul>无论何种情况, 接下来开始执行的地址都是 4 字节对齐的.</li></ul>
| `mcause`/`scause` | 分别在 M/S 模式可见. 发生中断时, 硬件写该寄存器, 以向软件反馈中断产生的原因. <ul><li>当 `mcause`/`scause` 最高位为 1 时表示外中断, 为 0 时表示内中断.</li><li>除最高位外的剩余位中, 存储中断码.</li></ul> |
| `mip`/`sip` | 分别在 M/S 模式可见. 当 `mip`/`sip` 的第 i 位为 1 时, 表示在 M/S 模式有一个中断码为 i 的外中断待处理 (pending).|
| `mepc`/`sepc` | 分别在 M/S 模式可见. 存储中断发生时, CPU 刚执行的那条指令的首地址. 如果启用了地址翻译, 那么存储的是虚拟地址.|
| `mtval`/`stval` | 分别在 M/S 模式可见. 存储内中断的错误码. 例如, 如果某条指令发生访存异常 (如缺页), 则把引起异常的内存地址保存到 `mtval`/`stval`. <br>如果对某类中断未定义错误码, 则 `mtval`/`stval` 被设置为零.|
| `mscratch`/`sscratch` | 分别在 M/S 模式可见. 存储中断处理过程中的临时值.|
| `satp` | 在 S 模式可见. 控制地址翻译模式. 在 RV64 指令集架构下, <ul><li>`satp` 的前 4 位为全零时, 表示不启用地址翻译, 物理地址 = 虚拟地址.</li><li>`satp` 的前 4 位为 `1000`/`1001` 时, 表示启用 Sv39/Sv48 地址翻译机制, 且 `satp` 的低 44 位存储一级页表物理页号.</li></ul>



