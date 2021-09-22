# Seccomp BPF

参见 [A seccomp overview](https://lwn.net/Articles/656307/)。BPF 相关参见 [从 BPF 到 eBPF](../bpf)。

Seccomp（SECure COMPuting）源自 05 年的一个 [patch](https://lwn.net/Articles/120192/)，只允许进程执行 4 个系统调用 read()、write()、exit() 和 [sigreturn()](../todo)，其他系统调用将导致进程终止。思路是在系统调用入口处过滤，根据系统调用号拦截不在白名单中的系统调用。

这一版本的 seccomp 在实现上缺乏灵活性，白名单硬编码在内核中，无法在运行时配置。BPF 引入后，[改用 BPF 实现](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e2cfabdfd075648216f99c2c03821cf3f47c1727)，3.4 发布，沿用至今。

[Seccomp BPF](https://www.kernel.org/doc/html/latest/userspace-api/seccomp_filter.html) 方案在执行系统调用前运行安装在当前进程上的所有过滤器。过滤器描述为 cBPF，在用户空间编写，附加到进程后生效，内核将其转换为 eBPF 执行。

而早期实现虽然不灵活，但是简单，在数值计算这种不需要其他系统调用的场景有效率优势，因此也保留了下来，即 seccomp 的 strict 模式。

## 编写过滤器
Seccomp 过滤器使用 cBPF 指令 [sock_filter](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/filter.h#L24)：
```c
struct sock_filter {    /* Filter block */
    __u16   code;       /* Actual filter code */
    __u8    jt;         /* Jump true */
    __u8    jf;         /* Jump false */
    __u32   k;          /* Generic multiuse field */
};
```
指令序列及长度封装为 [sock_fprog](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/filter.h#L31)： 
```c
struct sock_fprog {
    unsigned short len;    /* Number of filter blocks */
    struct sock_filter __user *filter;
};
```
过滤器根据封装为结构体 [seccomp_data](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/seccomp.h#L60) 的系统调用信息进行响应：

- nr：系统调用号，[__X32_SYSCALL_BIT](https://elixir.bootlin.com/linux/v5.13/source/arch/x86/include/uapi/asm/unistd.h#L13) 如果设置了，也会包含在内；
- arch ：系统架构 [AUDIT_ARCH_*](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/audit.h#L383)；
- instruction_pointer：系统调用的返回地址；
- args[6]：系统调用参数，无论类型和架构，均按 u64 处理，不会解引用指针。

典型结构为：
```c
if (arch != ARCH_X86_64)
    return KILL;
return ALLOW;
```
分解为 cBPF 指令：

1. BPF_LD | BPF_W | BPF_ABS 指令：将 seccomp_data 在 k = 4 字节偏移处的 32 位数据，即 arch 字段，压入寄存器 A；
2. BPF_JMP | BPF_JEQ | BPF_K 指令：比较 A 与 k = ARCH_X86_64，jt = 0 指向下一条指令，jf = 1 指向下下一条指令；
3. BPF_RET | BPF_K 指令：返回 k = ALLOW；
4. BPF_RET | BPF_K 指令：返回 k = KILL；

使用 [BPF_STMT/BPF_JUMP](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/filter.h#L49) 宏描述该指令序列：
```c
struct sock_filter f[] = {
    BPF_STMT(BPF_LD | BPF_W | BPF_ABS, 4);
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, AUDIT_ARCH_X86_64, 0, 1);
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW);
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL);
}
```
封装为 sock_fprog：
```c
struct sock_fprog prog = {
    .len = 4
    .filter = f,
};
```

过滤器的返回值 32 位。低 16 位用于回传数据；高 16 位为 action = [SECCOMP_RET_*](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/seccomp.h#L36)，优先级从高到低为：

- 拒绝：
    - SECCOMP_RET_KILL_PROCESS：以 SIGSYS 终止当前进程组的所有进程；
    - SECCOMP_RET_KILL{_THREAD}：以 SIGSYS 终止当前进程；
    - SECCOMP_RET_TRAP：向当前进程发送 SIGSYS；
    - SECCOMP_RET_ERRNO：返回自定义错误号，放在低 16 位中；
- 询问用户空间，无应答则返回错误号 [ENOSYS](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/asm-generic/errno.h#L18)：
    - SECCOMP_RET_USER_NOTIF：unotify 机制，让用户空间决定如何响应；
    - SECCOMP_RET_TRACE：让 tracer 决定如何响应；
- 允许：
    - SECCOMP_RET_LOG：登记日志，允许该系统调用；
    - SECCOMP_RET_ALLOW，允许该系统调用。

如果有多个过滤器，依次运行，按优先级最高的 action 响应。

## 安装过滤器
### [no_new_privs](https://www.kernel.org/doc/html/latest/userspace-api/no_new_privs.html)

no_new_privs 是一个进程标志，记录在 [task_struct->atomic_flags](https://elixir.bootlin.com/linux/v5.13/source/include/linux/sched.h#L857) 的最低位（[PFA_NO_NEW_PRIVS](https://elixir.bootlin.com/linux/v5.13/source/include/linux/sched.h#L1636)）。通过 prctl() 的 [PR_SET_NO_NEW_PRIVS](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/prctl.h#L175) 选项设置：
```c
prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);
```

带有该标志的进程无法通过 execve() 提升权限，如执行带 S 位的文件时，uid/gid 保持不变。fork()/clone() 的子进程会继承该标志位。execve() 不会覆盖该标志位。

安装过滤器，即将过滤器附加到当前进程，要求进程有 [CAP_SYS_ADMIN](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/capability.h#L279) 权限，或者设置了 no_new_privs 标志位。否则低权限进程可以利用过滤器提权。

考虑一个针对 setuid() 的过滤器，它允许 setuid(0)，但对于 setuid(uid>0) 返回错误号 0。如果没有 no_new_privs 限制，持有该过滤器的进程 execve() 一个属于 root 用户的、带 S 位的文件即可获得 root 权限。

### [seccomp()](https://elixir.bootlin.com/linux/v5.13/source/kernel/seccomp.c#L1949)
通过系统调用 seccomp() 的 [SECCOMP_SET_MODE_FILTER](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/seccomp.h#L16) 选项安装过滤器：
```c
seccomp(SECCOMP_SET_MODE_FILTER, flags, args);
```
args 指向用户定义的 sock_fprog，flags = [SECCOMP_FILTER_FLAG_*](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/seccomp.h#L21)，包括：

- SECCOMP_FILTER_FLAG_TSYNC：同步过滤器到进程组其他进程；
- SECCOMP_FILTER_FLAG_LOG：登记 action 日志；
- SECCOMP_FILTER_FLAG_SPEC_ALLOW：不执行 [arch_seccomp_spec_mitigate()](https://elixir.bootlin.com/linux/v5.13/source/arch/x86/kernel/cpu/bugs.c#L1333)；
- SECCOMP_FILTER_FLAG_NEW_LISTENER：返回文件描述符，用于 unotify 机制；

另外，可以通过 [SECCOMP_SET_MODE_STRICT](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/seccomp.h#L15) 选项启用 strict 模式：
```c
seccomp(SECCOMP_SET_MODE_STRICT, 0, NULL);
```

### prctl()
prctl() 提供 [PR_SET_SECCOMP](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/prctl.h#L68) 选项，功能是 seccomp() 的子集。

- 安装过滤器，等价于 `seccomp(SECCOMP_SET_MODE_FILTER, 0, args)`：
```c
prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, args);  // flags not supported
```
- 启用 strict 模式，等价于 `seccomp(SECCOMP_SET_MODE_STRICT, 0, NULL)`：
```c
prctl(PR_SET_SECCOMP, SECCOMP_MODE_STRICT);
```

## 内核执行
### 安装过滤器
seccomp() 和 prctl() 都是调用 [do_seccomp()](https://elixir.bootlin.com/linux/v5.13/source/kernel/seccomp.c#L1924) -> [seccomp_set_mode_filter()](https://elixir.bootlin.com/linux/v5.13/source/kernel/seccomp.c#L1787)：

1. [seccomp_prepare_user_filter()](https://elixir.bootlin.com/linux/v5.13/source/kernel/seccomp.c#L685) 将用户传入的 sock_fprog 转换为 eBPF 格式。其先 copy_from_user() 取得 sock_fprog，然后调用 [seccomp_prepare_filter()](https://elixir.bootlin.com/linux/v5.13/source/kernel/seccomp.c#L632)：
    1. 检查指令长度，必须大于 0，且不超过 4096（[BPF_MAXINSNS](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf_common.h#L54)）；
    2. 检查权限，current 必须有 CAP_SYS_ADMIN 权限，或设置了 no_new_privs 标志；
    3. 调用 [bpf_prog_create_from_user() 将 sock_fprog 转换为 bpf_prog](../bpf#_9)，即过滤器的 eBPF 表示，记录在 [seccomp_filter](https://elixir.bootlin.com/linux/v5.13/source/kernel/seccomp.c#L215) 中。
2. [seccomp_attach_filter()](https://elixir.bootlin.com/linux/v5.13/source/kernel/seccomp.c#L860) 将 eBPF 过滤器头插到 current 持有的过滤器单链表，如果指定了 TSYNC 标志，调用 [seccomp_sync_threads()](https://elixir.bootlin.com/linux/v5.13/source/kernel/seccomp.c#L575) 同步过滤器；
3. [seccomp_assign_mode()](https://elixir.bootlin.com/linux/v5.13/source/kernel/seccomp.c#L437) 设置 current 为 filter 模式，通过 [set_task_syscall_work()](https://elixir.bootlin.com/linux/v5.13/source/include/linux/thread_info.h#L141) 宏设置 [SYSCALL_WORK_BIT_SECCOMP](https://elixir.bootlin.com/linux/v5.13/source/include/linux/thread_info.h#L41) 位。

第 2 步的过滤器链表通过 seccomp_filter 的 prev 字段维护，头节点 filter 记录在结构体 [seccomp](https://elixir.bootlin.com/linux/v5.13/source/include/linux/seccomp.h#L35) 中：
```c
struct seccomp {
    int mode;                         // filter or strict
    atomic_t filter_count;
    struct seccomp_filter *filter;    // accessed without locking
};
```

seccomp 是 task_struct 的成员，但只有 current 才能访问 filter。fork()/clone() 的子进程会继承父进程的 filter，因此不同过滤器可以有相同的 prev。execve() 不会覆盖 filter。

### 过滤系统调用
执行系统调用前，[do_syscall_64() 会先进行 thread_info->syscall_work 要求的工作](../syscall-execution#_3)。安装过滤器时设置了 SYSCALL_WORK_BIT_SECCOMP 位，因此会进入 [syscall_trace_enter()](https://elixir.bootlin.com/linux/v5.13/source/kernel/entry/common.c#L44) 并运行过滤器：
```c
/* Do seccomp after ptrace, to catch any tracer changes. */
if (work & SYSCALL_WORK_SECCOMP) {
    ret = __secure_computing(NULL);
    //...
}
```
[SYSCALL_WORK_SECCOMP](https://elixir.bootlin.com/linux/v5.13/source/include/linux/thread_info.h#L50) 是对应掩码。[__secure_computing()](https://elixir.bootlin.com/linux/v5.13/source/kernel/seccomp.c#L1301) 将调用 [__seccomp_filter()](https://elixir.bootlin.com/linux/v5.13/source/kernel/seccomp.c#L1164) 运行 current 上的过滤器：

1. [populate_seccomp_data()](https://elixir.bootlin.com/linux/v5.13/source/kernel/seccomp.c#L234) 收集系统调用信息，即 nr，arch，instruction_pointer 和 args[6]，封装为 seccomp_data；
2. [seccomp_run_filters()](https://elixir.bootlin.com/linux/v5.13/source/kernel/seccomp.c#L394) 从 current.seccomp->filter 起依次运行链表上所有过滤器，优先级最高的 action 作为最终响应。


