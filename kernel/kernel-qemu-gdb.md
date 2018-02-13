# 通过QEMU+GDB调试Linux内核

## 环境

宿主机（调试机）环境:

* Ubuntu 16.04.3 LTS
* Intel(R) Core(TM) i5-4590 CPU @ 3.30GHz
* `4.4.0-112-generic x86_64`

qemu虚拟机（被调试）：

* kernel: 主线版本 v4.9

## 内核编译

下载并配置并编译内核,

```shell
$ cd linux
$ make defconfig
$ make menuconfig
```

建议的配置如下，注意`[*]`或`<*>`代表选择，`[ ]`代表关闭．

```shell
Kernel hacking  --->
  [*] KGDB: kernel debugger  --->
    <*> KGDB: use kgdb over the serial console (NEW)
    [*] KGDB: Allow debugging with traps in notifiers
  Compile-time checks and compiler options  --->
    [*] Compile the kernel with debug info
    [*]   Provide GDB scripts for kernel debugging

Kernel hacking  --->
  Memory Debugging  --->
    [ ] Testcase for the marking rodata read-only

Processor type and features  --->
  [ ] Randomize the address of the kernel image (KASLR)
```

编译内核，用虚拟机跑调试内核，不需要install．可以使用`-j`选项设置make的cpu数提高编译速度．

> 新版本Kernel需要关闭KASLR，否则无法设置断点调试.

```shell
make # or make -j#cpu
make modules
```

看看是否编译成功，

```
$ ls vmlinux ./arch/x86/boot/bzImage
./arch/x86/boot/bzImage  vmlinux
```

## 制作根文件系统

下面，我们使用[busybox](https://www.busybox.net/)制作一个简单的根文件系统．

### 编译busybox

首先需要编译busybox，下载个稳定版本的busybox并编译，

```shell
$ wget http://busybox.net/downloads/busybox-1.27.2.tar.bz2
$ tar vjxf busybox-1.27.2.tar.bz2
$ cd busybox-1.27.2/
$ make menuconfig
```

相关的配置如下，同样`[*]`的代表打开，`[ ]`代表关闭.可以适当增加删减一些配置.

```shell
Busybox Settings  --->
  [*] Don't use /usr (NEW)
  --- Build Options
  [*] Build BusyBox as a static binary (no shared libs)
  --- Installation Options ("make install" behavior)
  (./_install) BusyBox installation prefix (NEW)

Miscellaneous Utilities  --->
  [ ] flash_eraseall
  [ ] flash_lock
  [ ] flash_unlock
  [ ] flashcp
```

编译busybox，会安装到`./_install`目录下

```shell
$ make # or make -j#cpu
$ make install

$ ls _install/
bin  linuxrc  sbin
```

### 制作根文件系统

根文件系统镜像大小256MiB，格式化为ext3文件系统．

```shell
# in working-dir
$ dd if=/dev/zero of=rootfs.img bs=1M count=256
$ mkfs.ext3 rootfs.img
```

将文件系统mount到本地路径，复制busybox相关的文件，并生成必要的文件和目录

```
# in working-dir
$ mkdir /tmp/rootfs-busybox
$ sudo mount -o loop $PWD/rootfs.img /tmp/rootfs-busybox

$ sudo cp -a busybox-1.27.2/_install/* /tmp/rootfs-busybox/
$ pushd /tmp/rootfs-busybox/
$ sudo mkdir dev sys proc etc lib mnt
$ popd
```

还需要制作系统初始化文件,

```shell
# in working-dir
$ sudo cp -a busybox-1.27.2/examples/bootfloppy/etc/* /tmp/rootfs-busybox/etc/
```

Busybox所使用的`rcS`，内容可以写成

```shell
$ cat /tmp/rootfs-busybox/etc/init.d/rcS
#! /bin/sh

/bin/mount -a
/bin/mount -t sysfs sysfs /sys
/bin/mount -t tmpfs tmpfs /dev
/sbin/mdev -s
```

接下来就不需要挂载的虚拟磁盘了

```
$ sudo umount /tmp/rootfs-busybox
```

## 安装qemu

安装并运行编译的调试内核

```shell
$ sudo apt-get install qemu # 或 Fedora/CentOS: yum install qemu
$ sudo qemu-system-x86_64 -kernel linux/arch/x86/boot/bzImage -append 'root=/dev/sda' -boot c -hda rootfs.img -k en-us
```

> tips: 使用`ctrl+alt+2`切换qemu控制台,使用`ctrl+alt+1`切换回调试kernel


## 通过qemu和gdb调试kernel

使用qemu运行Kernel，然后切换到qenu控制台，输入

```shell
(qemu) gdbserver tcp::1234
Waiting for gdb connection on device 'tcp::1234'
```

> 或这直接使用qemu的`-serial`选项启动内核: `-serial tcp::4321,server`.

打开另一个终端，使用调试工具(gdb/ddd/cgdb)调试vmlinux文件，并连接gdb server．

```shell
$ cd linux
$ gdb vmlinux
... ...
Reading symbols from vmlinux...done.
(gdb) target remote localhost:1234
Remote debugging using localhost:1234
0xffffffffa4f88b50 in ?? ()
(gdb) b ip_rcv
Breakpoint 1 at 0xffffffff817d9b70: file net/ipv4/ip_input.c, line 413.
(gdb) c
Continuing.
```

# 参考

https://www.jianshu.com/p/02557f0d29dc
http://blog.csdn.net/ganggexiongqi/article/details/5877756
https://www.binss.me/blog/how-to-debug-linux-kernel/