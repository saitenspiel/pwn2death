# gdb 调试内核
## 安装 QEMU
### 0. 安装 [Ninja](https://github.com/ninja-build/ninja.git) 及其依赖 [re2c](http://re2c.org/index.html)
```bash
wget https://github.com/skvadrik/re2c/releases/download/2.1.1/re2c-2.1.1.tar.xz
tar xf re2c-2.1.1.tar.xz
cd re2c-2.1.1

./configure
make
sudo make install
```
```bash
git clone git://github.com/ninja-build/ninja.git
cd ninja

./configure.py --bootstrap
sudo cp ninja /usr/bin
```

### 1. 下载 [QEMU 源码](https://download.qemu.org/?C=M;O=D)
```bash
wget https://download.qemu.org/qemu-6.0.0.tar.xz
tar xf qemu-6.0.0.tar.xz
```
### 2. 配置编译目标
配置前先安装编译依赖，--target-list 指定编译目标。-softmmu 后缀对应 QEMU 的系统模式，-linux-user 对应用户模式，将分别生成 qemu-system- 前缀和 qemu- 前缀的可执行程序。调试内核用前者：
```bash
cd qemu-6.0.0
sudo apt build-dep qemu
./configure --target-list=x86_64-softmmu,i386-softmmu
```


### 3. 编译、安装
```bash
make -j8
sudo make install
```
## 准备内核镜像
### 1. 下载 [内核源码](https://cdn.kernel.org/pub/linux/kernel)
```bash
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.12.3.tar.xz
tar xf linux-5.12.3.tar.xz
```

### 2. 配置编译选项
```bash
cd linux-5.12.3
make defconfig
```
有些调试选项不包含在默认配置中，可手动启用。在 make menuconfig 中按 / 可查找选项位置。

- CONFIG_DEBUG_INFO，带上调试信息，类似 gcc -g
    - Kernel hacking > Compile-time checks and compiler options > ...
- gdb，kgdb 和 kdb 设置参考官方文档，相关选项 GDB_SCRIPT，FRAME_POINTER，KGDB 等
    - [使用 gdb 调试内核](https://www.kernel.org/doc/html/latest/dev-tools/gdb-kernel-debugging.html)，[启用 kgdb 和 kdb](https://www.kernel.org/doc/html/latest/dev-tools/kgdb.html)
- 视情况启用 KASAN、SLUB 等选项。
    - Kernel hacking > Memory Debugging > ...

### 3. 编译
```bash
make -j8
```
得到原始镜像 ./vmlinux 和压缩后的镜像 ./arch/x86/boot/bzImage。

## 准备 rootfs 镜像
### 1. 下载 [BusyBox](https://busybox.net)
```bash
wget https://busybox.net/downloads/busybox-1.33.1.tar.bz2
tar xf busybox-1.33.1.tar.bz2
```

### 2. 配置编译选项
```bash
cd busybox-1.33.1
make menuconfig
```
启用 Settings > Build static binary (no shared libs)。

### 3. 编译
```bash
make -j8
make install
```
得到 _install 目录。

### 4. 编写启动脚本
默认启动脚本为 etc/init.d/rcS，在其中挂载 /proc 和 /sys，创建设备节点：
```bash
cd _install
mkdir -p etc/init.d
vim etc/init.d/rcS
```
```txt
#!/bin/sh
mkdir proc && mount -t proc proc /proc
mkdir sys && mount -t sysfs sysfs /sys
/sbin/mdev -s
```
给启动脚本加上可执行权限：
```bash
chmod +x etc/init.d/rcS
```

### 5. 生成 rootfs 镜像
打包 _install 目录得到 rootfs 镜像，必须使用 newc 格式：
```bash
find . | cpio -o -H newc > ../../rootfs.img
```

## gdb 调试内核
### 1. 启动 QEMU
```txt
qemu-system-x86_64 -kernel linux-5.12.3/arch/x86/boot/bzImage \
                   -initrd rootfs.img \
                   -append "rdinit=/sbin/init console=ttyS0" \
                   -nographic -s
```
QEMU [启动参数](https://qemu-project.gitlab.io/qemu/system/invocation.html) 很多，要看情况设置。上述启动命令涉及到的参数：

- -kernel 指定内核镜像 bzImage
- -initrd 指定根文件系统镜像。这里用准备好的 rootfs.img
- -append 指定内核参数
    - rdinit 指定 init 脚本路径，默认 /init。用 BusyBox 的话这里要重载为 /sbin/init
    - console 重定向内核输出。这里重定向到串口设备 ttyS0，配合 -nographic 使用
- -nographic 重定向串口的输入输出到当前 terminal，不启动图形界面，等效于 -serial stdio
- -s 在 1234 端口上启动 gdb server，等效于 -gdb tcp::1234

退出 QEMU：terminal下，先按 Ctrl + a，再按 x ；图形界面下 Ctrl + Alt + q。

### 2. gdb 调试
```bash
gdb linux-5.12.3/vmlinux
```
注意这里只是从 vmlinux 加载符号信息，相当于：
```txt
(gdb) file linux-5.12.3/vmlinux
```
但 target 是运行在 QEMU 中的内核，需要远程连接过去：
```txt
(gdb) target remote :1234
```
