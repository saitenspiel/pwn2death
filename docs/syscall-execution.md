# 执行系统调用
系统调用从用户态发起，在内核态执行，完成后又返回用户态。其执行可以分为四个阶段：

1. 在用户空间调用 libc 接口，libc 将其转换成 syscall 指令；
2. syscall 指令从用户态切换到内核态，转交控制权到内核；
3. 内核执行系统调用，完成后执行 sysret 指令；
4. sysret 指令从内核态切换回用户态，转交控制权到用户空间。

## 从 libc 到 syscall 指令
以 glibc 的 open() 函数为例。在用户空间，包含头文件 fcntl.h 即可使用 open() 函数，这个 open() 不是系统调用，是 glibc 对系统调用的封装。封装入口是 [__libc_open64()](https://elixir.bootlin.com/glibc/glibc-2.33/source/sysdeps/unix/sysv/linux/open64.c#L36)：
```c
int
__libc_open64 (const char *file, int oflag, ...)
{
  // ...
  return SYSCALL_CANCEL (openat, AT_FDCWD, file, oflag | EXTRA_OPEN_FLAGS,
                         mode);
}
```
进入 [SYSCALL_CANCEL](https://elixir.bootlin.com/glibc/glibc-2.33/source/sysdeps/unix/sysdep.h#L91) 宏，注意这里实际执行的系统调用不是 open()，而是 openat()。该宏负责处理异步线程取消：如果是单线程，直接执行系统调用；多线程则先启用异步取消模式。不管哪种情况，都进入 [INLINE_SYSCALL_CALL](https://elixir.bootlin.com/glibc/glibc-2.33/source/sysdeps/unix/sysdep.h#L88) 宏，该宏解析变长参数，提出参数个数 nr，最终执行到 [INLINE_SYSCALL](https://elixir.bootlin.com/glibc/glibc-2.33/source/sysdeps/unix/sysv/linux/sysdep.h#L42) 宏：
```c
#define INLINE_SYSCALL(name, nr, args...)				\
  ({									\
    long int sc_ret = INTERNAL_SYSCALL (name, nr, args);		\
    __glibc_unlikely (INTERNAL_SYSCALL_ERROR_P (sc_ret))		\
    ? SYSCALL_ERROR_LABEL (INTERNAL_SYSCALL_ERRNO (sc_ret))		\
    : sc_ret;								\
  })
```
进入 [INTERNAL_SYSCALL](https://elixir.bootlin.com/glibc/glibc-2.33/source/sysdeps/unix/sysv/linux/x86_64/sysdep.h#L234) 宏，其与体系结构相关，x64 下定义为：
```c
#define INTERNAL_SYSCALL(name, nr, args...)				\
    internal_syscall##nr (SYS_ify (name), args)
```
根据参数个数 nr 展开到相应宏，[SYS_ify](https://elixir.bootlin.com/glibc/glibc-2.33/source/sysdeps/unix/sysv/linux/x86_64/sysdep.h#L34) 宏会将 name 替换为对应的系统调用号。因为 openat() 是 4 个参数，所以进入 [internal_syscall4](https://elixir.bootlin.com/glibc/glibc-2.33/source/sysdeps/unix/sysv/linux/x86_64/sysdep.h#L302) 宏：
```c
#define internal_syscall4(number, arg1, arg2, arg3, arg4)		\
({									\
    unsigned long int resultvar;					\
    TYPEFY (arg4, __arg4) = ARGIFY (arg4);			 	\
    TYPEFY (arg3, __arg3) = ARGIFY (arg3);			 	\
    TYPEFY (arg2, __arg2) = ARGIFY (arg2);			 	\
    TYPEFY (arg1, __arg1) = ARGIFY (arg1);			 	\
    register TYPEFY (arg4, _a4) asm ("r10") = __arg4;			\
    register TYPEFY (arg3, _a3) asm ("rdx") = __arg3;			\
    register TYPEFY (arg2, _a2) asm ("rsi") = __arg2;			\
    register TYPEFY (arg1, _a1) asm ("rdi") = __arg1;			\
    asm volatile (							\
    "syscall\n\t"							\
    : "=a" (resultvar)							\
    : "0" (number), "r" (_a1), "r" (_a2), "r" (_a3), "r" (_a4)		\
    : "memory", REGISTERS_CLOBBERED_BY_SYSCALL);			\
    (long int) resultvar;						\
})
```
先根据 [调用约定](../todo) 传参到相应寄存器，然后通过 [内联汇编](../todo) 执行 syscall 指令。

## [syscall 指令](https://www.felixcloutier.com/x86/syscall)：ring3 到 ring0
用户态和内核态的切换，本质是 CPU 特权级的改变。syscall 指令会将 CPL 设置为 0，即切换到内核态。该指令还将根据 [MSR 寄存器](../register#msr) 更新 RIP，设置 CS 和 SS 这两个 [段寄存器](../register#_1)。具体操作如下：

1. 保存返回地址（即 syscall 下一条指令的地址，和 call 类似）到 RCX，保存 RFLAGS 到 R11；
2. 设置 RIP 为 IA32_LSTAR MSR 的值（wrmsr 指令保证这一定是个 [合规地址](../canonical-address)）；
3. 根据 IA32_FMASK MSR 清空 RFLAGS 的相应位；
4. 设置代码段寄存器 CS：
    - Selector = IA32_STAR[47:32] & FFFCH，与操作是为了设置 RPL 为 0；
    - Base = 0；Limit = FFFFFH，界限字段 20 位，4K 粒度下就是 4G 的段空间；
    - Type = 11，代码，可执行、可读，已被访问；
    - S = 1，非系统段；DPL = 0；P = 1；L = 1，64 位模式；D = 0；G = 1，4K 粒度；
5. CPL = 0，进入内核态；
6. 设置堆栈段寄存器 SS：
    - Selector = IA32_STAR[47:32] + 8，SS 的描述符紧跟着 CS；
    - Base = 0；Limit = FFFFFH，界限字段 20 位，4K 粒度下就是 4G 的段空间；
    - Type = 3，数据，可读、可写，已被访问；
    - S = 1；DPL = 0；P = 1；B = 1；G = 1。

涉及到的 IA32_{LSTAR，FMASK，STAR} 这三个 MSR 寄存器由内核负责初始化。调用栈为 [start_kernel()](https://elixir.bootlin.com/linux/v5.13/source/init/main.c#L875) -> [trap_init()](https://elixir.bootlin.com/linux/v5.13/source/arch/x86/kernel/traps.c#L1155) -> [cpu_init()](https://elixir.bootlin.com/linux/v5.13/source/arch/x86/kernel/cpu/common.c#L1945) -> [syscall_init()](https://elixir.bootlin.com/linux/v5.13/source/arch/x86/kernel/cpu/common.c#L1752)：
```c
void syscall_init(void)
{
    // IA32_STAR, for CS/SS.Selector
    wrmsr(MSR_STAR, 0, (__USER32_CS << 16) | __KERNEL_CS);
    // IA32_LSTAR, for RIP
    wrmsrl(MSR_LSTAR, (unsigned long)entry_SYSCALL_64);
    // ...
    // IA32_FMASK, for RFLAGS clearing
    wrmsrl(MSR_SYSCALL_MASK,
           X86_EFLAGS_TF|X86_EFLAGS_DF|X86_EFLAGS_IF|
           X86_EFLAGS_IOPL|X86_EFLAGS_AC|X86_EFLAGS_NT);
}
```
IA32_STAR 的低 32 位被设置为 0，高 32 位被设置为 __USER32_CS 和 __KERNEL_CS 的串联。前者用于 sysret 时 CS/SS 的设置，后者用于 syscall 时 CS/SS 的设置。

IA32_LSTAR 被设置为一段汇编指令 entry_SYSCALL_64 的地址，这就是系统调用的内核入口点。当 syscall 指令执行完成后，控制权将转移到这段汇编指令。

IA32_FMASK 设置了 X86_EFLAGS_IF 位，意味着 syscall 指令会清除 RFLAGS 的 IF 位，可屏蔽中断将被屏蔽。禁止中断重入的一个原因是 swapgs 指令不支持。

syscall 指令既不更新也不保存 RSP，这一任务将由内核完成。

## [entry_SYSCALL_64](https://elixir.bootlin.com/linux/v5.13/source/arch/x86/entry/entry_64.S#L87)
作为系统调用的内核入口，entry_SYSCALL_64 先进行一些准备工作，然后执行系统调用，最后使用 sysret 指令返回到用户态。

### 准备工作
1. 执行 swapgs 指令，交换 IA32_GS_BASE 和 IA32_KERNEL_GS_BASE 这两个 MSR 寄存器的值，让 GS 段指向内核 per-cpu 数据区域；
2. 保存 RSP，通过 [SWITCH_TO_KERNEL_CR3](https://elixir.bootlin.com/linux/v5.13/source/arch/x86/entry/calling.h#L199) 宏切换 CR3，让 RSP 指向内核栈：
```c
    // save RSP to scratch space
    movq	%rsp, PER_CPU_VAR(cpu_tss_rw + TSS_sp2)
    // switch CR3 to kernel page-dir base
    SWITCH_TO_KERNEL_CR3 scratch_reg=%rsp
    // point RSP to kernel stack
    movq	PER_CPU_VAR(cpu_current_top_of_stack), %rsp
```
[PER_CPU_VAR(var)](https://elixir.bootlin.com/linux/v5.13/source/arch/x86/include/asm/percpu.h#L14) 宏用于访问由内核管理的 per-cpu 数据，在 64 位多处理器下展开为 %gs:var。因此，使用该宏时必须保证 IA32_GS_BASE 指向内核 GS，即 swapgs 指令恰执行了奇数次。

3. 在栈上填充 [pt_regs](https://elixir.bootlin.com/linux/v5.13/source/arch/x86/include/asm/ptrace.h#L59)：
```x86asm
    pushq	$__USER_DS                          ; pt_regs->ss
    pushq	PER_CPU_VAR(cpu_tss_rw + TSS_sp2)   ; pt_regs->sp
    pushq	%r11                                ; pt_regs->flags
    pushq	$__USER_CS                          ; pt_regs->cs
    pushq	%rcx                                ; pt_regs->ip
    ; ...
    pushq	%rax                                ; pt_regs->orig_ax

	PUSH_AND_CLEAR_REGS rax=$-ENOSYS
```
这里 [PUSH_AND_CLEAR_REGS](https://elixir.bootlin.com/linux/v5.13/source/arch/x86/entry/calling.h#L100) 宏会依次入栈 RDI、RSI、RDX、RCX、RAX（实际入栈的是 $-ENOSYS，作为系统调用号非法时的返回值）、R8、R9、R10、R11、RBX、RBP、R12、R13、R14、R15。此时各寄存器的值已按 pt_regs 定义的顺序保存在栈上，之后通过 [寄存器同名宏](https://elixir.bootlin.com/linux/v5.13/source/arch/x86/entry/calling.h#L70) 可以方便地将其还原到寄存器中

### 执行系统调用
```x86asm
    movq	%rax, %rdi      ; %rdi = syscall number (1st arg)
    movq	%rsp, %rsi      ; %rsi = &pt_regs (2nd arg)
    call	do_syscall_64
```
[do_syscall_64()](https://elixir.bootlin.com/linux/v5.13/source/arch/x86/entry/common.c#L39) 即系统调用的执行函数：
```c
__visible noinstr void do_syscall_64(unsigned long nr, struct pt_regs *regs)
{
    // so the kernel stack base varies between syscalls
	add_random_kstack_offset();
    nr = syscall_enter_from_user_mode(regs, nr);
	//...
	if (likely(nr < NR_syscalls)) {
        // clamp 'nr' within [0, NR_syscalls)
		nr = array_index_nospec(nr, NR_syscalls);
        // the return value will overwrite -ENOSYS
		regs->ax = sys_call_table[nr](regs);
    //...
}
```
对应流程为：

1. [add_random_kstack_offset()](https://elixir.bootlin.com/linux/v5.13/source/include/linux/randomize_kstack.h#L35) 为内核栈基址引入随机性，参见 [KASLR](../todo)。
2. [syscall_enter_from_user_mode()](https://elixir.bootlin.com/linux/v5.13/source/kernel/entry/common.c#L100) 调用：
    1. [__enter_from_user_mode()](https://elixir.bootlin.com/linux/v5.13/source/kernel/entry/common.c#L16)：通知 [lockdep]() 可屏蔽中断已屏蔽等；
    2. [__syscall_enter_from_user_work()](https://elixir.bootlin.com/linux/v5.13/source/kernel/entry/common.c#L85)：完成 [thread_info->syscall_work](https://elixir.bootlin.com/linux/v5.13/source/arch/x86/include/asm/thread_info.h#L58) 要求的工作，如 [系统调用过滤](../seccomp-bpf) 等；
3. [array_index_nospec()](https://elixir.bootlin.com/linux/v5.13/source/include/linux/nospec.h#L51) 宏将非法 nr 修改为 0。因为即使有 if 判断，CPU 错误的分支预测仍可能导致非法 nr 被执行；
4. 根据系统调用号查表，执行系统调用。

如果系统调用号设置了 [__X32_SYSCALL_BIT](https://elixir.bootlin.com/linux/v5.13/source/arch/x86/include/uapi/asm/unistd.h#L13) 位，将查找 32 位的系统调用表。

### 执行 sysret 指令
在返回前，先从栈上的 pt_regs 恢复部分寄存器进行检查：

- pt_regs.RCX == pt_regs.RIP 且其值为合规地址，其中 pt_regs.RIP 临时恢复到 R11；
- $__USER_CS == pt_regs.CS，仅检查栈上内容，不恢复到 CS；
- pt_regs.R11 == pt_regs.RFLAGS；
- (RF | TF) & pt_regs.R11 == 0，RFLAGS 的 RF 和 TF 位必须为 0；
- RSP 无须检查；
- $__USER_DS == pt_regs.SS，仅检查栈上内容，不恢复到 SS。

如果检查未通过，将改用 iret 指令返回。如果通过，则调用 [POP_REGS](https://elixir.bootlin.com/linux/v5.13/source/arch/x86/entry/calling.h#L149) 宏恢复剩余寄存器，这里跳过了马上要使用的 RDI：
```c
    POP_REGS pop_rdi=0 skip_r11rcx=1    // RCX and R11 are already restored 
```
现在 RSP 指向 pt_regs.RDI 的位置。切换到 trampoline 栈，进行可能需要的善后工作：
```c
    // now all regs are restored except RSP and RDI
    movq	%rsp, %rdi                                  // save the old stack pointer to RDI
    movq	PER_CPU_VAR(cpu_tss_rw + TSS_sp0), %rsp     // go to the trampoline stack

    // now RDI points to the old stack
    pushq	RSP-RDI(%rdi)   // save pt_regs.RSP there
    pushq	(%rdi)          // save pt_regs.RDI there

    // now we are on the trampoline stack, all regs except RDI are live
    
    // in the future, other exit work can be done here
    // for now, only gcc may do something, see CONFIG_GCC_PLUGIN_STACKLEAK
    STACKLEAK_ERASE_NOCLOBBER   
```
通过 [SWITCH_TO_USER_CR3_STACK](https://elixir.bootlin.com/linux/v5.13/source/arch/x86/entry/calling.h#L244) 宏切换 CR3，将刚刚入栈的 pt_regs.RDI 和 pt_regs.RSP 恢复到对应寄存器，让 GS 段指回用户空间，执行 sysret 指令：
```c
    SWITCH_TO_USER_CR3_STACK scratch_reg=%rdi

    popq	%rdi
    popq	%rsp
    swapgs
    sysretq
```

## [sysret 指令](https://www.felixcloutier.com/x86/sysret)：ring0 回到 ring3
sysret 指令将恢复之前 syscall 指令修改的内容，包括 CPL、RIP/RFLAGS 和 CS/SS，具体操作如下：

1. 如果 CPL 不为 0 或 RCX 的值不是合规地址，抛 #GP(0) 异常；
2. 设置 RIP 为保存在 RCX 中的返回地址；
3. 从 R11 中 恢复 RFLAGS；
4. 设置代码段寄存器 CS：
    - Selector = (IA32_STAR[63:48] + 16) | 3，或操作是设置 RPL 为 3；
    - Base = 0；Limit = FFFFFH；Type = 11；DPL = 3；...
5. CPL = 3，返回用户态；
6. 设置堆栈段寄存器 SS：
    - Selector = (IA32_STAR[63:48] + 8) | 3，或操作是设置 RPL 为 3；
    - Base = 0；Limit = FFFFFH；Type = 3；DPL = 3；...

现在 RIP 指向之前 syscall 指令的下一条指令，控制流将继续向前。