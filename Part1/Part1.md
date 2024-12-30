# UFI801 固件构建笔记

## Part1：内核编译

#### 一、环境构建
本机使用Deepin V23进行构建，安装依赖：
```
sudo apt install binfmt-support qemu-user-static fakeroot mkbootimg bison flex gcc-aarch64-linux-gnu pkg-config libncurses-dev libssl-dev unzip git android-sdk-libsparse-utils
```

#### 二、下载源码
使用`msm8916-mainline`提供的内核源码：
```
git clone git@github.com:msm8916-mainline/linux.git --depth=1
```
进入目录后进行编译：
```
export CROSS_COMPILE=aarch64-linux-gnu-
export ARCH=arm64
make mrproper
make O=out msm8916_defconfig
```
如果需要对内核进行裁剪或者增添的话请执行：
```
make O=out menuconfig
```
比如说我需要开启对光驱设备的支持，那么我就需要打开：
```
CONFIG_CDROM=y
CONFIG_BLK_DEV_SR=y
CONFIG_ISO9660_FS=y
CONFIG_JOLIET=y
CONFIG_ZISOFS=y
CONFIG_UDF_FS=y
CONFIG_CRC_ITU_T=y
```
开始编译：
```
make -j$(nproc) O=out
```
PS：需要超频可以参考此[commit](https://github.com/windbell-project/kernel_ufi_msm8916/commit/4adbea3cb94ab9a584e703950303091f163ce228)

参考过网上许多教程，有使用以下两条命令的：
```
make -j`nproc` O=out deb-pkg
make -j`nproc` O=out bindeb-pkg
```
目的就是在于生成`deb`格式的内核安装包，在Debian系的Rootfs中能够直接安装，但我在编译`kernel-6.1.x`的时候一直遇到：

`dpkg-checkbuilddeps: error: Unmet build dependencies: libssl-dev`
经检查，在Deepin V23下安装了`libssl-dev`，推测是某些内核选项没有开启。

无妨，可以手搓所需的文件，启动流程简单分析就是`lk2nd`引导`boot`分区中的内核，内核依靠`cmdline`启动对应`UUID`的`rootfs`分区，最终完成启动。我们所需的也就是：`kernel`、`initrd.img`、`dtb`、`vmlinuz`这几个东西。

因为`msm8916`平台已mainline，所以建议从其他已有的`rootfs`进行定制，也是后文的内容。

#### 三、构建相关文件
1.Kernel

`out/arch/arm64/boot/Image.gz`就是我们所需的内核，目录下还有个`Image`，使用 gzip 算法直接对 Image 进行压缩即可生成`Image.gz`。

2.dtb

对于UZ801设备来说，`dtb`的路径为：

```
out/arch/arm64/boot/dts/qcom/msm8916-yiming-uz801v3.dtb
```

3.合并`kernel`和`dtb`

在`uboot`支持的情况下，`dtb`是可以`append`在`kernel`之后的：

```
cat Image.gz msm8916-yiming-uz801v3.dtb>kernel
```

我们可以得到`kernel`，这即是我们所需要的内核。

4.boot.img
对于现有的固件，直接使用`Android-Image-Kitchen`解包，使用`kernel`替换掉原有的内核然后打包。

5.Initrd.img和其余产物

这一步的本质就是使用`chroot`在`rootfs`中执行相关的操作，对于`x86`架构的机器我们需要安装`qemu-user-static`相关的组件，教程可以搜索到。我是使用的`Arm`架构的`Archlinux`，所以我直接使用`arch-chroot`这一工具，包含于`arch-install-scripts`这一脚本中，可以使用如下语句安装：
```
sudo apt install arch-install-scripts
```

然后转换`rootfs`的格式并挂载:
```
mkdir rootfs
simg2img rootfs.img rootfs.raw
sudo mount rootfs.raw rootfs
```
删除掉`rootfs`根目录下`boot`目录中的文件：
```
sudo rm -rf rootfs/boot/*
```
在`kernel`源码目录下执行，安装编译产生的`modules`、`vmlinuz`:
```
sudo make ARCH=arm64 O=out CROSS_COMPILE=aarch64-linux-gnu-  INSTALL_MOD_PATH=<rootfs挂载目录的路径> modules_install

sudo make ARCH=arm64 O=out INSTALL_PATH=<rootfs挂载目录的路径>/boot install
```
`chroot`进入`rootfs`：
```
sudo arch-chroot rootfs su -
```
生成`initrd.img`和其他所需文件：
```
kerver=$(ls /usr/lib/modules)
mkinitcpio --generate /boot/initrd.img-$kerver --kernel $kerver
```
PS：对于ubuntu可以使用：
```
update-initramfs -k all -c
```

卸载`rootfs`：
```
sudo umount rootfs
```
转换`rootfs`为`simg`格式以缩小体积：
```
img2simg rootfs.raw rootfs-new.img
```
然后刷入即可