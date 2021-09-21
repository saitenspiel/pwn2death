# 使用 Syzkaller
## 安装 QEMU
参考 [gdb 调试内核：安装 QEMU](../kernel-debugging-qemu-busybox-gdb#qemu)。

## 准备内核镜像
拉源码，配置，编译。流程参考 [gdb 调试内核：准备内核镜像](../kernel-debugging-qemu-busybox-gdb#_1)。Syzkaller 要求至少启用以下配置选项：

- CONFIG_KCOV：采集覆盖率信息。
- CONFIG_DEBUG_INFO：带上调试信息。
- CONFIG_KASAN、CONFIG_KASAN_INLINE：开启 [KASAN](../todo)，检测 OOB、UAF 等漏洞。INLINE 表示使用内联插桩，和外联插桩的区别是：外联插函数调用，占用空间小，但是慢；内联则展开函数直接插代码，省去了过程调用的开销，效率更高，代价是生成的镜像会大很多。
- CONFIG_CONFIGFS_FS、CONFIG_SECURITYFS：Syzkaller 使用 Debian 镜像作为 [根文件系统](../root-filesystem)，为此需要启用 [configfs](https://www.kernel.org/doc/html/latest/filesystems/configfs.html) 和 [securityfs](https://www.linux.org/threads/pipefs-sockfs-debugfs-and-securityfs.9638/)。

其他选项可参考 [kernel_configs.md](https://github.com/google/syzkaller/blob/master/docs/linux/kernel_configs.md) 配置，也可以直接用 Syzkaller 提供的 [.config](https://github.com/google/syzkaller/blob/master/dashboard/config/linux/upstream-apparmor-kasan.config) 文件。

### Quick Start
```bash
cd kernel/linux-5.12.13
wget https://raw.githubusercontent.com/google/syzkaller/master/dashboard/config/linux/upstream-apparmor-kasan.config -O .config
make oldconfig
[Enter]...
make -j8
```
该配置文件开启了 CONFIG_DEBUG_INFO_BTF 选项，即带上 BPF 的调试信息。该选项要求的 [pahole](https://git.kernel.org/pub/scm/devel/pahole/pahole.git) 版本较高，建议关掉。否则内核编译时会报 pahole 版本太旧，并尝试自己关掉，有可能导致编译失败。目前官方是强制关掉该选项，参见 [jobs.go](https://github.com/google/syzkaller/blob/master/syz-ci/jobs.go) 和 [issue #2096](https://github.com/google/syzkaller/issues/2096)。

这个选项是用来 fuzz BPF 模块的，要开的话，建议安装最低可用版本：
```bash
sudo apt install libdw-dev
git clone https://git.kernel.org/pub/scm/devel/pahole/pahole.git
cd pahole
git checkout v1.16
mkdir build && cd build
cmake -D__LIB=lib ..
sudo make install
```
最新版本的 pahole（v1.21）在链接 .btf.vmlinux.bin.o 时会无限吃内存然后被 OOM killer 干掉。


## 准备 Debian 镜像
### 1. 安装 debootstrap
```bash
sudo apt install debootstrap
```

### 2. 创建镜像
```bash
mkdir stretch
cd stretch
wget https://raw.githubusercontent.com/google/syzkaller/master/tools/create-image.sh -O create-image.sh
chmod +x create-image.sh
sudo ./create-image.sh
```
得到根文件系统镜像 stretch.img，挂载需要的驱动已经包含在其中，不用另外准备 rootfs 镜像。相应的，启动 QEMU 时也无需指定 initrd 参数。

### 3. 测试
启动 QEMU，系统起来后用 root 用户登录，无密码。QEMU 启动脚本如下：
```txt
qemu-system-x86_64 \
	-m 2G \
	-smp 2 \
	-kernel kernel/linux-5.12.13/arch/x86/boot/bzImage \
	-append "console=ttyS0 root=/dev/sda earlyprintk=serial net.ifnames=0" \
	-drive file=stretch/stretch.img,format=raw \
	-net user,host=10.0.2.10,hostfwd=tcp:127.0.0.1:10021-:22 \
	-net nic,model=e1000 \
	-enable-kvm \
	-nographic \
	-pidfile vm.pid \
	2>&1 | tee vm.log
```
然后新开一个终端，ssh 连接到 QEMU：
```bash
sudo ssh -i stretch/stretch.id_rsa -p 10021 -o "StrictHostKeyChecking no" root@localhost
```

## 安装 Syzkaller
### 0. 安装 [Golang](https://golang.google.cn/)
Go 是 Syzkaller 的主力语言，Ubuntu 下可以通过 snap 直接安装，或者：
```bash
wget https://golang.google.cn/dl/go1.16.5.linux-amd64.tar.gz
tar xf go1.16.5.linux-amd64.tar.gz
```
配置 \$PATH，该方式只对当前终端有效：
```bash
export PATH=$PATH:`pwd`/go/bin
```

### 1. 下载 [Syzkaller 源码](https://github.com/google/syzkaller.git)
在配置了 \$PATH 的终端执行：
```bash
git clone https://github.com/google/syzkaller.git
cd syzkaller
```

### 2. 编译
```bash
make -j8
```

## 开始 fuzzing
### 1. 配置
```bash
cd syzkaller
mkdir workdir
vim my.cfg
```
```txt
{
    "target": "linux/amd64",
    "http": "127.0.0.1:56741",
    "workdir": "./workdir",
    "kernel_obj": "../kernel/linux-5.12.13",
    "image": "../stretch/stretch.img",
    "sshkey": "../stretch/stretch.id_rsa",
    "syzkaller": ".",
    "procs": 4,
    "type": "qemu",
    "vm": {
        "count": 4,
        "kernel": "../kernel/linux-5.12.13/arch/x86/boot/bzImage",
        "cpu": 2,
        "mem": 2048
    }
}
```

### 2. 运行
```bash
sudo ./bin/syz-manager -config=my.cfg
```