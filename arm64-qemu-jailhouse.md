# QEMU simulates ARM64 kernel

## 1. Install the cross compiler aarch64-none-linux-gnu- 10.3

Website: [https://developer.arm.com/downloads/-/gnu-a](https://developer.arm.com/downloads/-/gnu-a)

Tool selection: AArch64 GNU/Linux target (aarch64-none-linux-gnu)

Download link: [https://developer.arm.com/-/media/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz?rev=1cb9c51b94f54940bdcccd791451cec3&hash=B380A59EA3DC5FDC0448CA6472BF6B512706F 8EC](https://developer.arm.com/-/media/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-aa rch64-none-linux-gnu.tar.xz?rev=1cb9c51b94f54940bdcccd791451cec3&hash=B380A59EA3DC5FDC0448CA6472BF6B512706F8EC)

```bash
wget https://armkeil.blob.core.windows.net/developer/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz
tar xvf gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu.tar.xz
ls gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/
```

After the installation is complete, remember the path, for example: /home/tools/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-, and you will use this path later.

## 2. Compile and install QEMU 7.0

```
# Install the required dependency packages for compilation
sudo apt install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev \
gawk build-essential bison flex texinfo gperf libtool patchutils bc \
zlib1g-dev libexpat-dev pkg-config libglib2.0-dev libpixman-1-dev libsdl2-dev \
git tmux python3 python3-pip ninja-build
# Download source code
wget https://download.qemu.org/qemu-7.0.0.tar.xz
# Unzip
tar xvJf qemu-7.0.0.tar.xz
cd qemu-7.0.0
# Generate configuration files
./configure --enable-kvm --enable-slirp --enable-debug --target-list=aarch64-softmmu,x86_64-softmmu
#Compile
make -j$(nproc)
```

Then edit the `~/.bashrc` file and add a few lines at the end of the file:

```
# Please note that the parent directory of qemu-7.0.0 can be flexibly adjusted according to your actual installation location
export PATH=$PATH:/path/to/qemu-7.0.0/build
```

Then you can update the system path in the current terminal `source ~/.bashrc`, or directly restart a new terminal. At this point, you can confirm the qemu version:

```
qemu-system-aarch64 --version # View version
```

> Note that the above dependency packages may not be complete, for example:
>
> - When `ERROR: pkg-config binary 'pkg-config' not found` appears, you can install the `pkg-config` package;
> - When `ERROR: glib-2.48 gthread-2.0 is required to compile QEMU` appears, you can install the `libglib2.0-dev` package;
> - When `ERROR: pixman >= 0.21.8 not present` appears, you can install the `libpixman-1-dev` package.

## 3. Compile Kernel 5.4

```bash
git clone https://github.com/torvalds/linux -b v5.4 --depth=1
cd linux
git checkout v5.4
make ARCH=arm64 CROSS_COMPILE=/root/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- defconfig
```

In order to enable Linux to generate a /dev/ram0 for subsequent use, you need to edit the local .config file and add a line:

```
CONFIG_BLK_DEV_RAM=y
```

Compile afterwards:

```shell
make ARCH=arm64 CROSS_COMPILE=/root/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- Image -j$(nproc)
```

> If an error occurs when compiling Linux:
>
> ```
> /usr/bin/ld: scripts/dtc/dtc-parser.tab.o:(.bss+0x20): multiple definition of 'yylloc'; scripts/dtc/dtc-lexer.lex.o:(.bss+0x0): first defined here
> ```
>
> Then modify `scripts/dtc/dtc-lexer.lex.c` in the Linux folder and add `extern` before `YYLTYPE yylloc;`. Compile again and you will get an error: openssl/bio.h: No such file or directory. Now execute `sudo apt install libssl-dev`

Compile and the kernel file is located at: arch/arm64/boot/Image. Remember the path of the entire linux folder, for example: home/korwylee/lgw/hypervisor/linux.

## 4. Build a file system based on ubuntu 20.04 arm64 base

The file system created by busybox is too simple (such as no apt tool), so we need to use a richer ubuntu file system to create the root file system of linux. Note that ubuntu22.04 is also OK.

Download: [ubuntu-base-20.04.5-base-arm64.tar.gz](http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz)

Link: [http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz](http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz)

```bash
wget http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.5-base-arm64.tar.gz

mkdir rootfs
# Create an ubuntu.img
dd if=/dev/zero of=ubuntu-20.04-rootfs_ext4.img bs=1M count=4096 oflag=direct
mkfs.ext4 ubuntu-20.04-rootfs_ext4.img
# Put ubuntu.tar.gz into the ubuntu.img mounted on rootfs
sudo mount -t ext4 ubuntu-20.04-rootfs_ext4.img rootfs/
sudo tar -xzf ubuntu-base-20.04.5-base-arm64.tar.gz -C rootfs/

# Let rootfs bind and obtain some information and hardware of the physical machine
# qemu-path is your qemu path
sudo cp qemu-path/build/qemu-system-aarch64 rootfs/usr/bin/
sudo cp /etc/resolv.conf rootfs/etc/resolv.conf
sudo mount -t proc /proc rootfs/proc
sudo mount -t sysfs /sys rootfs/sys
sudo mount -o bind /dev rootfs/dev
sudo mount -o bind /dev/pts rootfs/dev/pts

# Executing this command may result in an error, please refer to the following solution
sudo chroot rootfs

apt-get update
apt-get install git sudo vim bash-completion -y
apt-get install net-tools ethtool ifupdown network-manager iputils-ping -y
apt-get install rsyslog resolvconf udev -y

# If the above packages are not installed, at least install the following packages
apt-get install systemd -y

apt-get install build-essential git wget flex bison libssl-dev bc libncurses-dev kmod -y

adduser arm64
adduser arm64 sudo
echo "kernel-5_4" >/etc/hostname
echo "127.0.0.1 localhost" >/etc/hosts
echo "127.0.0.1 kernel-5_4">>/etc/hosts
dpkg-reconfigure resolvconf
dpkg-reconfigure tzdata
exit

sudo umount rootfs/proc
sudo umount rootfs/sys
sudo umount rootfs/dev/pts
sudo umount rootfs/dev
sudo umount rootfs
```

Finally unmount and mount
, complete the creation of the root file system.

> When executing `sudo chroot .`, if the error `chroot: failed to run command ‘/bin/bash’: Exec format error` is reported, you can execute the command:
>
> ```
> sudo apt-get install qemu-user-static
> sudo update-binfmts --enable qemu-aarch64
> ```

## 5. Compile jailhouse

```bash
sudo mount ubuntu-20.04-rootfs_ext4.img rootfs/
# Enter a folder in rootfs
```

Compile rootfs:

```bash
# for x86
git clone https://github.com/siemens/jailhouse.git
cd jailhouse
make

# for aarch64
git clone https://github.com/siemens/jailhouse.git
cd jailhouse
git checkout v0.10
# If you want to run the hypervisor with the compiled jailhouse, you need to execute the next instruction to patch; otherwise, please do not do this operation
patch -f -p1 < hypervisor/hypervisor.patch #See the following instructions, which contain the warehouse address
# KDIR is the previously compiled linux folder
make ARCH=arm64 CROSS_COMPILE=/home/tools/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- KDIR=/home/user/lgw/hypervisor/linux

sudo make ARCH=arm64 CROSS_COMPILE=/home/tools/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- KDIR=/home/user/lgw/hypervisor/linux DESTDIR=/home/user/lgw/hypervisor/linux install
```

Note: When compiling jailhouse, you must specify KDIR and sysroot target to compile successfully. You also need to compile a linux as sysroot in advance. Otherwise, the corresponding library code will be found in the local linux source directory by default.

> the hypervisor is located at https://github.com/saltytine/hypervisor

## 6. Start QEMU

```bash
qemu-system-aarch64 \
-machine virt,gic_version=3 \
-machine virtualization=true \
-cpu cortex-a57 \
-machine type=virt \
-nographic\
-smp 16 \
-m 1024 \
-kernel ./linux/arch/arm64/boot/Image \
-append "console=ttyAMA0 root=/dev/vda rw mem=768m" \
-drive if=none,file=ubuntu-20.04-rootfs_ext4.img,id=hd0,format=raw \
-device virtio-blk-device,drive=hd0 \
-netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
-device virtio-net-device,netdev=net0,mac=52:55:00:d1:55:01
```

The kernel is the previously compiled Linux image.

## VII. Run jailhouse

```shell
cd ~/jailhouse
sudo mkdir -p /lib/firmware
sudo cp hypervisor/jailhouse.bin /lib/firmware/
sudo insmod driver/jailhouse.ko
sudo ./tools/jailhouse enable configs/arm64/qemu-arm64.cell
# To shut down jailhouse, execute the following command
sudo ./tools/jailhouse disable
```

Start a gic-demo:

```shell
sudo ./tools/jailhouse cell create configs/arm64/qemu-arm64-gic-demo.cell
sudo ./tools/jailhouse cell load gic-demo inmates/demos/arm64/gic-demo.bin
sudo ./tools/jailhouse cell start gic-demo
sudo ./tools/jailhouse cell destroy gic-demo
```

Frequently Asked Questions:

> 1. If jailhouse reports an error saying that the glibc version is low, chroot to the rootfs on the host to change the source and add the Tsinghua source for Ubuntu 22.04. The steps are as follows:
>
> Execute the command
>
> ```
> sudo vi /etc/apt/sources.list
> ```
>
> Replace the file content with:
>
> ```
> # The source mirror is commented by default to improve the apt update speed. You can uncomment it if necessary.
> deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal main restricted universe multiverse
> # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal main restricted universe multiverse
> deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-updates main restricted universe multiverse
> # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-updates main restricted universe multiverse
> deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-backports main restricted universe multiverse
> # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-backports main restricted universe multiverse
>
> # deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-security main restricted universe multiverse
> # # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-security main restricted universe multiverse
>
> deb http://ports.ubuntu.com/ubuntu-ports/ focal-security main restricted universe multiverse
> # deb-src http://ports.ubuntu.com/ubuntu-ports/ focal-security main restricted universe multiverse
>
> # Pre-release software source, not recommended to be enabled
> # deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-proposed main restricted universe multiverse
> # # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-proposed main restricted universe multiverse
> deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ jammy main
> ```
>
> After replacement, execute:
>
> ```
> sudo apt update
> sudo apt install libc6
> ```
>
> 2. If the ext4 error file system error is reported when running jailhouse, you can execute on the host:
>
> ```
> e2fsck -f ubuntu-20.04-rootfs_ext4.img
> ```

## 8. Start a non-root-linux on qemu

When non-root-linux starts, the file system needs to be mounted. The following tutorial is divided into memory file system and disk file system.

### 8.1 Memory file system

#### 8.1.1 Compile busybox 1.36.0 (file system)

```bash
wget https://busybox.net/downloads/busybox-1.36.0.tar.bz2
tar -jxvf busybox-1.36.0.tar.bz2

cd busybox-1.36.0
sudo apt-get install libncurses5-dev
make menuconfig
```

In the pop-up window, enter and set:

* Settings
* [*] Build static binary (no shared libs)
* Cross compiler prefix is set to: `/cross compiler path/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-`

```bash
# Compile and install
make -j$(nproc) && make install
```

After the compilation is complete, busybox is generated in the _install directory.

#### 8.1.2 Create a console for the file system

```bash
cd _install
mkdir dev
cd dev
sudo mknod console c 5 1
sudo mknod null c 1 3
sudo mknod tty1 c 4 1
sudo mknod tty2 c 4 2
sudo mknod tty3 c 4 3
sudo mknod tty4 c 4 4
cd ..
mkdir -p etc/init.d/
cd etc/init.d;
touch rcS
chmod +x rcS
vi rcS

#Return to the _install directory
cd ../..
# Compress into cpio.gz file system
find . -print0 | cpio --null -ov --format=newc | gzip -9 > initramfs.cpio.gz
```

Then mount the root file system and transfer initramfs.cpio.gz to the guest linux. Then start QEMU and enable jailhouse.

> If an error message is displayed saying that python cannot be found, execute the following command:
>
> ```
> sudo ln -s /usr/bin/python3 /usr/bin/python
> ```

Then execute:

```bash
cd tools/
sudo ./jailhouse cell linux ../configs/arm64/qemu-arm64-linux-demo.cell ../linux-Image -c "console=ttyAMA0,115200 root=/dev/ram0 rdinit=/linuxrc" -d ../configs/arm64/dts/inmate-qemu-arm64.dtb -i ../initramfs.cpio.gz
```

linux-Image is the previously compiled linux image.

### 8.2 Disk file system

Make a new ubuntu image:

> To save time, only a simple image is made here. For a more complete image file, refer to Chapter 4.

```bash
dd if=/dev/zero of=second-ubuntu-20.04-rootfs_ext4.img bs=1M count=256 oflag=direct
mkfs.ext4 second-ubuntu-20.04-rootfs_ext4.img
sudo mount -t ext4 second-ubuntu-20.04-rootfs_ext4.img rootfs/
sudo tar -xzf ubuntu-base-20.04.5-base-arm64.tar.gz -C rootfs/
sudo cp qemu-path/build/qemu-system-aarch64 rootfs/usr/bin/
sudo cp /etc/resolv.conf rootfs/etc/resolv.conf
```

To allow non-root-linux to start a disk file system, it should be able to find the corresponding disk in the virtio-blk device simulated by qemu through the device tree.

In order to find the virtio-blk device simulated by qemu, you need to export the device tree information simulated by qemu:

```bash
sudo qemu-system-aarch64 \
-machine virt,gic_version=3 \
-machine virtualization=true \
-cpu cortex-a57 \
-machine type=virt \
-nographic \
-smp 16 \
-m 1024 \
-kernel ./linux/arch/arm64/boot/Image \
-append "console=ttyAMA0 root=/dev/vda rw mem=768m" \
-drive if=none,file=second-ubuntu-20.04-rootfs_ext4.img,id=hd1,format=raw \
-device virtio-blk-device,drive=hd1 \
-drive if=none,file=ubuntu-20.04-rootfs_ext4.img,id=hd0,format=raw \
-device virtio-blk-device,drive=hd0 \
-netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
-device virtio-net-device,netdev=net0,mac=52:55:00:d1:55:01 \
-machine dumpdtb=qemu-virt.dtb
```

Machine dumpdtb exports the qemu device tree to a dtb file, which needs to be converted into a readable dts file. Execute in the shell:

```bash
dtc -I dtb -O dts -o qemu-virt.dts qemu-virt.dtb
```

The virtio devices specified by qemu are in reverse order in the device tree. Since we specify second-ubuntu-20.04-rootfs_ext4.img as the non-root-linux disk, it is the first device in the qemu startup parameters, so the last virtio-mmio area is found:

```bash
virtio_mmio@a003e00 {
dma-coherent;
interrupts = <0x00 0x2f 0x01>;
reg = <0x00 0xa003e00 0x00 0x200>;
compatible = "virtio,mmio";
};
```

This shows that the mmio area where the disk is located starts at 0xa003e00, has a size of 0x200, and uses the interrupt 0x2f of the SPI interrupt, that is, interrupt 32+47=79. Write it to inmate-qemu-arm64.dts, and add the corresponding mem region to the cell configuration file of non-root-linux:

```c
/*disk*/ {
.phys_start = 0x0a003e00,
.virt_start = 0x0a003e00,
.size = 0x0200,
.flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE |
JAILHOUSE_MEM_IO | JAILHOUSE_MEM_IO_32 | JAILHOUSE_MEM_IO_8 | JAILHOUSE_MEM_IO_16 | JAILHOUSE_MEM_IO_64,
},
```

Since non-root-linux uses interrupt 79 when probing virtio-blk, modify the cell configuration file:

```c
.irqchips = {
/* GIC */ {
.address = 0x08000000,
.pin_base = 32, // pin_base indicates the number of interrupts to start from, the type of pin_bitmap is u32[4],
.pin_bitmap = { // Each element represents 32 interrupts, and the interrupts with bits set to 1 will be cancelled by the root cell
(1 << (33 - 32)),
(1 << 15), // Interrupt No. 79
0,
(1 << (140 - 128))
},
},
},
```

Then compile jailhouse and start qemu:

```bash
sudo qemu-system-aarch64 \
-machine virt,gic_version=3 \
-machine virtualization=true \
-cpu cortex-a57 \
-machine type=virt \
-nographic \
-smp 16\
-m 1024 \
-kernel ./linux/arch/arm64/boot/Image \
-append "console=ttyAMA0 root=/dev/vda rw mem=768m" \
-drive if=none,file=second-ubuntu-20.04-rootfs_ext4.img,id=hd1,format=raw \
-device virtio-blk-device,drive=hd1 \
-drive if=none,file=ubuntu-20.04-rootfs_ext4.img,id=hd0,format=raw \
-device virtio-blk-device,drive=hd0 \
-netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
-device virtio-net-device,netdev=net0,mac=52:55:00:d1:55:01
```

Then start non-root-linux:

```bash
sudo insmod driver/jailhouse.ko
sudo ./tools/jailhouse enable configs/arm64/qemu-arm64.cell
cd tools/
sudo ./jailhouse cell linux ../configs/arm64/qemu-arm64-linux-demo.cell ../linux-Image -c "console=ttyAMA0,115200 root=/dev/vda rw" -d ../configs/arm64/dts/inmate-qemu-arm64.dtb
```

## Appendix

### 1. Configure qemu network to communicate with the outside world

```bash
+-----------------------------------------------------------------+
|  Host                                                           |
| +---------------------+                                         |
| |                     |                                         |
| | br0:                |                                         |
| |   192.168.0.32/24   +-----+                                   |
| |                     |     |                                   |
| +----+----------------+     |       +-------------------------+ |
|      |                      |       |  Guest                  | |
|      |                      |       | +---------------------+ | |
| +----+----------------+  +--+---+   | |                     | | |
| |                     |  |      |   | | eth0:               | | |
| | eth1:               |  | tap0 |   | |   192.168.0.33/24   | | |
| |   192.168.0.175/24  |  |      +-----+                     | | |
| |                     |  |      |   | +---------------------+ | |
| +---------------------+  +------+   +-------------------------+ |
+-----------------------------------------------------------------+
```

Network connection diagram

Configure bridge `br0`

```bash
sudo ip link add name br0 type bridge
sudo ip link set dev br0 down
sudo ip addr flush dev br0
sudo ip addr add 192.168.0.32/24 dev br0
sudo ip link set dev br0 up
```

Configure tap device `tap0`

```bash
sudo ip tuntap add name tap0 mode tap
sudo ip link set dev tap0 up
```

Connect the host network interface `eth0` and `tap0` to the bridge `br0`

```bash
sudo ip link set eth1 master br0
sudo ip link set tap0 master br0
```

Then qemu starts the virtual machine, in the virtual machine

```bash
sudo ifconfig eth0 up
sudo ifconfig eth0 192.168.0.33
```

Now the guest can ping the host

#### Optimize the network so that it can communicate with the outside world (not trustworthy yet)

After completing the above steps, the virtual machine can be interconnected with other devices in the host network (including the host), and can also connect to the Internet through the specified gateway, but the host cannot connect to the Internet at this time. The solution is as follows:

Delete the default gateway of the `eth0` interface:

```bash
sudo ip route del default dev eth1
```

Add a default gateway for `br0`:

```bash
sudo ip route add default via 192.168.0.1 dev br0
```

### 2. Img expansion

When rootfs When the chroot space is insufficient, you need to expand the capacity. Follow the steps below to perform lossless expansion:

```bash
# Unmount img first
umount ./rootfs

## bug: umount: /root/rootfs: target is busy.
## Solution: umount ./rootfs -l Forced unmount (use with caution)

dd if=/dev/zero of=add.img bs=1M count=4096 # Create a new 4G space
cat add.img >> ubuntu-20.04-rootfs_ext4.img
e2fsck -f ubuntu-20.04-rootfs_ext4.img
resize2fs ubuntu-20.04-rootfs_ext4.img

mount ubuntu-20.04-rootfs_ext4.img rootfs
```
