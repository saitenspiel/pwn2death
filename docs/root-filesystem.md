# 根文件系统
## Linux 引导过程
一个典型的 Linux 引导过程如下：

1. BIOS 上电自检，加载 MBR，控制权给其中的一阶段 bootloader。
2. 一阶段 bootloader 加载二阶段 bootloader 并移交控制权。
3. 二阶段 bootloader 加载内核镜像和 initramfs 镜像，将后者的内存起始地址传递给内核，并移交控制权。
4. 内核自解压，然后解压 initramfs 到预挂载的 rootfs 区域，控制权给其中的 init 脚本，默认为 /init。
5. init 脚本挂载实际的根文件系统，并执行其中的 /sbin/init。

引导（boot）到这里就算结束。接下来是 /sbin/init 负责的启动（startup）阶段，/sbin/init 作为 1 号进程，负责将系统带入到一个用户可操作状态。现在主流的 /sbin/init 程序是 systemd，从趋势上讲它代替了早期 SystemV 的方案，成为目前各大发行版默认的 init 程序。

## 文件系统的两种含义
在最广泛的意义上，文件系统（filesystem）是数据的存取规范。在此区分它的两种含义。

第一个含义是物理存储上的读写规范，对应于「文件系统类型」。其规定了写存储时要怎么组织数据结构，读存储时要怎么解析回来。一般来说，这里的存储是指磁盘这类稳定存储。但是，既然 RAM 也是可读可写的，而且速度更快，能不能在 RAM 中也建立文件系统呢？答案是肯定的，这就是内存文件系统 ramfs。如果不加限制，对 ramfs 的写操作可以持续进行，直到写满整个 RAM，因此这类 ramfs 只能允许 root 用户写。为了让普通用户也能用，在 ramfs 的基础上加上容量限制并启用 swap 空间，这就是 tmpfs。

文件系统的第二个含义是目录树的组织结构。它描述的是如何组织文件路径，不关心具体的物理存储。秉承着一切皆文件的思想，内核封装了对不同设备的访问操作，向上只暴露一个统一的文件系统接口，即 VFS。读写设备，自然也包括存储设备，就是读写文件，必须先建立文件路径（即挂载点）到物理设备的映射（即挂载），然后通过文件路径进行。文件路径共同构成了目录树，它的根，也就是常说的根目录 /，必须最先被挂载。只有根目录被挂载了，其他设备才能顺着这个根目录往下挂载。注意，根目录不完全等于 /，它只是一个逻辑上的根节点，可以在系统运行时切换。

总的来说，两种含义都是存取规范，但是：

- 「物理存储的读写规范」偏向物理层面。其关心的是描述符、位图等元信息的管理，数据块的分配，日志的使用等。常说的 ext3、ext4 文件系统，也包括 ramfs/tmpfs 这样的内存文件系统，就是这种含义。这和 mount -t 选项的可用参数，还有 df -T 的 Type 列是对应的。
- 「目录树的组织结构」偏向逻辑层面。其关心的是数据应该放在哪个目录，通过特定的文件路径能不能访问到符合预期的数据，比如设备文件要在 /dev 目录下，配置文件要在 /etc 目录下等。这在系统引导期间尤为关键，许多文件路径是硬编码在内核代码中的，例如 /init 这个默认的 init 脚本路径。

两种含义之间有隐性的蕴含关系。第一，目录树要有物理设备来承载。换句话说，有了底层存储设备的读写规范，才能有逻辑层的目录树。后者是依托前者实现的。回想一下，目录实际上是存储子节点信息的文件。第二，这里的目录树要么指整个树，要么指某个设备的目录树（设备是逻辑上的，标准是具有独立的文件系统类型，如硬盘不同分区算不同设备）。换句话说，这里的目录树要么对应整个系统，要么对应一个有独立文件系统类型的设备。前者虽包含不同类型的文件系统，但也是通过一系列的挂载操作得到的，可以归约到后者。这也就是说，第二种含义中的目录树往往有特定的文件系统类型与之对应，因此蕴含了第一种含义。

## 文件系统挂载
所谓挂载设备，更准确的说法是挂载该设备的文件系统。这里的文件系统兼具以上两种含义。挂载是为了引入新设备，它有自己的文件系统类型，据此可以导出它的目录树（至少有一个根目录）。这个目录树以子树的形式附加到挂载点，然后其中的数据就能通过文件路径访问了。

挂载操作是 VFS 提供的接口之一。从实现上讲，挂载一个文件系统的前提是能解析该文件系统，且之后的读写操作虽然通过 VFS 进行，但最终也要转换成对设备的直接访问。这些过程都是依赖设备驱动的。因此，挂载一个设备上的文件系统，必须有该设备的驱动。

## 根文件系统和 rootfs
根文件系统是最先挂载的文件系统，挂载点即根目录。这里的文件系统取第二个含义，也就是说，根文件系统关心的是文件有没有按要求组织在目录树中。标准可以参考 [Chapter 3. The Root Filesystem](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/ch03.html)，其中给出了 14 个必要的一级目录，包括 /bin、/boot、/dev 等。这个标准并非强制，可以按需剪裁，毕竟归根结底，标准是为功能服务的。比如 Linux 就对其进行了调整，引入了 /proc、/sys 等，参见 [6.1. Linux](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/ch06.html#linux)。

根文件系统的目的很纯粹。它不关心文件系统类型，ext4 也行，ramfs 也行，只要能把系统引导到其他文件系统可以挂载的状态就行。这是它作为第一个挂载的文件系统必须完成的任务，甚至可以说是唯一要完成的任务。其他功能都可以放在后续挂载的文件系统中。不过，从健壮性的角度考虑，根文件系统通常还要求包含修复工具，以便诊断和恢复系统故障。

> [The Linux Information Project](http://www.linfo.org/root_filesystem.html): The contents will include the files that are necessary for booting the system and for bringing it up to such a state that the other filesystems can be mounted as well as tools for fixing a broken system and for recovering lost files from backups.

也就是说，根文件系统可以任意大，但不能任意小。为了保证系统成功引导，它至少要包含 init 脚本及其依赖的配置文件。在此基础上再去支持其他功能。另一方面，尽管可以任意大，但根文件系统是越小越好的。一是节省存储，根文件系统一般包含特定设备驱动，不通用，都是各用各的本地镜像。这在嵌入式场景下尤其重要；二是降低故障率，体积小便于维护。

挂载根文件系统最大的困难是要适配不同硬件。根文件系统可能存储在各类设备中，它们有各自的驱动。对于发行版来说，把驱动直接编译进内核是不可行的。因为发行版要适配大量设备，把所有驱动模块全编译进去会让内核臃肿得难以想象，而且每次更新驱动都要重新发布内核。另一方面，以 [LKM](../todo) 的方式编译驱动也不可行，因为 bootloader 只负责加载内核，驱动模块的选择和加载得由用户程序完成，这就产生了死锁：

- 必须先挂载根文件系统才能加载驱动，因为驱动和加载程序都在根文件系统中。
- 必须先加载驱动才能挂载根文件系统，因为挂载文件系统需要设备驱动。

解决方案是引入 rootfs 这样一个临时的、建立于 RAM 中的根文件系统，它持有所需驱动，负责挂载真正的根文件系统。所谓真正的根文件系统，就是系统正常运行期间（rootfs 只在引导阶段发挥作用）挂载在根目录上的那个文件系统，一般是第一块硬盘的第一个逻辑分区（参考 df 命令）。以下用「根文件系统」指代这个「真正的根文件系统」。

> [Wiki](https://en.wikipedia.org/wiki/Initial_ramdisk): This root file-system can contain user-space helpers which do the hardware detection, module loading and device discovery necessary to get the real root file-system mounted.

内核先挂载 rootfs，因为没有适配问题，所需驱动可以硬编码在内核中。然后，rootfs 再来挂载根文件系统，所需驱动必须包含在 rootfs 中。如果够用的话，也可以直接把 rootfs 当成根文件系统，即不进行第二次挂载。

rootfs 将根文件系统的加载放在用户程序实现，不仅更好维护，还精简了内核。有了 rootfs，内核就不用操心驱动，根文件系统也可以灵活存储，比如放在 NFS、RAID、逻辑卷甚至加密设备上。实际操作中，可以让 rootfs 只包含常见设备的驱动，作为通用镜像发布。用户要适配新设备可以自己制作镜像，在其中包含相应驱动即可。

## initrd 和 initramfs
rootfs 的前身是 initrd，2.6.13 以后正式换成了 initramfs 实现，兼容 initrd。不过 initrd 这个历史名称被保留了下来，很多地方都有体现，包括 /boot 下的 initrd.img，GRUB 配置文件里的 initrd 参数，还有内核参数 rdinit 等等，即使它们实际上用的是 initramfs 方案。

initrd 的全称是 initial ramdisk。所谓 ramdisk，就是用一段固定大小的内存区域模拟磁盘：

1. bootloader 加载 initrd 镜像到内存。
2. 内核将内存中的 initrd 解压到 /dev/ram0，将其当作 rootfs 挂载，并调用其中的 init 脚本 /linuxrc。
3. /linuxrc 挂载根文件系统，并调用 [pivot_root()](../todo) 将其设为根目录。
4. 内核解挂载 /dev/ram0，释放其占用的内存，然后调根文件系统的 /sbin/init。

对 ramdisk 的 I/O 相当于磁盘 I/O，这意味着：第一，会触发 [页缓存](../todo)。而 ramdisk 的数据本来就在内存，没必要去浪费页缓存。第二，需要驱动。ramdisk 用的是 ext2 这样的磁盘文件系统，必须将相应驱动编译进内核中。

initramfs 方案更为简洁。它不在内存中模拟磁盘，而是直接建立文件系统：

1. bootloader 加载 initramfs 镜像到内存，其可以是独立的外部镜像，也可以编译进内核的 .init.ramfs 段中。
2. 内核解压内存中的 initramfs 到预先挂载好的 rootfs 区域，并调用其中的 init 脚本 /init。
3. /init 不会返回，它挂载根文件系统，借助 switch_root 工具将其设为根目录，并执行其中的 /sbin/init。

initramfs 下的 rootfs 不能被解挂载（内核为所有挂载点维护了一个双向链表，rootfs 是该链表的头），因此切换根目录不能用 pivot_root()。取而代之的是 [switch_root](https://git.busybox.net/busybox/tree/util-linux/switch_root.c)。/init 用该工具做三件事：

1. 删除 rootfs 的内容以释放内存；
2. 将根文件系统的挂载点重设为根目录；
3. 执行根文件系统中的 /sbin/init。

## BusyBox
BusyBox 将常用工具精简后包含进单个可执行文件中，广泛用于制作 rootfs 镜像。使用时注意：

- 启用静态链接选项。如果使用动态链接，得额外包含依赖的库文件。静态链接生成的可执行程序虽然大一点，但对内核调试来说没什么影响，只会更加方便。
- 制作 initramfs 镜像，即使用 cpio 打包 _install 目录，注意指定 newc 格式。
- 内核调 init 脚本默认是调 /init，但 BusyBox 将该脚本放置在 /sbin/init（可能是因为已经在根目录下放了 /linuxrc），需要通过内核命令行参数 rdinit 重载其路径。
    - 对于 QEMU 来说，即在启动时指定 `-append "rdinit=/sbin/init"`。也可以在打包镜像前手动生成 /init，即在 _install 目录下执行 `ln -s /bin/busybox init`。
- 所有可执行文件（/bin，/sbin，/usr/bin，/usr/sbin）都是软链接到 BusyBox，/sbin/init 也是。BusyBox 的默认启动脚本是 /etc/init.d/rcS，需要在其中挂载 /proc 和 /sys，否则内核无法正常工作。
- BusyBox 提供 udev 的变种 mdev 工具，用于创建设备节点。


