---
title: QEMU运行ARM Linux
description: 本文介绍如何在QEMU上运行ARM架构的Linux。QEMU作为一款非常开放且灵活的模拟器，支持市面上众多的处理器架构和SoC，因此使用QEMU进行实验和测试，是非常方便的。
categories:
 - software
tags:
---

本文介绍如何在QEMU上运行ARM架构的Linux。QEMU作为一款非常开放且灵活的模拟器，支持市面上众多的处理器架构和SoC，因此使用QEMU进行实验和测试，是非常方便的。

![image](/images/qemu_overall.png)

通常，决定使用QEMU模拟嵌入式环境的时候，首先要做的是确定目标处理器的构架和机器类型（MCU、SoC的board或者PC）。这两项确定后，模型硬件的处理器、存储设备、外设以及外设的存储映射就确定下来了。

接下来需要为选定的硬件编译软件。对于运行一个嵌入式系统来说，至少需要Linux内核、DTB（设备树文件）以及rootfs。内核需要管理硬件并协调上层的应用程序。DTB用来分离内核驱动中与特定硬件相关的细节的。由于有了DTB，才能使Linux内核中的驱动程序更加通用，而不必每次硬件有变更时都要重新编译驱动。rootfs是Liunx启动后第一个加载的文件系统，上面放置内核初始化的脚本以及内核运行过程中需要持续访问的文件。嵌入式Linux的rootfs的结构可以参考FHS v3 (Filesystem Hierarchy Standard)。

上述两步就可以在QEMU上启动arm的Linux虚拟机了，只要把kernel、dtb和rootfs作为参数传给qemu-system-arm hypervisor即可。这种方式QEMU直接把kernel加载到特定的内存位置并执行，而不必将kernel预先保存在存储里，所以不需要专门的bootloader。

这种方式虽然方便，但实际环境却很难做到的。大多数时候，我们需要一个bootloader去加载kernel。为了应对不同的嵌入式机器类型，u-boot应运而生。u-boot可以应对不同硬件的启动过程。最后，我们会看一下如何使用u-boot，再通过TFTP或者NFS远程下载内核的方式来运行Linux虚拟机。这个过程还需要对构建kernel的过程做一些简单的调整。

本文后面会详细介绍上面的每个步骤，分为四部分，1. 定义处理器和机器类型；2. 编译内核与dtb；3. 制作rootfs；4. 使用u-boot启动。

# 定义处理器架构和机器类型

QEMU支持很多种处理器架构，用 
```bash
ls /usr/bin/qemu-system-* 
```
命令可以看到你的机器上支持的类型，下图来自于我的laptop的结果：

![image](/images/output_of_ls_qemu_system-.png)

以上每条命令都对应一个模拟某种处理器的QEMU hypervior，从中也就可以看出QEMU支持的处理去类型。如图，我的laptop上支持arm, arm64(aarch64), mips, sparc, tricore, x86_64等30种不同级别的处理器，从低功耗的嵌入式微控制器到大型主机的CPU。

本文将选择arm作为实验的环境。

确实了处理器架构之后，就要确定实验用的机器类型。一种机器类型就是一系列设备的集合，这个集合的最小集是处理器和主存，不过绝大多数机器集合都还会包括各种外设，以及外设的存储映射。

对于嵌入式的处理器，会有不用的厂商开发不同的SoC，有些SoC就会在QEMU种找到对应的硬件类型，可以通过 
```bash
qemu-system-[ARCH] -M help 
```
命令查看特定处理器下QEMU支持的硬件。由于ARM是使用最广泛的嵌入式处理器，因此QEMU支持的硬件类型也最多。在我的laptop下，大约有100种。

```bash
~/qemu:~$ qemu-system-arm -M help
Supported machines are:
akita                Sharp SL-C1000 (Akita) PDA (PXA270)
ast2500-evb          Aspeed AST2500 EVB (ARM1176)
ast2600-evb          Aspeed AST2600 EVB (Cortex-A7)
bletchley-bmc        Facebook Bletchley BMC (Cortex-A7)
borzoi               Sharp SL-C3100 (Borzoi) PDA (PXA270)
canon-a1100          Canon PowerShot A1100 IS (ARM946)
cheetah              Palm Tungsten|E aka. Cheetah PDA (OMAP310)
collie               Sharp SL-5500 (Collie) PDA (SA-1110)
connex               Gumstix Connex (PXA255)
…
vexpress-a15         ARM Versatile Express for Cortex-A15
vexpress-a9          ARM Versatile Express for Cortex-A9
virt-2.10            QEMU 2.10 ARM Virtual Machine
...
```

而像tricore这类比较封闭的处理器，支持的机器类型也非常少，只有1种，还是testboard：

```bash
~/qemu:~$ qemu-system-tricore -M help
Supported machines are:
KIT_AURIX_TC277_TRB  Infineon AURIX TriBoard TC277 (D-Step)
none                 empty machine
tricore_testboard    a minimal TriCore board
```

当然，机器类型本质上就是很多设备的集合，所以可以按照SoC厂商的spec去自己定义各个元器件的虚拟peer，不过这种做法对于嵌入式来说完全没有必要。但是对于PC来说，却可以通过给QEMU传递大量的device, netdev, ram, sd等参数来设置机器的细节（基于QEMU的KVM就是这么做的，可以在运行KVM的机器上通过 
```bash 
ps axu | grep qemu-system-x86_64 
```
查看具体的命令），因为PC的标准化程度更高，不同厂商之间的差距完全取决于CPU、Memory、Disk的选择上。

本文使用arm的vexpress-a9 machine作为实验环境。vexpress-a9是arm官方发布的一款测试版，用于帮助开发者快速了解并测试产品原型。

# 编译内核与dtb

接下来编译Linux kernel和dtb。下文运行环境基于Ubuntu 20.04，对于其他的distribution，过程大同小异。

kernel的源代码可以从[kernel.org](http://kernel.org/)下载。网站上会列出stable、longterm和rc的源码包。为了在遇到问题时能找到更多的帮助，我选择了v4.9.315。下载源代码并且解压，

```bash
$ wget -c https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.9.315.tar.xz
$ tar xvd linux-4.9.315.tar.xz
$ cd linux-4.9.315
```

下面是配置源码以匹配vexpress-a9板，

```bash
$ make ARCH=arm vexpress_defconfig
```
如果上述命令没有问题，则可以开始编译kernel，编译过程的耗时取决于host机器，大概从几分钟到半小时不等。

```bash
$ make ARCH=arm \
       CROSS_COMPILE=arm-linux-gnueabi- \
       zImage \
       LOADADDR=0x60003000
```

上面的命令中，CROSS_COMPILE=arm-linux-gnueabi- 告诉make交叉编译工具链的前缀，即c的编译器是arm-linux-gnueabi-gcc，c++编译器是arm-linux-gnueabi-g++，链接器是arm-linux-gnueabi-ld。zImage 是生成kernel image的命令。zImage是用于ARM Linux image一种压缩格式，它有vmlinux经由gzip而成。zImage文件一般存放到闪存存储上。vexpress-a9板上带有闪存。后面我们启动QEMU虚拟机的时候需要挂载它。LOADADDR=0x60003000 参数告诉QEMU的加载程序把kernel image加载到内存的0x60003000地址。

如果编译成功，则可以在目录 arch/arm/boot 下找到kernel的二进制文件zImage。

![image](/images/files_in_arch_arm_boot.png)

除了image以外，kernel中还有一些moudle也需要编译，如下，

```bash
$ make ARCH=arm \
       CROSS_COMPILE=arm-linux-gnueabi- \
       modules
```

接下来编译dtb。Linux源码目录的 arch/arm/ 目录下可以看到Linux支持的板子的类型，每个mach-开头的目录下是一个特定板子相关的代码，包括英飞凌的imx、联发科的mediatek、STM32等等。我们所用的板子对于的目录是 arch/arm/mach-vexpress。而arch/arm/boot 下面则保存了各个板子的dts文件。

![image](/images/files_in_arch_arm.png)

我们下面编译dtb的任务就要用到上述几个目录的文件，当然相关的配置已经在 make ARCH=arm vexpress_defconfig 时完成了。这时只要调用make dtbs就可以了，

```bash
$ make ARCH=arm \
       CROSS_COMPILE=arm-linux-gnueabi- \
       dtbs
```

完成后，检查 arch/arm/boot 目录，会看到 vexpress-v2p-ca9.dtb 已经编译出来了。

```bash
~/qemu/linux-4.9.313/arch/arm/boot:~$ ls dts/vexpress-v2p-ca9* -l
-rw-rw-r-- 1 isaachan isaachan 14708 May 15 16:37 dts/vexpress-v2p-ca9.dtb
-rw-rw-r-- 1 isaachan isaachan  8423 May 12 18:14 dts/vexpress-v2p-ca9.dts
```

到目前为止，kernel和dts已经就位了，虽然还没有rootfs，但是仍然可以运行下面的命令，来看看会得到什么结果，

```bash
$ qemu-system-arm -M vexpress-a9 \
                  -m 512M \
                  -kernel linux-4.9.313/arch/arm/boot/zImage \
                  -dtb linux-4.9.313/arch/arm/boot/dts/vexpress-v2p-ca9.dtb \
                  -append "console=ttyAMA0" \
                  -nographic
```

如果前面的过程都没有问题，QEMU的执行会停止在如下的输出：

![image](/images/qemu_output_1.png)

“VFS: Unable to mount root fs on unknown-block(0,0)”这行提示告诉我们，QEMU已经加载了Linux Kernel，但是没有发现挂载的rootfs。所以，接下来就要制作rootfs了。

# 制作rootfs

制作一个简单的rootfs的流程并不复杂，首先是准备rootfs的文件（执行程序，库，设备），然后制作一个rootfs的image，最后临时把image挂载到host机器上，再把编译出来的工具集复制到image上。

## 准备文件

首先准备rootfs上的文件。需要的文件分为三种，可执行程序工具集，比如mkdir vi fdisk ifconfig等等；重要的库，比如glibc；以及console和tty的设备文件。

编译一套Linxu最小化的工具集，可以从Busybox开始。Busybox项目提供很多大家熟悉的UNIX工具，不过这些工具的功能比对应的GNU版本要简单很多，同时体积也小很多，所以，Busybox非常适合为嵌入式系统提供一套完整的环境。

下面的命令首先下载并解压了busybox的代码，然后配置ARCH和CROSS_COMPILE为arm，最后通过make和make install进行编译和安装。安装成功后，Busybox默认会在自身源码目录下建立一个_install目录来存放目标的二进制软件包。

```bash
$ wget -c https://busybox.net/downloads/busybox-1.35.0.tar.bz2
$ tar xvd busybox-1.35.0.tar.bz2
$ cd busybox-1.35.0
$
$ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- \
       defconfig
$ echo “CONFIG_STATIC=y” >> .config
$ 
$ make
$ make install
```

需要注意的是“echo “CONFIG_STATIC=y” >> .config”。这行config中的配置项会在编译所有工具的时候，把依赖的library作为静态库直接打包到executable中。这样做的好处在于，在后面运行工具程序的时候，可以避免动态库的发现与链接的问题。当然，这也会导致最终软件包的大小会有膨胀。在我的laptop上测试了一下，static和shared编译生成的软件包大小分别是2MB和1.2MB，在实验环境里可以忽略不计的。

接下来可以在host环境上创建一个rootfs的目录，把所有文件都放到里面去。下面的命令里，第三行把刚刚Busybox生成的文件复制到了rootfs中。第四行命令把/usr/arm-linux-gnueabi/lib/目录下所有的文件复制到了rootfs/lib下面，这些文件就是在Linux运行的程序需要的库的最小集。

```bash
$ mkdir -p rootfs/{lib,dev}
$ cd rootfs
$ cp [PATH_OF_BUSYBOX]/_install/* .
$ cp /usr/arm-linux-gnueabi/lib/* lib/
```

最后，通过下面的命令，我们为rootfs在创建四个设备文件，一个null，一个终端（console），两个串口节点（tty1/tty2）。这些对于简单的实验环境已经足够了。

```bash
$ mkmod -m 666 rootfs/dev/null 1 3
$ mkmod -m 666 rootfs/dev/tty1 4 0
$ mkmod -m 666 rootfs/dev/tty2 4 1
$ mknod -m 666 rootfs/dev/console 5 1
```

## 制作rootfs image

首先，创建一个“空”的image，

```bash
$ dd if=/dev/zero of=rootfs.ext3 bs=1M count=64
```

上述命令会读取/dev/zero设备，每次读1M，一共读取64次，输出到文件rootfs.ext3中。/dev/zero作为一个特殊的虚拟设备，它每次被读取时都返回0。命令结束后，会有一个67M左右的rootfs.ex3文件。

接下来，将image文件格式化为ext3，就像它的后缀名所示的那样，

```bash
$ mkfs.ext3 rootfs.ext3

mke2fs 1.45.5 (07-Jan-2020)
Discarding device blocks: done
Creating filesystem with 16384 4k blocks and 16384 inodes

Allocating group tables: done
Writing inode tables: done
Creating journal (1024 blocks): done
Writing superblocks and filesystem accounting information: done
```


## 完成rootfs

最后，把新创建的image挂载到host机器上，然后把rootfs下的文件复制到挂载image的分区里面，

```bash
$ mount -t ext3 rootfs.ext3 /mnt
$ cp rootfs/* /mnt/
$ umount /mnt
```

至此，rootfs.ext3文件就可以作为QEMU虚拟机的rootfs了。利用下面的命令，启动ARM Linux虚拟机，

```bash
$ qemu-system-arm -M vexpress-a9 \
                  -m 512M \
                  -kernel linux-4.9.313/arch/arm/boot/zImage \
                  -dtb linux-4.9.313/arch/arm/boot/dts/vexpress-v2p-ca9.dtb \
                  -sd rootfs.ext3 \
                  -append "console=ttyAMA0 root=/dev/mmcblk0" \
                  -nographic
```

几秒钟后，如果看到类似的输出，那么说明QEMU已经成功地运行了ARM Linux虚拟机。

![image](/images/qemu_output_2.png)


# 使用u-boot启动

在实际的嵌入式环境中，很多时候是需要bootloader来加载系统。本文的最后一部分，我们使用u-boot作为bootloader，加载并运行ARM Linux Image。

使用u-boot的第一步是编译u-boot。另外，u-boot加载的Linux Image的格式是uImage，因此我们需要回到Linux源码目录，再编译一个uImage文件。u-boot启动后，会通过TFTP或者NFS下载Linux Image。在本文中，我们使用TFTP来演示u-boot。因此，第二步要设置QMEU可以访问的TFTP服务。最后，加载u-boot，执行boot command，下载并启动Linux Image。

这里，执行boot command有几种方式，一是设置u-boot源码中的BOOTCMD宏，这种方式的自动化程度最高，不过本文并不会使用。本文会让u-boot启动，然后进入u-boot的CLI模式，在那里依次执行boot command中的多条指令。这么做的目的是为了最大程度地学习u-boot的启动细节。

## 编译u-boot

```bash
$ wget -c https://ftp.denx.de/pub/u-boot/u-boot-2022.04.tar.bz2
$ tar xvf u-boot-2022.04.tar.bz2
$ cd u-boot-2022.04
$ 
$ make vexpress_ca9x4_defconfig
$ # make menuconfig （非必要的。这里可以设置bootcmd，以及修改vexpress_ca9x4_defconfig的默认设置）
$ make ARCH=arm CROSS_COMPILE=arm-none-eabi-   
$ 
$ ls -l u-boot
-rwxrwxr-x 1 xxxx xxxx 5279848 May 23 00:09 u-boot
```

如果编译过程没有错误，就可以看到一个u-boot文件。

## 编译uImage格式的Linux Kernel
```bash
$ make ARCH=arm \
       CROSS_COMPILE=arm-linux-gnueabi- \
       uImage \
       LOADADDR=0x60003000
```
编译uImage和编译zImage的几乎是一样的，只是调用了make uImage而不是zImage任务而已。

如果前面的zImage文件还在，那么这个过程会非常快，因为uImage文件只是在zImage文件上加了一个64字节的header。这样的uImage，u-boot称其为Legacy Image，与之对应的另一种是FIT-image (Flattened uImage Tree)，它会将Kernel、dtb和rootfs全部打包到一个image文件中。本文我们使用的是Legacy Image。

## 设置TFTP

本文不赘述设置TFTP的过程。网上有很多相关的文档。当设置了TFTP后，把uImage文件和vexpress-v2p-ca9.dtb文件（vexpress板子的dtb）交给TFTP托管。

如果TFTP运行在host机器，那个可以通过下面的方式测试TFTP，

```bash
$ tftp [HOST IP]
tftp> get uImage
tfpt> get vexpress-v2p-ca9.dtb
```

如果能够通过TFTP下载uImage和vexpress-v2p-ca9.dtb，则说明TFTP已经准备好了。

## 加载u-boot，执行bootcmd

```bash
$ qemu-system-arm -M vexpress-a9 \
                  -m 512M \
                  -kernel u-boot-2022.04/u-boot 
                  -nographic \
                  -sd rootfs.ext3
```

这一次，kernel已经不是zImage了，而是u-boot。dtb文件也不需要了，而是u-boot从TFTP下载。上面的命令执行后会遇到下面类似的错误，

```bash
Usage:
cp [.b, .w, .l, .q] source target count
Wrong Image Format for bootm command
ERROR: can't get kernel image!
=>
```

因为没有设置bootcmd，所以u-boot没有加载kernel，于是报了错就进入交互模式了。接下来，我们会依次在u-boot的命令行中执行启动板子的命令，

```bash
=> setenv serverip [TFTP SERVER ADDR]
=> tftp 0x60003000 uImage
smc911x: detected LAN9118 controller
smc911x: phy initialized
smc911x: MAC 52:54:00:12:34:56
Using ethernet@3,02000000 device
TFTP from server 192.168.0.110; our IP address is 10.0.2.15; sending through gateway 10.0.2.2
Filename ‘uImage'.
Load address: 0x60003000
Loading: #################################################################
     #################################################################
     #################################################################
     ###############################################
     2.3 MiB/s
done
Bytes transferred = 3543744 (3612c0 hex)
smc911x: MAC 52:54:00:12:34:56
=> tftp 0x60500000 vexpress-v2p-ca9.dtb
smc911x: detected LAN9118 controller
smc911x: phy initialized
smc911x: MAC 52:54:00:12:34:56
Using ethernet@3,02000000 device
TFTP from server 192.168.0.110; our IP address is 10.0.2.15; sending through gateway 10.0.2.2
Filename 'vexpress-v2p-ca9.dtb'.
Load address: 0x60500000
Loading: ##
     13.7 KiB/s
done
Bytes transferred = 14708 (3974 hex)
smc911x: MAC 52:54:00:12:34:56
```

第一行命令设置TFTP的地址，第二行命令通过TFTP下载uImage文件，并把它加载到内存的0x60003000的位置。后面的输出表示uImage已经从TFTP下载成功了。第三行命令（tftp 0x60500000 vexpress-v2p-ca9.dtb）下载了dtb文件，并把它加载到内存的0x6050000的位置。

接下来的两个命令，首先设置了启动参数，包括挂载mmc设备，以及控制台的重定向，然后通过bootm命令启动Linux，

```bash
=> setenv bootargs 'root=/dev/mmcblk0 console=ttyAMA0'
=> bootm 0x60003000 - 0x60500000
```

几秒钟之后，你会看到和之前一样的欢迎画面，

![image](/images/qemu_output_3.png)

