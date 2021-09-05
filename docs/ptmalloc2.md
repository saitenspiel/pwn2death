# ptmalloc2
## 概述
[ptmalloc](http://www.malloc.de/en/)（pthreads malloc）是 C 风格内存管理 malloc()/free() 的一个实现，在其前身 dlmalloc（Doug Lea malloc）的基础上添加了多线程支持。ptmalloc2 是它的第二个版本，发布后很快集成到 glibc 中，之后的版本迭代也直接在 glibc 中进行。版本迭代一是会增加新的完整性检查，有些利用原语会随之失效；二是会引入新机制，这可能带来新的利用点。因此 [glibc 下的堆利用](../heap-exp-house-of-xxx) 与版本关系密切。

### 主要思路
- 小内存在堆区分配，大内存在 mmap 区分配；
- 为子线程在 mmap 区分配模拟堆；
- 将堆区内存看作连续的 chunk，大小等元数据记录在 chunk 头部，对应用程序不可见；
- 通过 tcache - fastbin - unsorted bin - small/large bin 这个层级结构管理释放的 chunk；
- 在堆顶维护空闲内存池 top chunk；

### 默认值
- tcache <= 65 个双字，fastbin <= 8 个双字；
- \>= 64 个双字为 large chunk；
- 释放的 chunk >= 64K，将整理 fastbin；
- \>= 128K，通过 mmap() 分配，动态增加，<= 32M；
- top chunk >= 128K，将回收系统内存，随 mmap 阈值更新；
- 模拟堆 64M，按照 mmap 阈值上限的两倍设计；

## 数据结构
### chunk
堆块（chunk）是内存申请和释放的基本单位。记前一物理相邻 chunk 为 prevchunk，地址更低，下一物理相邻 chunk 为 nextchunk，地址更高。chunk 由结构体 [malloc_chunk](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1158) 描述：
```c
struct malloc_chunk {
  INTERNAL_SIZE_T mchunk_prev_size; /* Size of previous chunk (if free).  */
  INTERNAL_SIZE_T mchunk_size;      /* Size in bytes, including overhead. */

  struct malloc_chunk* fd;          /* double links -- used only if free. */
  struct malloc_chunk* bk;

  /* Only used for large blocks: pointer to next larger size.  */
  struct malloc_chunk* fd_nextsize; /* double links -- used only if free. */
  struct malloc_chunk* bk_nextsize;
};
```
前两个 *_size 字段分别记录 prevchunk 和当前 chunk 的大小，类型为 [INTERNAL_SIZE_T](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc-internal.h#L55)，即 size_t，64 位系统下为 8 字节。这实际上维护了一条隐形双链表，由此物理上相邻的 chunk 能够在 O(1) 时间内互相定位，大大提升了 chunk 合并的效率。

这两个字段合在一起称为 chunk 的头部（header），对于用户来说 chunk header 是不可见的，malloc() 返回到用户的地址 [mem](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1193) 是 chunk 起始地址跳过 header，即加上 [CHUNK_HDR_SZ](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1297) 之后的地址。

chunk 最小也要两个双字（[MINSIZE](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1314)）：chunk header 占一个，另一个留给 fd/bk 指针（对于空闲 chunk）或用户数据（对于已分配 chunk）使用。

此外，chunk 还是双字对齐的。这里的字长为 [SIZE_SZ](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc-internal.h#L59)，即 sizeof(size_t)，8 字节，双字即 16 字节。具体来说，每个 chunk 的起始地址双字对齐，大小双字对齐，且因为 header 大小为双字，所以 mem 也是双字对齐。双字对齐意味着 mchunk_size 字段的低 4 位用不到。其中低 3 位用来存储标志，从低到高分别是：

- [PREV_INUSE](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1375)，P 标志位。P = 1 表示 prevchunk 已分配，此时 mchunk_prev_size 字段，即 chunk 的第一个字不使用，改存 prevchunk 的用户数据。第一个（地址最低）chunk 的 P 位强制为 1，空闲 chunk 的 P 位始终为 1，因为不会出现两个物理相邻的空闲 chunk。该位也叫作 prevchunk 的 inuse 位。
- [IS_MMAPPED](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1382)，M 标志位。M = 1 表示当前 chunk 是通过 mmap() 分配的，此时另两个标志位无效。
- [NON_MAIN_ARENA](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1391)，A 标志位。A = 1 表示当前 chunk 是从 non-main arena 分配出去的。

fd 和 bk 指针将空闲 chunk 组织成循环双链表，按大小排序。对于 large chunk，还会启用 fd_nextsize 和 bk_nextsize 字段，另外维护一条循环双链表，chunk 大小沿 bk_nextsize 方向严格递增。这四个字段在已分配 chunk 中不使用，存用户数据。

### bin
chunk 被释放后会先放在相应的箱子（bin）中，即通过 fd 和 bk 指针组织成循环双链表。下次用户请求内存时，如果 bin 中有可用 chunk，则直接返回该 chunk，不经过系统调用。bin 通过缓存空闲 chunk 减少了系统调用的次数。

从 bin 中取 chunk 的基本原则是 best fit。在此前提下，small chunk 采用先进先出：沿 fd 方向插入，沿 bk 方向取出。large chunk 按大小排序，大小相同则后进先出：某尺寸的第一个 chunk 到达后，链入 nextsize 链表，作为头节点，通过 fd/bk 维护一条同尺寸 chunk 链表；之后该尺寸的 chunk 在这个小的同尺寸链表上后进先出，不接入 nextsize 链表；头节点最后被使用。

bin 一共 127 个（bin 0 不存在）：

- bin 1 为 **unsorted bin**，每个空闲 chunk 在进入 small/large bin 前先放进 unsorted bin。等下次 malloc() 查找 unsorted bin 时，会把遍历到的 chunk 相应归入 small/large bin；
- bin 2 - bin 63 为 **small bin**，存放 **small chunk**，每个 bin 的 chunk 大小固定。bin x 存放大小为 x 个双字的 chunk。这种分法不会有遗漏，因为 chunk 最小两个双字且双字对齐；
- bin 64 - bin 126 为 **large bin**，存放 **large chunk**，最小为 64 个双字，每个 bin 的 chunk 大小在特定区间内。区间长度指数增大，对应的 bin 的数量指数减少。具体分法参见 [largebin_index_64](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1587) 宏；
- bin 127 不使用，始终为空。

bin 表示为指针数组 [bins](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1831)，这里 [NBINS](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1555) 宏取值 128，[mchunkptr](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1102) 为 chunk 指针：
```c
  // two chunk pointer ('fd' and 'bk') for each bin
  mchunkptr bins[NBINS * 2 - 2];
```
头节点只作索引用，不存数据，保存 fd 和 bk 即可。

使用 [bin_at](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1520) 宏取出目标 bin 的头节点，[mbinptr](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1517) 也是 chunk 指针，这里可以看出 bin 0 不存在：
```c
#define bin_at(m, i) \
  (mbinptr) (((char *) &((m)->bins[((i) - 1) * 2])) \
             - offsetof (struct malloc_chunk, fd))
```

### fastbin
默认情况下，fastbin 负责管理大小不超过 8 个双字（[DEFAULT_MXFAST](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L855)）的 chunk。这一参数记录在全局变量 [global_max_fast](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1765) 中，可通过 [set_max_fast](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1776) 宏更改，但上限为 10 个双字（[MAX_FAST_SIZE](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1731)）。

对于刚刚释放的 chunk，如果其大小在 fastbin 管理范围内，就先放入 fastbin。下次落在 fastbin 范围内的 malloc() 会优先查找 fastbin。和 bin 不同，fastbin 使用单链表，后进先出，总是在头部插入和取出。

fastbin 从 0 开始索引。每个 fastbin 的 chunk 大小固定，fastbin x 存放大小为 x + 2 个双字的 chunk。这样最多只用定义 9 个 fastbin，默认情况则只用得到前 7 个。

fastbin 表示为 [fastbinsY](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1822)，这里 [NFASTBINS](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1733) 宏展开为 9，[mfastbinptr](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1722) 为 chunk 指针：
```c
  mfastbinptr fastbinsY[NFASTBINS];
```

fastbin 中的 chunk 视作已分配，inuse 位不会清空。调用 [malloc_consolidate()](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L4671) 可合并 fastbin 中的 chunk 并将其放入 unsorted bin，其主要触发时机：

- free() 的 chunk 大小不低于 64K（[FASTBIN_CONSOLIDATION_THRESHOLD](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1746)）；
- malloc() 请求 large chunk，准备查 unsorted bin 之前；
- malloc() 请求 samll chunk，一路查到 top chunk 都不够用，而 fastbin 中仍有 chunk 时。

### binmap
binmap 是 bin 的位图，指示 bin 中是否有 chunk，为 0 表示一定空，为 1 表示可能不空。其表示为 uint 数组 [binmap](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1834)，但每个 uint 只使用低 32 位。

引入 binmap 是为了遍历时能快速跳过空 bin，只要没有 false negative 就行，而对 false positive 有一定容忍度。因此向 bin 中放入 chunk 后需要立即设置对应 bit 为 1，而从 bin 中取走最后的 chunk 时，不必立即清空对应 bit，推迟到之后遍历时再进行，更有效率。

### top chunk
top chunk 为一 chunk 指针，指向堆区最顶端（地址最高）的 chunk，记录在 [top](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1825) 指针中。top chunk 不会放在任何 bin 中，包括 fastbin 和 tcache。

malloc() 时，如果其他 chunk 都不满足，会尝试从 top chunk 分配。如果 top chunk 也不够用，会先调用 [sysmalloc()](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L2411) 向系统请求内存，增加其大小，再从 top chunk 分配。top chunk 自然也是空闲 chunk，因此其 P 位永远为 1。

free() 时，如果发现 top chunk 太大，会考虑削减其大小，削下来的内存以页为单位还给系统。削减后 top chunk 的可分内存（即减去一个最小 chunk 后的内存）不会超过一页。

top chunk 的初始化发生在首次调用 sysmalloc() 时。在此之前它被设置为 [initial_top](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1680)，指向 unsorted bin 头节点对应的假 chunk。这么做的好处是不用在 malloc() 中特判 top chunk 是否存在。这个方法可行，因为这个假 chunk 的 size 字段（即 bins 的前一个字，last_remainder）在首次调用 sysmalloc() 时一定为 0。

### last remainder
每次 malloc() 时，选中的 chunk 不一定是 exact fit，可能比需要的更大。这时会进行 chunk split，即从 chunk 中分出一个 exact fit，用来返回给用户。剩下的部分就叫作 remainder，会被立即放入 unsorted bin，唯一的例外是 top chunk，其产生的 remainder 将作为新的 top chunk 继续独立存在，不进入任何 bin 中。chunk split 的前提是 remainder 满足 chunk 的大小要求，即至少为两个双字。

last remainder 记录最近一次因请求 small chunk 而产生的 remainder，维护在 [last_remainder](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1828) 指针中：
```c
  /* The remainder from the most recent split of a small request */
  mchunkptr last_remainder;
```

每次请求 small chunk 查找到 unsorted bin 时，如果其中只剩 last remainder，那么只要它可以 split，就会进行 split 以响应请求，产生的 remainder 将成为新的 last remainder。该机制下，连续请求 small chunk 会产生一个典型的分配序列：第一次请求选中某个 large chunk，发生 split，更新 last remainder，接下来的请求一直从 last remainder 分配，直到其耗尽，即小到无法 split。

### arena
分配区（arena）要描述的就是 [平坦地址空间模型](todo) 下的堆。它表示为结构体 [malloc_state](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1809)，在其中记录堆管理需要的数据结构，包括上面提到的 bins、fastbinsY、binmap、top 和 last_remainder。内存分配就是通过操作这些数据结构，用最快的速度，找出最合适的 chunk。

由于一个进程只有一个堆，为了更好地支持多线程，考虑通过 mmap() 为每个子线程分配一个模拟堆，以此避免太激烈的资源争用。这样单线程的话就一个 arena，即 **main arena**，对应于进程堆。而多线程的话，会在 main arena 外创建若干 **non-main arena**，对应于为子线程分配的模拟堆。

如果线程很多，没必要为每个线程都创建 arena。因为就算是多核，就算开了超线程，能同时运行的线程数也有限，远小于能创建的线程数。为此有必要设定一个上限，在 arena 数量达到上限之前，新线程可以创建新的 arena 并独占之（访问仍需加锁 [mutex](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1812)，竞争可能在这之后发生）。达到上限之后，新线程将复用已有的 arena。实践中，这一上限值被设定为 CPU 核心数的 8 倍（[NARENAS_FROM_NCORES](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1924)）。

> 问题：arena 对锁的处理在 malloc() 和 free() 时不一样：_int_free() 有一个 have_lock 参数，为什么不统一先加锁？[__libc_free() 调用它的时候确实也没有加锁](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L3309)。

所有 arena 通过 [next](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1837) 指针构成循环单链表，头节点为 [main_arena](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1896)，对应锁为 [list_lock](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/arena.c#L97)。所有没有绑定线程的 non-main arena 通过 [next_free](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1841) 指针构成单链表，头节点为 [free_list](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/arena.c#L83)，对应锁为 [free_list_lock](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/arena.c#L81)。绑定线程数维护在 [attached_threads](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1846) 变量中。

所谓绑定，就是线程将其持有的线程局部变量 [thread_arena](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/arena.c#L74) 设置为某个特定的 arena，之后该线程的分配请求都会在该 arena 上进行。绑定 main arena 的线程分配失败后，会改绑一个 non-main arena 并尝试再次分配。线程退出时会在 [__malloc_arena_thread_freeres()](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/arena.c#L996) 中解绑 arena。线程绑定 arena 的逻辑如下：

-  主线程在 [ptmalloc_init()](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/arena.c#L324) 时绑定 main_arena；
-  子线程首次 malloc()，或绑定 main_arena 的线程分配失败需要改绑时，优先绑定 free_list 中的 arena；
-  若 free_list 为空，则考虑调用 [_int_new_arena()](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/arena.c#L736) 创建一个新的 arena 并绑定之;
-  若 arena 数量已达到上限，则调用 [reused_arena()](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/arena.c#L862) 从静态变量 [next_to_use](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/arena.c#L866) 开始遍历 arena 链表，绑定碰到的第一个未加锁 arena。next_to_use 从 main_arena 开始，在 arena 链表上循环移动，从而将新线程均匀地绑定到这些已有的 arena 上。

top chunk 内存不够用时会向系统请求内存。main arena 拥有整个进程堆，使用 sbrk() 扩展之即可。sbrk() 可能失败，如遇到 mmap() 区或 OS 地址空间不平坦，这时会 mmap() 一段内存作为新的 top chunk，大小至少 1M（[MMAP_AS_MORECORE_SIZE](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L556)），并设置 [NONCONTIGUOUS_BIT](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1757) 位（记录在 [flags](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1815) 中）将 main arena 标识为不连续。

而 non-main arena 一开始就用的是 mmap() 来的模拟堆，空间比进程堆小得多，也无法连续扩展，因此需要更精细的数据结构和管理算法。思路是一个模拟堆耗尽后再申请一个，新的模拟堆空闲后将其删除。模拟堆用 [_heap_info](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/arena.c#L53) 结构体描述，其所属的 arena 由 [ar_ptr](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/arena.c#L55) 指针记录。属于同一 arena 的模拟堆通过 [prev](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/arena.c#L56) 指针组织成单链表。主要操作如下：

- 初始化：使用 [new_heap()](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/arena.c#L503) 来 mmap() 新模拟堆。每个分配 64M（[HEAP_MAX_SIZE](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/arena.c#L32)），最底部放 _heap_info 结构体，必须对齐 HEAP_MAX_SIZE，这样在释放 chunk 时可以通过 [arena_for_chunk](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/arena.c#L129) 宏快速找到对应的 arena。要保证 64M 对齐，需要每次先 mmap() 双倍内存 128 M，从中挑出 64M 对齐区域，再 munmap() 剩余的。堆的当前大小记录在 [size](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/arena.c#L57) 字段，始终按页对齐，初始化为 _heap_info 加上所需 top chunk 的大小。
- 扩展堆：和 main arena 一样，堆的扩展发生在 top chunk 不够用时。当前堆 64 M 的空间够用时，扩展只需调用 [grow_heap()](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/arena.c#L581) 增加 size，并相应更新 top chunk 即可。如果不够用，则调用 new_heap() 分配一个新堆，移动 top chunk 到新堆，让 prev 指向旧堆即可。
- 收缩堆：每次释放 non-main arena 中的 chunk 时，都会调用 heap_trim() 尝试移除空闲的模拟堆。即从当前堆开始沿 prev 扫描，如果整个堆都是 top chunk，就移除当前堆，在 prev 中重构 top chunk，直到 prev 的空闲空间不足。然后像 systrim() 那样收缩 top chunk。

arena 还持有 [have_fastchunks](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1819) 标志，指示 fastbin 中有无 chunk（可能不准，但不影响正确性），以减少对 malloc_consolidate() 的调用。在 [system_mem](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1849) 中记录该 arena 拥有的系统内存大小，即堆/模拟堆的当前大小，不包括模拟堆的未使用空间，不包括 mmapped chunk。

### tcache
线程 cache（tcache）是 17 年 [glibc 2.26 引入](https://sourceware.org/git/?p=glibc.git;a=commit;h=d5c3fafc4307c9b7a4c7d5cb381fcdbfad340bcc) 的新机制，主要特性是引入线程专属 cache 降低锁开销，同时主动缓存与近期 malloc() 同尺寸的 chunk，以更好地利用局部性，提升分配性能。

tcache 由结构体 [tcache_perthread_struct](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L3071) 描述，维护了 64 个 bin（[TCACHE_MAX_BINS](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L309)）：
```c
typedef struct tcache_perthread_struct
{
  char counts[TCACHE_MAX_BINS];
  tcache_entry *entries[TCACHE_MAX_BINS];
} tcache_perthread_struct;
```
其中 counts 记录每个 bin 当前包含的 chunk 数。[tcache_entry](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L3059) 为单链表节点，成员 key 指示 chunk 是否在 tcache 中，在则指向 tcache，否则为空，仅用于检测 double free：
```c
typedef struct tcache_entry
{
  struct tcache_entry *next;
  /* This field exists to detect double frees.  */
  struct tcache_perthread_struct *key;
} tcache_entry;
```

tcache 的 bin 和 fastbin 有很多相似点：单链表，后进先出，bin x 存 x + 2 个双字的 chunk，inuse 位为 1，从 0 开始索引。

不同在于：第一，tcache bin 数量更多，因此管理范围更大，不超过 65 个双字的 chunk 都可以进 tcache；第二，tcache bin 容量有限，每个 bin 最多放 7 个 chunk（[TCACHE_FILL_COUNT](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L328)）；第三，链入 tcache bin 的不是 chunk 地址，而是将要返回给用户的地址 mem，换句话说，tcache_entry 的 next 指针实际上是复用 chunk 的 fd 字段；第四，fastbin 中的 chunk 有两条离开路径，返回给用户，或者通过 malloc_consolidate() 进入 unsorted bin，但 tcache 中的 chunk 只会返回给用户，不会进入下级结构。

tcache 的初始化通过 [tcache_init()](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L3142) 进行，通常发生在第一次 malloc() 时。对 tcache 的操作包括 [tcache_put()](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L3083) 和 [tcache_get()](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L3099)，两者均不进行完整性检查，索引的合法性以及对应 bin 的未满和非空由调用者保证。

malloc() 时最先检查 tcache，如果命中（exact fit）就直接返回，无需任何加锁。tcache 的 chunk 有两大来源，除了接收释放的 chunk 外，还会在下级结构命中时主动缓存同尺寸 chunk，当然前提是对应 bin 未满：

- fastbin/small bin 命中，在返回 chunk 给用户前，将该 fastbin/small bin 下其他 chunk 放入 tcache；
- unsorted bin 命中，如果大小落在 tcache 范围内，不再直接返回，而是继续遍历 unsorted bin，以将所有同尺寸 chunk 放入 tcache，遍历结束后再从 tcache 返回一个该大小的 chunk。

## malloc()
malloc() 接受一个 size_t 的参数 size，返回一个 void* 指针，指向一片大小至少为 size 的连续内存，或者为空，表示分配失败。对于 malloc(0)，可以返回空指针，也可以返回一个有效但不希望被解引用的指针，取决于具体实现。glibc 的选择是返回一个最小 chunk，对应用程序来说，此时的可用内存为三个字，包括了 nextchunk 的第一个字。

malloc() 的 glibc 定义是 __libc_malloc()，主要工作在其调用的 _int_malloc() 中进行，后者在内存不足时调用 sysmalloc() 向系统请求内存。

### [__libc_malloc()](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L3192)
__libc_malloc() 先检查 tcache，如果不命中，则对 arena 进行必要加锁并调用 _int_malloc()：

1. 通过 [checked_request2size()](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1349) 调用 [request2size](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1328) 宏将用户请求的内存大小 bytes 转成所需 chunk 的大小 tbytes，即加一个字然后向上对齐双字，至少为 MINSIZE。加字是为了放 chunk header，只加一个是因为 nextchunk 会提供它的第一个字。转换前检查 bytes，如果超过 [PTRDIFF_MAX](https://elixir.bootlin.com/glibc/glibc-2.33/source/stdlib/stdint.h#L210)，返回 NULL，错误号 [ENOMEM](https://elixir.bootlin.com/glibc/glibc-2.33/source/sysdeps/mach/hurd/bits/errno.h#L48)；
2. 检查 tcache 是否包含 tbytes 大小的 chunk，命中则取出 chunk 直接返回即为 mem；
3. 如果是单线程（[SINGLE_THREAD_P](https://elixir.bootlin.com/glibc/glibc-2.33/source/sysdeps/unix/sysv/linux/single-thread.h#L33)），在 main_arena 上调用 _int_malloc()，返回其返回值；
4. 如果是多线程，调用 [arena_get](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/arena.c#L113) 宏对绑定的 arena 加锁（未绑定则先绑定）并调用 _int_malloc()。如果其分配失败，调用 [arena_get_retry()](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/arena.c#L977) 尝试在另一个 arena 上分配：如果是在 non-main arena 上失败，表明 mmap() 空间不足，但 sbrk() 可能还有空间，因此尝试在 main arena 上分配；反之则表明 sbrk() 空间不足，改绑一个 non-main arena，尝试通过 mmap() 分配；

_int_malloc() 返回 mem，其对应的 chunk 要么来自 mmap()，要么来自被选中的 arena。

### [_int_malloc()](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L3727)
_int_malloc() 将在指定的 arena 中选出一个可用内存不小于 bytes 字节的 chunk，返回其对应的 mem。查找顺序大体为 fastbin 到 small bin 到 unsorted bin 到 large bin 到 top chunk，如果 top chunk 也不够用，则请求系统内存。下面提到的返回某个 chunk 都是指返回其对应的 mem：

1. 通过 checked_request2size() 将 bytes 转换成所需 chunk 的大小 nb，同 __libc_malloc() 的第 1 步；
2. 如果 arena 为 NULL，**调用 sysmalloc()**，返回其返回值；
3. 如果 nb 落在 fastbin，且对应 fastbin 非空，**返回第一个**，缓存其他 chunk 到 tcache；
4. 如果 nb 落在 small bin，且对应 small bin 非空，**返回 bk 方向的第一个**，缓存其他 chunk 到 tcache；
5. 如果 nb 落在 large bin，且 have_fastchunks，调用 malloc_consolidate()；
6. 沿 bk 方向扫描 unsorted bin，将 chunk 归入对应的 small/large bin：
    1. **考虑返回 last remainder**；
    2. 如果 chunk 大小为 nb，能缓存就缓存到 tcache 然后 continue，**不能就直接返回该 chunk**；
    3. 根据 chunk 大小，将其归入对应的 small bin 或 large bin；
    4. 如果检查过的 chunk 数达到 10000（[MAX_ITERS](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L4122)），结束扫描；
7. 如果上一步在 tcache 中缓存了 nb 大小的 chunk，**取出一块返回**；
8. 如果 nb 落在 large bin，且对应 large bin 中的最大 chunk 大于 nb，沿 bk_nextsize 方向扫描，**返回 best fit**，有同尺寸则后进先出；
9. 从 nb 对应 bin 的下一个 bin 开始扫描，**返回第一个非空 bin 中 bk 方向的第一个 chunk**；
10. **考虑返回 top chunk**；
11. 如果 have_fastchunks，调用 malloc_consolidate() 并转至第 6 步再次扫描 unsorted bin；
12. **调用 sysmalloc()**，返回其返回值。

第 6(i) 步中返回 last remainder 需要满足三个条件：第一，nb 落在 small bin；第二，unsorted bin 中只有 last remainder 这一个 chunk；第三，其大小大于 nb + MINISIZE（等于？）。如果均满足，就将其 split，返回 nb 大小的 chunk，remainder 作为新的 last remainder 继续留在 unsorted bin 中。任一条件不满足都会导致该 last remainder 被归入 small/large bin，之后这一步始终不会执行直到 last remainder 被更新。

第 6(ii) 步缓存的条件是 nb 落在 tcache 范围且对应的 tcache bin 未满。continue 而不直接返回是为了缓存更多潜在的 nb 大小 chunk。

第 8 步和第 9 步通过 [unlink_chunk()](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1605) 从 bin 中取出 chunk。返回 chunk 前需要考虑 split（在这之前返回的，除了 last remainder 外都是 exact fit）：如果大于等于 nb + MINSIZE 就 split，只返回 nb 大小的 chunk，remainder 放入 unsorted bin；否则返回整个 chunk。第 9 步中如果发生了 split 且请求的是 small chunk，就会更新 last remainder，这是 last remainder 的唯一来源。

第 9 步返回的 chunk 必是 non-exact but best fit。如果请求的是 small chunk，那么 exact fit 必在第 4 步或 第 6(ii)/7 步被返回，如果是 large chunk，exact fit 必在第 6(ii)/7 步或第 8 步被返回。

第 10 步要返回 top chunk 必须先 split，只返回 nb 大小的 chunk，remainder 将成为新的 top chunk。不能 split 的话（小于 nb + MINSIZE）就跳过这一步。

第 11 步最多执行一次，因为 malloc_consolidate() 会清空 have_fastchunks 标志。这一步实际上是对第 5 步的补充：请求的是 large chunk 就在第 5 步整理 fastbin；是 small chunk 则延迟到第 11 步整理。这个拆分是为了减少整理 fastbin 的次数：执行到第 11 步意味着，这个对 small chunk 的请求，bin 中现有 chunk 满足不了，top chunk 也满足不了，必须把 fastbin 的内存也利用起来，这种情况很少见。

### [sysmalloc()](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L2411)
sysmalloc() 通过 sbrk() 或 mmap() 以页为单位向系统请求内存，返回一个合法 chunk 对应的 mem：

1. 如果 arena 为 NULL，或 nb 大于等于 [mmap_threshold](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1858)，**使用 mmap() 分配一个孤立 chunk 返回**；
2. 如果是 non-main arena：
    1. 如果当前模拟堆还有空间，扩展 top chunk 到 nb + MINSIZE；
    2. 通过 mmap() 分配一个新的模拟堆，在其中初始化新的 top chunk，大小为 nb + MINSIZE；
    3. 如果分配失败，**使用 mmap() 分配一个孤立 chunk 返回**，同第 1 步；
3. 如果是 main arena：
    1. 使用 sbrk() 扩展进程堆，扩展的内存全部给 top chunk；
    2. 如果 sbrk() 失败，使用 mmap() 分配一个新的 top chunk；
4. 如果 top chunk 可以 split，**分出 nb 大小的 chunk 返回**，remainder 作为新的 top chunk；
5. 系统可用内存不足，设置错误号 ENOMEM，返回 0。

第 1 步和第 2(iii) 步中通过 mmap() 分配了一个页对齐的孤立 chunk，它没有 nextchunk 的第一个字可复用，至少要比 nb 多分配一个字。该 mmapped chunk 会设置 M 标志位，释放时直接 munmap() 即可。

mmap_threshold 初始为最小值 128K（[DEFAULT_MMAP_THRESHOLD_MIN](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L964)），动态设置为释放的最大 mmapped chunk 的大小，最大不超过 32M（[DEFAULT_MMAP_THRESHOLD_MAX](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L975)）。

第 2(ii) 和 3(ii) 步构造完新的 top chunk 后，会在旧的 top chunk 尾部插入 fencepost，然后剩余部分如果不低于 MINSIZE，将通过 _int_free() 释放。fencepost 属于特殊 chunk，P 位为 1，inuse 位也为 1，用于防止合并。它占用两个双字的空间，但大小设置为一个双字，这样就能在第二个双字中存储 inuse 位。

## free()
free() 接受一个 void* 指针，释放该指针指向的内存，无返回值。free(0) 不进行操作。和 malloc() 对称，free() 的 glibc 定义为 __libc_free()，主要工作在其调用的 _int_free() 中进行。后者会考虑收缩 top chunk，归还一定内存给系统，在 main/non-main arena 上分别通过 systrim() 和 heap_trim() 进行。

### [__libc_free()](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L3259)
检查传入的指针 mem，若为 0 则返回；将 mem 转成对应的 chunk，如果是 mmapped chunk，考虑更新 mmap_threshold 为 chunk 大小，然后通过 [munmap_chunk()](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L2973) 调用 munmap() 释放该 chunk；如果不是，通过 _int_free() 释放。

### [_int_free()](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L4377)
1. 如果 chunk 的大小 size 落在 tcache，且对应 bin 未满，**放入 tcache**，返回；
2. 如果 size 落在 fastbin，**放入 fastbin**，返回；
3. 如果不是 mmapped chunk：
    1. 考虑后向合并，即如果 prevchunk 空闲，合并之；
    2. 考虑前向合并，即如果 nextchunk 空闲，合并之，否则清空其 P 位；
    3. 新 chunk **沿 fd 方向插入 unsorted bin**，或者，**成为新的 top chunk**；
    4. 如果 size 不小于 64K，通过 systrim()/heap_trim() 交还系统内存；
4. 如果是 mmapped chunk，通过 munmap_chunk() 释放 chunk。

第 3(i)/(ii) 步的 chunk 合并：将新 chunk 的大小设为两者之和，P 位设为 1，然后 unlink_chunk() 从 bin 中摘除新 chunk。

第 3(iii) 步判断 nextchunk 是不是 top chunk 即可。

第 3(iv) 步，为了回收尽可能多的内存，systrim()/heap_trim()前先调用 malloc_consolidate() 释放 fastbin 中的 chunk，步骤同 _int_free()：后向合并，前向合并，放入 unsorted bin 或者成为新的 top chunk。另外，调用 systrim() 需要 top chunk 的大小达到 [trim_threshold](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1856)，但调用 heap_trim() 没有此条件，因为即使 top chunk 很小，heap_trim() 也有机会释放整个模拟堆的空间。

trim_threshold 初始为 128K（[DEFAULT_TRIM_THRESHOLD](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L922)），等于 mmap 阈值的初始值。更新 mmap 阈值时，会更新 trim 阈值为其两倍大小。

### [systrim()](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L2906)/[heap_trim()](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/arena.c#L646)
和申请时一样，向系统归还内存也是以页为单位。systrim() 和 heap_trim() 会将 top chunk 的可分内存收缩至不超过一页，即 top chunk 减去一个 MINSIZE 后的大小不会超过一页，边界条件下，top chunk 的大小会被压缩至 MINSIZE 加一个双字：
```c
release_size = ALIGN_DOWN(top_size - MINSIZE - 1, pagesize)
```
systrim() 总是通过 sbrk() 归还内存，不特殊处理 main arena 不连续的情形。

heap_trim() 从当前模拟堆开始，沿 prev 回收完全空闲的模拟堆，移动 top chunk 到 prev，直到其尾部的空闲内存小于 MINSIZE + pagesize。然后对 top chunk 进行收缩。heap_trim() 回收模拟堆的方式为直接 munmap()。但收缩 top chunk 只是将内存通过 [shrink_heap()](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/arena.c#L609) 调用 madvice() 标记为 MADV_DONTNEED，这些内存不能直接 munmap() 还给系统，因为之后 top chunk 还会扩展回来。