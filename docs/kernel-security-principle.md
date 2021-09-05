# 内核安全机制设计思路
参见 [Kernel Self-Protection](https://www.kernel.org/doc/html/latest/security/self-protection.html)。其对威胁的假设很强：普通用户能任意读写内核内存。理想的安全机制：有效，默认开启，不用开发者做取舍，不影响性能，不妨碍内核调试，测试充分。

## 降低攻击面
### 内核内存
  1. 区分代码（可执行区域）和数据（可写区域），即代码不可写，数据不可执行。这里的代码包括内核和内核模块，也包括 JIT 区域等。这里的问题是，如果要支持断点等机制，就做不到可执行区域始终不可写。折中方案是 **临时** 授予写权限，写入数据（如断点指令）后立即撤销权限。总之基本思路，维持可写区域尽量小，只读区域尽量大。
  2. 敏感数据结构，尤其是涉及到函数指针的，尽量加 const 丢到 .rodata 节。对于在 __init 期间初始化但之后不再更新的变量，引入 [post-init read-only](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c74ba8b3480da6ddaea17df2263ec09b869ac496) 机制，支持 [__ro_after_init](https://elixir.bootlin.com/linux/v5.14/source/include/linux/cache.h#L37) 属性。标记了该属性的变量会被丢到 .data..ro_after_init 节，初始化完成后将被设为只读。另外，对于 GDT 这种只是偶尔需要更新的结构，考虑引入新机制，理想效果是「只有负责数据更新的 CPU 能获得临时写权限，且该写操作不可被中断」。
  3. 隔离内核内存和用户内存，内核不得执行用户内存。x86 的 [SMEP/SMAP](../todo) 和 ARM 的 PXN/PAN 机制即是。

### 系统调用
一是编译期间去除不必要的系统调用，比如关掉 CONFIG_COMPAT 选项，不过适用场景不多；二是借助如 [Seccomp BPF](../todo) 机制，限制进程可使用的系统调用；三是避免 BPF，perf 等敏感接口的滥用，只允许少量可信任进程访问。

### 内核模块
加载原则上由 root 用户进行。如果需要更高的安全性，可以考虑对模块签名，或者干脆禁用模块加载。禁用方式是置位内核变量 [modules_disabled](https://elixir.bootlin.com/linux/v5.14/source/kernel/sysctl.c#L2072)，这可以通过配置 /etc/sysctl.conf 在引导期间进行，也可以在系统启动且手动加载完所需模块后，通过 sysctl 设置。该变量一旦置 1 就不能变回 0。

> 问题：内核默认开启 CONFIG_STRICT_KERNEL_RWX 选项，不关掉也能用 gdb 软件断点，how？

## 内存完整性保护
针对常见内存漏洞进行缓解：

- 栈缓冲区溢出：使用 [Canary](../todo)；使用 [Shadow Stack](../todo)。
- 栈溢出：内核栈溢出可覆写 thread_info，考虑加 [guard page](../todo)。
- [「堆」利用](../todo)：在 [内核内存分配器](../todo) 中进行检查。
- 整数溢出：加检查，尤其要注意计数器和各处 size 的计算。

注意，这里的栈指 [内核栈](../todo)。另外严格来说，内核空间没有堆，这里用「堆」指代分配器动态分配的那部分内核内存，参见 [动态内存分配](../todo)。

## 引入随机性
- 依赖秘密值的机制，如 Canary，秘密值需要足够随机且避免复用。对于 BPF 这种允许即时编译的机制，内核最终执行的指令受用户空间影响，要考虑加入随机性。
- [KASLR](../todo)。代码、模块、栈和「堆」的基地址随机偏移。在用户空间，栈的随机化指不同进程栈基址不同，但对内核栈来说，要做到每次 syscall 栈基址都不同。其他敏感数据结构的位置也应随机偏移。

## 防止信息泄露
- 地址不能泄露。对 printk 的 [%p](https://www.kernel.org/doc/html/latest/core-api/printk-formats.html#pointer-types) 进行控制，指针打印前先通过 [ptr_to_id()](https://elixir.bootlin.com/linux/v5.14/source/lib/vsprintf.c#L821) 哈希；不要将地址直接作为 ID 使用，需要 ID 的话，通过 [IDR](https://www.kernel.org/doc/html/latest/core-api/idr.html) 机制分配。
- 数据不能泄露。内存复制到用户空间前要初始化；释放内存前清理掉原始数据，不仅是「堆」，还包括如 syscall 返回时释放的栈上空间。

简而言之，在信息进入用户空间前，要么清理，要么混淆。尤其要注意对 procfs 和 sysfs 的写操作，设计它们就是为了能从用户空间访问内核数据。对于攻击者来说，这是从内核态向用户空间泄露信息的天然通道。
