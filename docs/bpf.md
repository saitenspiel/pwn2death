# 从 BPF 到 eBPF
Berkeley 包过滤器（Berkeley Packet Filter，BPF）是一种基于虚拟机（pseudo machine）的过滤机制，在 92 年 Berkeley 的论文 [The BSD Packet Filter](http://www.tcpdump.org/papers/bpf-usenix93.pdf) 中提出。论文原型在 BSD 上实现，所以将其命名为 BPF，但随着 BPF 被各大 OS 接受，B 的含义也从 BSD 逐渐变成 Berkeley。

BPF 本来是为网络数据包过滤设计，但由于方案优越，现已成长为内核的独立子系统 [/kernel/bpf](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf)，广泛用于流量控制、跟踪、性能监控/调优等领域。

eBPF（extended BPF）将 BPF 的虚拟机从 32 位扩展到 64 位，重新设计指令集，引入 map 机制与用户空间通信，极大地拓宽了 BPF 的适用场景。

eBPF 提出后，原先的 BPF 改称 cBPF（classic BPF）以示区分。现在的内核使用 eBPF，但允许用户空间传入 cBPF 程序，内核负责将其转换为 eBPF 格式。

## CSPF
BPF 之前，主流包过滤方案是 87 年提出的 [CMU/Stanford Packet Filter](https://dl.acm.org/doi/pdf/10.1145/37499.37505)，CSPF。当时内核网络栈还不成熟，包过滤普遍放在用户进程做。CSPF 给出了在内核进行包过滤的有效方案。

包过滤即根据字段的取值响应数据包，丢弃或分发到相应的处理进程。核心是布尔表达式求值，如过滤 TCP 包，关键是对表达式 ether.type == IP && ip.proto == TCP 求值。

CSPF 设计了一个虚拟机，在内核运行，负责解释执行用户空间传入的过滤规则，即用 CSPF 指令序列描述的布尔表达式。CSPF 的指令集非常简单，每条指令包含一个栈操作和一个二元运算：栈操作要么压入一个立即数，要么压入包的第 n 个字（16 位）；二元运算可以是大小比较或者逻辑运算。指令执行时，先进行栈操作，然后出栈两个操作数进行二元运算，结果入栈。最终栈顶元素即为布尔表达式的值。

CSPF 数据宽度只有 16 位，对于 32 位的数据，如 IP 地址，要分两次计算；访问以字为单位，奇数长度的数据包最后一个字节无法访问。这些问题可以通过简单扩展指令集解决。但 CSPF 的根本局限在于，将布尔表达式抽象为二叉树，基于栈自底向上进行求解，这意味着：

- 不支持短路，导致求解不必要的分支；
- 不支持单元运算，如间接运算，因此无法处理变长头部；
- 指令集必须围绕栈操作设计，效率上无法支撑复杂逻辑。

CSPF 方案有两点尤其值得借鉴：

- 将数据包视作字节数组，过滤机制的设计独立于具体协议，用户程序负责定义过滤规则；
- 用虚拟机做过滤，可扩展性极强。

这两点为 BPF 所继承，奠定了 BPF 成功的基础。BPF 能发展出如今的通用性，CSPF 功不可没。

## BPF
BPF 在 CSPF 的基础上做了两大改进：

- 将布尔表达式抽象为控制流图，短路不必要的分支求解；
- 改用基于寄存器的虚拟机，更加灵活。例如，可以在寄存器中动态计算偏移，处理变长头部。

BPF 将布尔表达式抽象为有向无环的控制流图（CFG），本质是一个特殊的有限状态机。CFG 由若干非叶节点和两个叶子节点 true 和 false 组成。每个非叶节点对应一次谓词判断，有且仅有 true 和 false 两个分支。从入度为 0 的节点向下计算，最终到达叶子节点，得到表达式的值。

BPF 使用基于寄存器的虚拟机实现，字长 32 位，包括：

- 一个 32 位累加器 A；
- 一个 32 位索引寄存器 X；
- 一小段随机存储 M[]，由 16 个 32 位寄存器组成，从 0 开始索引；
- 一个隐含的 PC 寄存器。

### 指令集
BPF 指令长度固定为 64 位，由四个字段组成：
```txt
opcode:16, jt:8, jf:8, k:32
```
jt 和 jf 对应 true 和 false 分支下的跳转目标，采用相对跳转。k 为通用存储。opcode 分为以下几类：

- Load 类，加载数据到 A 或 X，包括立即数、包数据、包长度或 M 中的数据；
- Store 类，将 A 或 X 中的数据存储到 M；
- ALU 类，算术/逻辑运算，源为 X 或立即数，目的为 A；
- Branch 类，比较 A 与 X 或立即数，相应跳转到 jt 或 jf；
- Return 类，返回接受的数据长度，为 0 表示丢弃整个数据包；
- Misc 类，加载 A 到 X 或加载 X 到 A。

该指令集提供相当完备的包过滤能力。例如，按 TCP 目的端口号进行过滤，这需要处理可变长度的 IP 头：
```x86asm
ldh [12]                    ; link layer type
jeq #ETHERPROTO_IP, L1, L5
L1: ldb [23]                ; IP protocol type
jeq #IPPROTO_TCP, L2, L5
L2: ldh [20]                ; flag | fragment offset
jset #0x1fff, L5, L3        ; check the fragment offset, reject if non-zero
L3:ldb [14]                 ; version | header length
and #0xf                    ; header length, with word offset
lsh #2                      ; scale by four, get the byte offset
tax
ldh [x+16]                  ; TCP destination port
jeq #N, L1, L2
L4: ret #TRUE               ; non-zero value indicating # of bytes to save
L5: ret #0                  ; reject the packet
```
`ldb/ldh [k]` 加载数据包 k 字节偏移处的 byte/halfword 到 A；`#k, Lt, Lf` 在谓词为 true/false 时跳转到 Lt/Lf，`jeq` 相等为 true，`jset` 计算 A & k，非零为 true；`tax` 加载 A 到 X。

### 早期实现
BPF 最早的内核实现是 2.1.75 版本的 [/net/core/filter.c](https://elixir.bootlin.com/linux/2.1.75/source/net/core/filter.c)，只有三个函数：

- [sk_run_filter()](https://elixir.bootlin.com/linux/2.1.75/source/net/core/filter.c#L46)，在 [skb->data](https://elixir.bootlin.com/linux/2.1.75/source/include/linux/skbuff.h#L105) 上运行过滤器，即 BPF 指令（[sock_filter](https://elixir.bootlin.com/linux/2.1.75/source/include/linux/filter.h#L19)）序列，返回截断长度；
- [sk_chk_filter()](https://elixir.bootlin.com/linux/2.1.75/source/net/core/filter.c#L266)，验证过滤器，跳转指令不能越界，对 M[] 的访问不能越界，必须以 ret 结尾；
- [sk_attach_filter()](https://elixir.bootlin.com/linux/2.1.75/source/net/core/filter.c#L333)，附加过滤器到指定 socket，即验证并拷贝用户 [通过 setsockopt() 传入](https://elixir.bootlin.com/linux/2.1.75/source/net/core/sock.c#L330) 的过滤器到 kmalloc() 的内存，让 [sk->filter_data](https://elixir.bootlin.com/linux/2.1.75/source/include/net/sock.h#L462) 指向该区域。之后 [sock_queue_rcv_skb()](https://elixir.bootlin.com/linux/2.1.75/source/include/net/sock.h#L885) 在该 socket 上接收到数据时，会先运行过滤器进行截断，如果没有完全丢弃，再调用 [sk->data_ready()](https://elixir.bootlin.com/linux/2.1.75/source/include/net/sock.h#L551) 通知上层应用。

这时只支持在指令中使用包长度 skb->len。后来逐渐扩展到 skb->protocol/hash 等，还支持生成随机数，方便包采样。这部分扩展称作 [BPF extensions](https://www.kernel.org/doc/html/latest/networking/filter.html#bpf-engine-and-instruction-set)。

### JIT 编译器
内核 3.0 引入 x86 下的 JIT 编译器 [bpf_jit_compile()](https://elixir.bootlin.com/linux/v3.0/source/arch/x86/net/bpf_jit_comp.c#L121)，将 BPF 指令序列编译为 native code：

- A 翻译为 EAX；X 翻译为 EBX；M 在栈上分配，M[k] 翻译为 RBP-0x10-k*4；
- skb 的地址通过 RDI 传入；返回值放 EAX；
- 编译有 10 个重复 pass，主要是为了向前看，确定是否需要保存 RBX/分配 M/引用 skb 内容；
- 如果本 pass 与上一 pass 生成的 native code 长度相同，表明编译成功，放入 module_alloc() 的内存中；如果始终不收敛，放弃编译；
- 通过 [SK_RUN_FILTER](https://elixir.bootlin.com/linux/v3.0/source/include/linux/filter.h#L163) 宏调用 [sk_filter-> bpf_func](https://elixir.bootlin.com/linux/v3.0/source/include/linux/filter.h#L142)，如果编译成功，该函数指针指向生成的 native code，否则指向 [sk_run_filter()](https://elixir.bootlin.com/linux/v3.0/source/net/core/filter.c#L112)。

函数指针 bpf_func 让 BPF 可以方便地迁移到其他场景，不再局限于 socket 上的过滤。内核 3.4 起，BPF 应用于系统调用的过滤 [seccomp](../seccomp-bpf)。

## eBPF
eBPF 最开始是作为一种中间表示在 3.15 引入，使用全新的指令集。BPF 程序加载到内核时，优先进行 JIT 编译，如果编译失败，则调用 [sk_convert_filter()](https://elixir.bootlin.com/linux/v3.15/source/net/core/filter.c#L863) 将其转化为中间表示，之后通过 [__sk_run_filter()](https://elixir.bootlin.com/linux/v3.15/source/net/core/filter.c#L141) 运行。

到 3.16，这一中间表示正式更名为 eBPF，对应的 JIT 编译器 [bpf_int_jit_compile()](https://elixir.bootlin.com/linux/v3.16/source/arch/x86/net/bpf_jit_comp.c#L869) 也一并引入。原来的 BPF 改名 cBPF，对应的 JIT 编译器不再使用。

3.17 开始，eBPF 成为内核独立子系统 /kernel/bpf。3.18 引入系统调用 bpf()，eBPF 框架基本成型，之后逐渐应用至流量控制（[XDP](https://docs.cilium.io/en/v1.10/bpf/#xdp)，[tc](https://docs.cilium.io/en/v1.10/bpf/#tc-traffic-control)）、跟踪（tracepoint，k/uprobe）、性能监控/调优（perf_event）等领域。

### 虚拟机
引入 eBPF 的初衷是向上提供易用的高级语言接口，向下配合 64 位架构，提高执行效率。构想是在用户空间用 C 编写，GCC/LLVM 编译时优化，eBPF 指令自然地映射为 native code 以降低 JIT 编译开销。

为此，eBPF 虚拟机在 BPF 的基础上做了如下改进：

- 寄存器增加到 11 个，R0-R10；
- 寄存器宽度扩展到 64 位，低 32 位可以作为子寄存器单独访问，写时零扩展到 64 位；
- 指令集重新设计，允许过程调用 BPF_CALL。

过程调用的调用约定：

- R0-R5 为 caller-saved，R1-R5 传参，R0 放返回值；
- R6-R9 为 callee-saved；
- R10 为只读的帧指针。

该调用约定是为了配合 CPU 架构。以 x64 架构为例，6 个参数寄存器，6 个 callee-saved 寄存器，因此 eBPF 寄存器能一对一映射到 CPU 寄存器，BPF_CALL 指令可以直接 JIT 编译为 call 指令：

- R0 映射为 RAX;
- R1-R5 映射为 RDI，RSI，RDX，RCX，R8；
- R6-R9 映射为 RBX，R13，R14，R15；
- R10 映射为 RBP。

### 指令集
eBPF 指令集按 RISC 架构设计，指令长度和 BPF 一样，固定为 64 位，但重新设计了编码方式：
```txt
opcode:8, dst_reg:4, src_reg:4, off:16, imm:32
```
16 位 opcode 空间太大，缩减为 8 位。目的 dst_reg 和源 src_reg 显式指明，方便 JIT 编译。off 用于内存操作数的寻址。通用存储 imm，即 cBPF 指令的 k，保持 32 位不变。

新编码用单分支 jt/fall-through 替代了双分支 jt/jf：

- 单分支性能更优。对于解释执行，两种方式性能差不多，但对于 JIT 编译，jt/fall-through 可以自然地编译为一次条件跳转，jt/jf 则需要一次条件跳转加一次无条件跳转；
- 单分支更符合过滤器需要的语义。双分支在过滤器中实际用得很少，经常是让 jf 分支指向下一条指令，当成 jt/fall-through 使用。

opcode 在 cBPF 基础上扩展，尽量复用其编码，以便后向兼容：

- Load/Store 类，规范寻址模式，附带支持少量原子运算；
- ALU 类，增加 BPF_MOV 指令，符号扩展的右移运算，以及大小端转换；
- JMP 类，完善条件跳转，增加 BPF_CALL 和 BPF_EXIT 指令。

对于 Load/Store 类指令，opcode 还编码了 size 和 mode。size 为操作宽度，可选 1/2/4/8 字节。mode 对指令进一步分类：

- BPF_MEM，常规存取指令，在内存和寄存器间移动数据，或存储 32 位立即数到内存；
- BPF_ATOMIC，原子更新目的内存，支持 add/and/or/xor/xchg/cmpxchg 运算；
- BPF_IMM，加载 64 位立即数到寄存器，高 32 位占用下一条指令的 imm 字段存储。

BPF_ATOMIC 仅用于 Store 类指令，操作宽度限于 4/8 字节。源操作数必须是 src_reg，不能是立即数。运算类型借用 imm 字段存储。

BPF_IMM 仅用于 BPF_LD 指令，操作宽度必须是 8 字节。可用于加载 map 地址、map 数据、BTF 变量地址或 eBPF 函数地址，src_reg 字段需相应设置为 BPF_PSEUDO_{[MAP_FD](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L1108)、[MAP_VALUE](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L1117)、[BTF_ID](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L1127)、[FUNC](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L1136)}。

对于 ALU 类和 JMP 类指令，opcode 用 1 bit 的 src 指示源操作数是 src_reg 还是 imm。BPF_MOV 寻址模式有限，仅在寄存器间移动数据，或加载 32 位立即数到寄存器，因此不完全对应 mov 指令。

### Map
eBPF map 是位于内核的 Key-Value 存储，eBPF 程序借此与用户空间通信：
- 用户进程通过系统调用 bpf() 管理 map 并访问其数据；
- 运行在内核的 eBPF 程序通过 helper 函数访问 map 数据。

Map 类似一个 Key-Value 版本的 [VFS](../todo)，底层有不同类型（[bpf_map_type](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L867)）的实现。基本类型是数组和哈希表，可以当槽用，如：

- [PROG_ARRAY](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/arraymap.c#L1087) 存储 eBPF 程序的地址，用于 tail call；
- [ARRAY_OF_MAPS](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/arraymap.c#L1317)/[HASH_OF_MAPS](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/hashtab.c#L2231) 存储 map 的地址。

另外有 [LPM_TRIE](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/lpm_trie.c#L721) 实现了 trie 树，可用于路由转发等前缀匹配场景。各类型 map 的可用操作不完全相同，通过结构体 [bpf_map_ops](https://elixir.bootlin.com/linux/v5.13/source/include/linux/bpf.h#L61) 声明。按 key 查找、更新和删除这三个核心操作需要请求 [RCU 锁](../todo)。

### 文件系统
创建 map 即生成一个 [匿名 inode](../todo) 并返回其文件描述符 fd，之后用户进程通过 fd 操作 map。一旦 fd 关闭，对应 map 就会丢失。为了延长 map 的生命周期，让不同进程能共享该 map，允许将其写入一个内存文件系统，供之后的进程访问。

该文件系统在 [inode.c](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/inode.c) 中定义，支持软硬链接，默认挂载点 /sys/fs/bpf，读写操作封装为 bpf() 命令：

- BPF_OBJ_PIN，将 fd 对应的 map 写入文件系统，即将匿名 inode 关联到某个路径；
- BPF_OBJ_GET，从对应路径取出 map。

eBPF 程序也一样，加载到内核时生成 fd，通过写入文件系统分享给其他进程。用户进程总是通过 fd 访问 eBPF 程序/map，内核负责将其解析为具体地址：

- 写 eBPF 程序/map 地址到相应类型的 map：[bpf_fd_array_map_update_elem()](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/arraymap.c#L759) 负责取得具体地址，用户进程只用传 fd 给 bpf()；
- 加载 map 地址到寄存器：Verifier 在 [resolve_pseudo_ldimm64()](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/verifier.c#L11220) 时，将 [BPF_LD_MAP_FD](https://elixir.bootlin.com/linux/v5.13/source/include/linux/filter.h#L202) 指令中的 fd 重写为 map 地址。编写 eBPF 程序时只用给出目标 map 的 fd。 

### 主程序
eBPF 程序有特定的类型 [bpf_prog_type](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L908)，通过 [bpf_prog_load()](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/syscall.c#L2079) 加载到内核 vmalloc() 的内存，然后通过 setsockopt() 等方式附加到特定数据结构。部分类型，主要是 [cgroup](../todo) 相关，附加操作没有用户空间接口，对此 bpf() 提供 BPF_PROG_ATTACH 命令，可附加 eBPF 程序到 [bpf_attach_type](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L942) 中指示的数据结构。

eBPF 程序通过 [bpf_prog->bpf_func](https://elixir.bootlin.com/linux/v5.13/source/include/linux/filter.h#L570) 执行：

- 优先 JIT 编译 [bpf_int_jit_compile()](https://elixir.bootlin.com/linux/v5.13/source/arch/x86/net/bpf_jit_comp.c#L2207) 执行，此时 bpf_func 指向 image，即生成的 native code；
- 如果无法编译执行，通过 [___bpf_prog_run()](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/core.c#L1369) 执行，此时 bpf_func 指向解释器 [interpreters](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/core.c#L1740)。

相关内核变量 bpf_jit_enable，内核选项 CONFIG_{BPF_JIT、BPF_JIT_ALWAYS_ON、HAVE_EBPF_JIT}。
JIT 编译只要可用，就会应用在所有过滤器上，没有类似 JVM 的 hotspot 机制。目前主流架构都支持 JIT 编译，但对高级特性如 bpf2bpf call 的支持还不成熟。

bpf_func 接收两个参数 ctx 和 insn。ctx 为 void 指针，指向待过滤数据，根据 eBPF 程序类型解析为不同数据结构，如 skb，参见 [bpf_types.h](https://elixir.bootlin.com/linux/v5.13/source/include/linux/bpf_types.h)。insn 指向 eBPF 指令序列，仅用于解释执行。JIT 编译执行时，ctx 通过 CPU 寄存器自然传递，如 x64 的 RDI。解释执行时，解释器会先 [将 ctx 放入 R1](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/core.c#L1702)。

eBPF 程序通过 BPF_EXIT 指令退出，退出前必须显式保存返回值到 R0。解释器遇到 BPF_EXIT 指令直接返回 R0 即可。x64 的 JIT 编译器仍使用帧指针 RBP，因此，对于 BPF_EXIT 需先发射 leave 指令再 ret。

### 过程调用
eBPF 使用 BPF_CALL 指令进行过程调用，最多五个参数，ctx 始终通过 RDI/R1 传递：

- 调用特定的 helper 函数；
- tail call，调用另一个 eBPF 程序，类型必须相同，调用后不返回；
- bpf2bpf call，调用 eBPF 函数，调用后返回到下一条指令。

#### helper call
helper 函数使用 [BPF_CALL_x](https://elixir.bootlin.com/linux/v5.13/source/include/linux/filter.h#L486) 宏定义，通过 [bpf_func_id](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L4912) 标识，用于访问 map 数据或者操作特定数据结构，如读写 skb。helper 函数预先定义在内核，不能动态装卸，只能重新编译内核。

定义 helper 函数需提供原型描述 [bpf_func_proto](https://elixir.bootlin.com/linux/v5.13/source/include/linux/bpf.h#L337)，以及 func_id 到 func_proto 的映射 [get_func_proto()](https://elixir.bootlin.com/linux/v5.13/source/include/linux/bpf.h#L464)。func_proto 包含返回值和参数类型等信息，Verifier 据此对 eBPF 程序进行类型检查。

调用 helper 函数只需设置 BPF_CALL 的 imm 为 func_id。Verifier 在 [do_misc_fixups()](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/verifier.c#L12351) 时，将 imm 重写为对应 helper 函数相对 [__bpf_call_base()](https://elixir.bootlin.com/linux/v5.13/source/include/linux/filter.h#L913) 的地址偏移。helper 函数地址也包含在 func_proto 中。

helper 函数地址通过相对偏移计算，然后：

- 对于 JIT 编译器，发射 call 指令即可；
- 对于解释器，用 R1-R5 作为参数调用 helper 函数，返回值写入 R0。

#### [tail call](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3f55b7ed5e5d4aa7291e3a1e2f7224eeba5810ba)

tail call 使用 BPF_CALL 指令，func_id 为 BPF_FUNC_tail_call，对应一个形式上的、没有具体实现的 helper 函数，接受三个参数：R1 为 ctx；R2 为指向 PROG_ARRAY 类型 map 的指针；R3 为 map 的索引，指示要调用的 eBPF 程序。Verifier 在 do_misc_fixups() 时将其 [替换为内部指令 BPF_TAIL_CALL](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/verifier.c#L12513)。

tail call 严格来说不算过程调用。它相当于一个到目标 eBPF 程序的无条件跳转，因此将复用当前栈帧，且不会返回到当前程序。如果索引越界，或目标程序为空，或 tail call 次数超过 32（[MAX_TAIL_CALL_CNT](https://elixir.bootlin.com/linux/v5.13/source/include/linux/bpf.h#L1015)），tail call 将失败，此时程序会继续执行。

PROG_ARRAY 记录的是结构体 [bpf_prog](https://elixir.bootlin.com/linux/v5.13/source/include/linux/filter.h#L550) 的地址：

- JIT 编译器从中取出 bpf_func，通过 [retpoline](../todo)，即 [RETPOLINE_RCX_BPF_JIT()](https://elixir.bootlin.com/linux/v5.13/source/arch/x86/include/asm/nospec-branch.h#L328) 宏跳转到该地址；
- 解释器从中取出 eBPF 指令序列，让 PC 指向第一条指令即可。

因此，tail call 链上的程序要么都 JIT 编译执行，要么都解释执行，不能交替。调用链可以动态调整，修改 PROG_ARRAY 即可，在包处理 pipeline 这类场景中很有用。

#### [bpf2bpf call](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ef9fde06a259f5da660ada63214addf8cd86a7b9)

bpf2bpf call 也使用 BPF_CALL 指令，设置 src_reg 为 [BPF_PSEUDO_CALL](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L1141)，imm 为到目标函数的指令偏移，其指向的 eBPF 指令即为函数起点。Verifier 按此将传入的 eBPF 程序分割为若干 subprog，主函数也包括在内，均通过 BPF_EXIT 返回。

Verifier 在 [fixup_call_args()](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/verifier.c#L12277) 时：

1. 调用 [jit_subprogs()](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/verifier.c#L12034)：
    1. 单独编译每个 subprog，此时 bpf2bpf call 仍表示为 BPF_CALL 指令，JIT 编译器对其按 helper call 处理，即发射一条目标为__bpf_call_base() + imm 的 call 指令，imm 事先重写为 1；
    2. 如果 subprog 都编译成功，重写 bpf2bpf call 指令的 imm 为目标 image 相对 __bpf_call_base() 的地址偏移，再次调用 JIT 编译器，修正 call 的目标地址。
2. 调用 [bpf_patch_call_args()](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/core.c#L1756)，截断 imm 放入 off，重写 imm 为带参解释器 [interpreters_args](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/core.c#L1748) 相对 [__bpf_call_base_args()](https://elixir.bootlin.com/linux/v5.13/source/include/linux/filter.h#L914) 的地址偏移，将 BPF_CALL 替换为内部指令 BPF_CALL_ARGS。

BPF_CALL_ARGS 指令仅用于解释执行。bpf2bpf call 是标准的过程调用，需要新栈帧，有返回值，因此只更新 PC 是不行的。解释器将其转为 C 过程调用执行，对应 [JMP_CALL_ARGS](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/core.c#L1522) 标签：
```c
JMP_CALL_ARGS:
    BPF_R0 = (__bpf_call_base_args + insn->imm)(BPF_R1, BPF_R2,
                                                BPF_R3, BPF_R4,
                                                BPF_R5,
                                                insn + insn->off + 1);
```

即调用带参解释器解释执行目标函数。因为要传递 R1-R5 五个参数，所以不能复用只传递 ctx 的 interpreters。但除此之外两者没有区别，都是调用 ___bpf_prog_run()。将 bpf2bpf call 实现为 C 过程调用可以简化解释器，弊端是不支持混用 bpf2bpf call 和 tail call，这样的 eBPF 程序只能 JIT 编译执行。

bpf2bpf call 有栈溢出的危险。Verifier 在 [check_max_stack_depth()](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/verifier.c#L3596) 进行检查：

- bpf2bpf call 最多嵌套 8 次（[MAX_CALL_FRAMES](https://elixir.bootlin.com/linux/v5.13/source/include/linux/bpf_verifier.h#L218)），总栈深不能超过 512 字节（[MAX_BPF_STACK](https://elixir.bootlin.com/linux/v5.13/source/include/linux/filter.h#L83)）；
- tail call 时栈深不能超过 256 字节。tail call 最多 32 次，因此最多消耗 8K 栈空间。

对用户来说，eBPF 编程是面向 C 的，没有 bpf2bpf call，函数照样能用，加上 always_inline 而已。bpf2bpf call 的主要意义是缩小 eBPF 程序体积，从而更快加载。

### 用户空间接口
用户空间通过系统调用 [bpf()](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/syscall.c#L4369) 操作 eBPF 程序/map。在 bpf() 的基础上封装了更易于使用的 [libbpf](https://elixir.bootlin.com/linux/v5.13/source/tools/lib/bpf)。[sock_example](https://elixir.bootlin.com/linux/v5.13/source/samples/bpf/sock_example.c#L35) 用 eBPF 进行数据包统计：

1. [bpf_create_map()](https://elixir.bootlin.com/linux/v5.13/source/tools/lib/bpf/bpf.c#L128) 创建 map；
2. 使用 [指令宏](https://elixir.bootlin.com/linux/v5.13/source/tools/include/linux/filter.h#L34) 编写 eBPF 程序，负责在 map 中更新统计信息；
3. [bpf_load_program()](https://elixir.bootlin.com/linux/v5.13/source/tools/lib/bpf/bpf.c#L369) 加载该程序；
4. setsockopt() 附加该程序到 lo 网卡；
5. [bpf_map_lookup_elem()](https://elixir.bootlin.com/linux/v5.13/source/tools/lib/bpf/bpf.c#L424) 查询 map，得到统计信息。

与指令宏相比，用高级语言编写 eBPF 程序更友好。LLVM 已有完善的 eBPF 后端，可以从 C 生成 [ELF 格式](../todo) 的目标文件，其中包含 eBPF 程序和 map 信息等。libbpf 能解析该目标文件，完成所需 map 的创建以及 eBPF 程序的加载。

### Verifier
eBPF 程序运行在内核态，加载到内核时调用 [bpf_check()](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/verifier.c#L13358)，即 Verifier 验证其安全性：

1. 解析 subprog，通过 [add_subprog()](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/verifier.c#L1545) 记录；
2. [check_subprogs()](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/verifier.c#L1776)：跳转不能超出当前 subprog，每个 subprog 必须以 BPF_EXIT 或无条件跳转结尾；
3. resolve_pseudo_ldimm64() 解析 [BPF_LD_IMM64_RAW](https://elixir.bootlin.com/linux/v5.13/source/include/linux/filter.h#L187) 指令，重写 map fd 等为具体地址；
4. [check_cfg()](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/verifier.c#L9406)：DFS，所有指令都要可达；
5. [do_check_main()](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/verifier.c#L12909) -> [do_check_common()](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/verifier.c#L12785) -> [do_check()](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/verifier.c#L10594)：依次检查主函数的每条指令：
    - 指令各字段必须合法，总数不能超过 100 万（[BPF_COMPLEXITY_LIMIT_INSNS](https://elixir.bootlin.com/linux/v5.13/source/include/linux/bpf.h#L1014)）；
    - [check_helper_call()](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/verifier.c#L5956)，[check_func_call()](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/verifier.c#L5698)：过程调用必须合法；
    - [check_mem_access()](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/verifier.c#L4060)：目标内存必须可访问。
6. check_max_stack_depth()：检查 bpf2bpf call 的栈深；
7. do_misc_fixups() 重写 func_id，处理 tail call 等；
8. fixup_call_args()：处理 bpf2bpf call。

### 后向兼容
#### cBPF
[bpf_prog_create_from_user()](https://elixir.bootlin.com/linux/v5.13/source/net/core/filter.c#L1402) 将用户空间传入的 cBPF 程序 [sock_fprog](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/filter.h#L31) 转换成 eBPF 程序 bpf_prog 运行。其通过 copy_from_user() 取得 cBPF 指令，然后调用 [bpf_prepare_filter()](https://elixir.bootlin.com/linux/v5.13/source/net/core/filter.c#L1308)：

1. [bpf_check_classic()](https://elixir.bootlin.com/linux/v5.13/source/net/core/filter.c#L1051) 检查 cBPF 程序，指令必须合法，跳转不能越界，必须以 ret 结尾，只有写过的 M[] 才能读；
2. [bpf_migrate_filter()](https://elixir.bootlin.com/linux/v5.13/source/net/core/filter.c#L1237) 调用两次 [bpf_convert_filter()](https://elixir.bootlin.com/linux/v5.13/source/net/core/filter.c#L555) 完成转换。第一次计算指令长度，分配内存，第二次正式转换：A 对应 R0，X 对应 R7，M[] 对应帧寄存器 R10；
3. [bpf_prog_select_runtime()](https://elixir.bootlin.com/linux/v5.13/source/kernel/bpf/core.c#L1840) 尝试 JIT 编译得到的 eBPF 程序，失败则解释执行。

#### 32 位机器
eBPF 是 64 位的虚拟机，如果要在 32 位机器上 JIT 编译执行，寄存器要么映射为两个 CPU 寄存器，要么映射到栈上。x86 的 JIT 编译器采用后一种做法，eBPF 寄存器映射到 EBP 上方预留的一段栈空间，通过 EBP 寻址，栈帧如下：
```text
+-------------------+ <- old ESP
| callee-saved regs |
+-------------------+
| eBPF regs mapping |
+-------------------+ <- EBP = R10
| bpf prog stack    |
+-------------------+ <- ESP
```
[emit_prologue()](https://elixir.bootlin.com/linux/v5.13/source/arch/x86/net/bpf_jit_comp32.c#L1199) 负责建立栈帧，[SCRATCH_SIZE](https://elixir.bootlin.com/linux/v5.13/source/arch/x86/net/bpf_jit_comp32.c#L177) 即预留空间大小：
```x86asm
push ebp
mov ebp, esp

push edi
push esi
push ebx

; STACK_SIZE = SCRATCH_SIZE + stack depth required by bpf_prog, 8-byte aligned
sub esp,STACK_SIZE
; SCRATCH_SIZE = 0x60, for R0-R10 and tail_call_cnt
sub ebp,SCRATCH_SIZE+12
```