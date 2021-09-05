### 段寄存器
在 [分段](/todo) 机制中，一个段寄存器就是一个内存区域的描述。但是长模式下，分段机制被进一步弱化。通过非空 selector 进行内存访问时，基址和界限字段将被忽略，不检查越界，只检查地址是否合规。

#### CS
长模式下，代码段寄存器 CS 的基址和界限字段将被忽略，但是属性字段仍然保留。

#### DS、SS 和 ES
数据段寄存器 DS、堆栈段寄存器 SS 和附加数据段寄存器 ES 不再使用，基址、界限和属性字段都会被忽略。

#### FS 和 GS
长模式下，FS 和 GS 是唯二能体现分段思想的段寄存器，它们的基址是有意义的。

这两个段的具体作用由 OS 决定。GDT 中没有它们的 descriptor，它们的 selector 可以设成 null。它们的基址通过特定的 MSR 寄存器设置。具体来说，FS 的基址由 [IA32_FS_BASE](#ia32_fs_base) 指定，GS 的基址由 [IA32_GS_BASE](#ia32_gs_base) 或 [IA32_KERNEL_GS_BASE](#ia32_kernel_gs_base) 指定。

一般来说，FS 用作 TLS。GS 用来放 per-cpu 数据，[Canary](/todo) 从这里取。

### MSR 寄存器

#### IA32_FS_BASE
存放 FS 基址。

#### IA32_GS_BASE
用户 GS 基址 GSBase。该寄存器实际上维护了一个在用户空间和内核空间来回切换的指针。在 ring3 下，该指针指向用户 GS，用不到的话就设为 0；在 ring0 下，该指针指向内核 GS，用于存放 per-cpu 数据。切换通过 swapgs 指令进行，该指令会交换 IA32_GS_BASE 和 IA32_KERNEL_GS_BASE 两个寄存器的值。

#### IA32_KERNEL_GS_BASE
内核 per-cpu 数据区域基址 KernelGSBase。该寄存器配合 swapgs 指令使用。进入 ring0 时，执行一次 swapgs，将 IA32_GS_BASE 设为 KernelGSBase，同时保存了其旧值 GSBase。返回 ring3 时，再执行一次 swapgs，还原。

如果内核代码要使用 per-cpu 数据，必须保证从进入 ring0 起，swapgs 的执行次数为奇数。否则，GS 段基址就不是 KernelGSBase，而是 GSBase。