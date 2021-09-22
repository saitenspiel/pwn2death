# 内存虚拟化
在虚拟化环境中，guest 和 host 拥有各自的地址空间：

- guest 的虚拟地址 gVA 和物理地址 gPA；
- host 的虚拟地址 hVA 和物理地址 hPA。

内存访问要经过 gVA->gPA->hVA->hPA 的地址转换。

另外有一套描述方式 VA->PA->MA，即虚拟地址到物理地址到机器地址，对应 gVA->gPA->hPA。这是在不关心 host 上其他应用，只考虑 guest-VMM-host 这一层级结构时，暂时忽略了 hVA。

## gPA-hVA 表
早期的内存虚拟化使用 gPA-hVA 表完成 gVA 到 hPA 的转换。

即在 guest 中，借助 guest 页表完成 gVA->gPA 的转换，然后让 guest 对 gPA 的访问陷入到 VMM 中，VMM 使用自己维护的 gPA-hVA 表将 gPA 转换为 hVA，而后借助 host 页表完成 hVA->hPA 的转换。

gPA-hVA 表的典型实现是 KVM 的 kvm_memory_slot 结构。这种方式要求 guest 的每次内存访问都陷入 VMM，开销巨大，为此提出了影子页表。

## 影子页表
影子页表（Shadow Page Table）是纯软件的实现方式，VMM 在其中维护 gVA->hPA 的映射。当 guest 将进程页表的基址写入 CR3 时，VMM 介入，将 CR3 写为对应影子页表的基址，这样物理 MMU 访问的始终是影子页表。

影子页表未命中时，发生 page fault，陷入 VMM：

1. VMM 查找 guest 页表，guest 页表也不命中，guest 进行缺页处理，更新 guest 页表；
2. guest 页表设置了写保护，再次陷入 VMM，VMM 根据所写内容更新影子页表。

随着系统运行，越来越多的 gVA 被访问，影子页表逐渐建立。而当影子页表命中时，就可以一次完成从 gVA 到 hPA 的转换，从而提高效率。

但是，影子页表有两个重大缺陷：

1. 需要为 guest 的每个进程维护一张影子页表；
2. 需要 VMM 介入所有 page fault 和 guest 页表的更新。

## EPT
Intel 的 EPT，Enhanced Page Table 方案为 gPA->hPA 的转换提供硬件支持。AMD 有类似机制 NPT，Nested Page Table。

EPT 方案引入专门的 EPT 页表，以及相应的 EPT Base Pointer 寄存器，负责存储 EPT 页表基址：

- gVA->gPA 的转换借助 guest 页表在 CR3 上进行；
- gPA->hPA 的转换借助 EPT 页表在 EPT Base Pointer 寄存器上进行。

两次地址转换都通过硬件完成，互相独立，可以交替进行。每次寻址时，先使用 gVA 查询guest 顶级页表获得下级页表的 gPA，再查询 EPT 获得 gPA 对应的 hPA，访问该 hPA 即可取出下级页表，这样的一轮查询被称作 nested walk。多级页表下，每次寻址都需要多轮 nested walk。

和影子页表一样，EPT 页表也是逐渐建立的。EPT 页表未命中时，发生 page fault，陷入 VMM，由 VMM 为目标 gPA 分配 hPA，并相应建立 EPT 表项。

只有 EPT 页表的 page fault 会导致陷入 VMM，而 guest 页表的 page fault 直接在 guest 内部处理。因为使用的基址寄存器不同，这两类 page fault 是可区分的。

EPT 方案解决了影子页表的两大缺陷：

1. 只需要为每个 guest 维护一张 EPT 页表；
2. guest 页表的 page fault 由 guest 自己处理，不会陷入 VMM。

