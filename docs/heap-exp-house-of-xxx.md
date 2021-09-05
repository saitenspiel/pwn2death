# 堆利用：House Of 系列
参见 [ctf-wiki](https://ctf-wiki.org/pwn/linux/user-mode/heap/ptmalloc2/introduction/)、[how2heap](https://github.com/shellphish/how2heap)。以 [ptmalloc2](/ptmalloc2) 为例。

分配和释放以 chunk 为单位。malloc() 时会先补上 chunk header 的开销：加上一个字然后向上对齐双字。换言之，malloc(0x10) 和 malloc(0x18) 在分配器看来一样，都是分配至少 0x20 大小的 chunk。相应地，free() 的地址也会先转成对应 chunk 的起始地址。为了说明方便，先引入两个记号：

- malloc_chunk(n) 表示请求 n 字节 chunk，对应到 malloc() 请求 n-0x10 到 n-8 字节都可；
- free_chunk(p) 表示释放起始地址为 p 的 chunk，对应 free(p+0x10)。

将 chunk 放进指定结构的通用方法（dw 指双字，16 字节）：

- tcache：释放 chunk（<=0x410，65 dw）；
- fastbin：填满对应的 tcache bin，释放 chunk（<=0x80，8 dw）；
- unsorted bin：填满对应的 tcache bin，释放 chunk（>0x80）；
- small bin：unsorted bin 放入 chunk（<=0x3F0，63 dw），请求 unsorted bin 无法满足的 chunk；

## House of Einherjar (latest)
条件：地址 addr 和 addr+8 可控，低地址 prevaddr 起连续三个字可控。

能力：拿到 prevaddr 到 addr，加 addr 之后的一段内存。

利用：在 addr 处伪造 P 位为 0 的 chunk，在 prevaddr-8 处伪造可以绕过 unlink 的 prevchunk，释放 chunk，后向合并 prevchunk，得到的新 chunk 进入 unsorted bin，再次申请即可拿到该 chunk：

1. 写 addr 为 chunk->prev_size = addr-prevaddr+8；写 addr+8 为 chunk->size，令 P 位为 0；
2. 在 prevaddr 处依次写入：
    1. prevchunk->size = chunk->prev_size；
    2. prevchunk->fd = prevchunk-0x18；
    3. prevchunk->bk = prevchunk-0x10；
3. 填满 chunk->size 对应的 tcache bin；
4. free_chunk(addr);
5. malloc_chunk(chunk->size + chunk->prevsize)。

第 1 步写 addr+8 主要是为了清 P 位，可只写最低字节为 0x00。利好 off-by-one/null：如果能拿到 addr 真正的 prevchunk，那 addr 直接可控，加上 off-by-one/null 正好写 P 位。

最好让 chunk->size 超出 fastbin 范围。否则第 4 步后还需 malloc_consolidate() 才能完成合并：

- 请求一个 large chunk，如果合并后的 chunk 就是，那直接第 5 步即可；
- 请求一个 top chunk 之前的结构都不能满足的 small chunk；
- 释放一个不低于 64K 的 chunk。

第 2 步是为了绕过 unlink 的完整性检查：

- [chunk->size 等于 nextchunk->prev_size](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1607)：
```c
  if ( chunk->size != nextchunk->prev_size )
    malloc_printerr ("corrupted size vs. prev_size");
```
- [chunk->fd->bk 和 chunk->bk->fd 都等于 chunk](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L1613)：
```c
  if ( chunk->fd->bk != p || chunk->bk->fd != p )
    malloc_printerr ("corrupted double-linked list");
```
有两个变式：

- 如果 size 受限，但另有某 midaddr 可控，可以写 size 和 midaddr 为 midaddr-&size+8，即在 midaddr 处伪造 prevchunk 的 nextchunk；
- 2(ii)/(iii) 有副作用 *prevchunk = prevchunk-0x18。另一绕过方式为 fd = bk = prevchunk，可 nop 掉 unlink 对 fd->bk 和 bk->fd 的修改。

## House of Force (<2.29)
条件：top chunk 的 size 字段可控。

能力：拿到任意地址。

利用：确定 top chunk 到目标内存的偏移 offset，以及目标内存长度 length。令 top chunk 的 size 为一特大值，至少不小于 offset + length + MINSIZE；申请 offset 大小的内存，top chunk 会被偏移到目标地址；再申请 length 大小的内存，即可拿到目标内存：

1. top->size = -1；
2. malloc_chunk(offset)；
3. malloc_chunk(length)。

glibc 2.29 引入 [patch](https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=30a17d8c95fbfb15c52d1115803b63aaa73a285c)：[top chunk 的大小不得超过持有的系统内存量](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L4329)：
```c
  if ( top->size > av->system_mem )
    malloc_printerr ("malloc(): corrupted top size");
```

## House of Lore (?)
条件：chunk0 释放进 small bin 后 bk 字段仍然可控，地址 addr 和 addr+8 可控，任意一段内存 m 可控。

能力：拿到 addr 处的一小段内存。

利用：在 addr-0x10 处伪造 chunk1，在内存区域 m 伪造 chunk2-7。控制 chunk0 的 bk 指向 chunk1。从 small bin 中拿走 chunk0，chunk1-7 被自动填入 tcache，再连续申请即可从 tcache 中拿到 chunk1：

1. 写 addr 为 chunk1->fd = chunk0，写 addr+8 为 chunk1->bk = chunk2；
2. 在 m 处伪造 chunk2-7，通过 bk 依次链接；
3. 将 chunk0 放入 small bin：
    1. 填满 chunk0->size 对应的 tcache bin；
    2. free_chunk(chunk0)，进入 unsorted bin，要求 chunk0->size 不落在 fastbin；
    3. 申请一个 unsorted bin 无法满足的 chunk，chunk0 进入 small bin；
4. 写 chunk0->bk 为 chunk1，即 addr-0x10；
5. 取空 chunk0->size 对应的 tcache bin；
6. malloc_chunk(chunk0->size)，返回 chunk0，chunk1-7 进入 tcache；
7. malloc_chunk(chunk0->size) * 7。

chunk1-7 不需要伪造 size，只要链在 chunk0 下，就会被看作大小相同，并不额外检查 size 字段。

chunk1 伪造 fd 是为了第 6 步取 chunk 0 时绕过 small bin 的完整性检查，chunk2-7 不需要。但是，如果没有开启 tcache，chunk1 只能从 small bin 分配，需要伪造 chunk2->fd = chunk1。

完整性检查：

- [victim->bk->fd 等于 victim](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L3866)：
```c
  if ( victim->bk->fd != victim )
    malloc_printerr ("malloc(): smallbin double linked list corrupted");
```

## House of Orange (?)
条件：top chunk 的 size 字段可控。

能力：无 free() 释放 top chunk。

利用：修改 top->size，然后触发 sysmalloc() 扩展 top chunk。此时旧堆顶，即 sbrk() 的返回值，将不等于 top+top->size，按发生外部 sbrk()，无法连续扩展 top chunk 处理：在旧堆顶附近构造新 top chunk，并将旧 top chunk 通过 _int_free() 释放。

House of Orange 的用途是在没有 free() 时，通过 malloc() 释放一个 chunk 到 tcache/fastbin/bin，以便后续利用。该原语来自 Hitcon'16 的 [house-of-orange](https://github.com/ctfs/write-ups-2016/tree/master/hitcon-ctf-2016/pwn/house-of-orange-500)，预期解为释放 chunk 到 unsorted bin，再 unsorted bin attack 劫持 _IO_list_all，然后通过 malloc_printerr() 触发 [FSOP](/todo) 完成利用。

## House of Spirit (latest)
条件：地址 addr 可控（，某较高地址 nextaddr 的值满足约束）。

能力：拿到 addr+8 处（到 nextaddr 之前）的一小段内存。

利用：在 addr-8 处伪造特定大小的 chunk，将其释放进 tcache（fastbin），再申请该大小的内存即可拿到整个伪造 chunk：

1. 写 addr 为 chunk->size（= nextaddr-addr；写 nextaddr 为 nextchunk->size）；
2. free_chunk(addr-8)；
3. malloc_chunk(chunk->size)。

House of Spirit 最早是针对 fastbin 的利用，需要绕过 nextchunk->size 的完整性检查。后来引入的 tcache 不检查这一项，不用再考虑 nextaddr 的约束，当然 chunk->size 得落在 tcache 范围（<=0x410）。

完整性检查：

- [chunk 双字对齐，nextchunk 不溢出 uint 的地址空间](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L4394)：
```c
  if ( (uintptr_t) chunk > (uintptr_t) -chunk->size || !aligned (chunk) )
    malloc_printerr ("free(): invalid pointer");
```
- [chunk->size 至少为两个双字，且双字对齐](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L4399)：
```c
  if ( chunk->size < MINSIZE || !aligned (chunk->size) )
    malloc_printerr ("free(): invalid size");
```
- [fastbin only] [nextchunk->size 大于一个双字，小于 system_mem](https://elixir.bootlin.com/glibc/glibc-2.33/source/malloc/malloc.c#L4461)：
```c
  if ( nextchunk->size <= CHUNK_HDR_SZ || nextchunk->size >= av->system_mem )
    malloc_printerr ("free(): invalid next size (fast)");
```