内核编译
==================

不幸的是，标准的Android内核并不足以运行Droidian。

好消息是，在支持GSI的设备上，通常只需要对内核进行修改。

目录
-----------------

* [摘要](#摘要)
* [先决条件](#先决条件)
* [包引入](#包引入)
* [内核信息选项](#内核信息选项)
* [内核适配](#内核适配)
* [编译](#编译)
* [获取启动镜像](#获取启动镜像)
* [提交更改](#提交更改)

摘要
-------

Droidian运行在[Halium](https://halium.org)上。
如果您的设备搭载Android 9或以上版本，那么有可能将Droidian移植到该设备上。
如果它已经有了符合Halium标准的内核，版本为halium-9.0或以上，那么Droidian很有可能无需太多修改就能工作。

如果您的设备搭载的是Android 8.1或9，它很可能支持GSI（通用系统镜像），因此可以使用已经可用的、带有Halium补丁的通用Android系统镜像。
这将显著减少移植者需要做的工作。

如果您的设备不支持GSI，您还需要编译一个打了补丁的系统镜像。
关于镜像创建的文档可以在[这个指南](./image-creation.md)中找到。

在Halium上，Android内核是通过标准的Android工具链构建的。
虽然这是有意义的，但在支持GSI的设备上，这可能是浪费时间，因为通常只需要修改内核。

因此，Droidian采用了一种不同的方法来编译内核——好处是您可以免费获得打包，以便内核可以通过APT进行空中升级，如果您愿意的话。

请注意，本指南假设您将在x86_64（amd64）机器上交叉编译arm64 Android内核，使用Droidian仓库中提供的Android预编译工具链。
禁用交叉编译并使用标准Debian工具链进行编译是微不足道的。

使用这种方法，您还可以编译和打包主线内核。

使用本指南打包的一个示例内核是[Android Kernel for the F(x)tec Pro1](https://github.com/droidian-devices/linux-android-fxtec-pro1/tree/bookworm/debian)。

先决条件
-------------

* 设备内核源代码
* 符合Halium标准的内核defconfig
* Docker

如果您还没有符合Halium标准的内核，您应该根据本指南后面的[内核适配](#内核适配)部分的建议修改您的内核配置。

包引入
----------------

假设您的内核源代码位于`~/droidian/kernel/vendor/device`，
您应该创建一个新的分支来包含Debian打包。

我们还假设您希望在`~/droidian/packages`中得到结果包。

Droidian工具期望内核源代码具有一个工作良好的git目录结构（从一个git仓库克隆的内核），并且分支以Debian代号命名，如`bookworm`。

	（主机）$ KERNEL_DIR="$HOME/droidian/kernel/vendor/device"
	（主机）$ PACKAGES_DIR="$HOME/droidian/packages"
	（主机）$ mkdir -p $PACKAGES_DIR
	（主机）$ cd $KERNEL_DIR
	（主机）$ git checkout -b bookworm

现在是时候启动Docker容器了。

	（主机）$ docker run --rm -v $PACKAGES_DIR:/buildd -v $KERNEL_DIR:/buildd/sources -it quay.io/droidian/build-essential:bookworm-amd64 bash

在Docker容器内，安装`linux-packaging-snippets`，
它提供了示例`kernel-info.mk`文件。

	（docker）# apt-get install linux-packaging-snippets

并创建骨架打包：

	（docker）# cd /buildd/sources
	（docker）# mkdir -p debian/source
	（docker）# cp -v /usr/share/linux-packaging-snippets/kernel-info.mk.example debian/kernel-info.mk
	（docker）# echo 13 > debian/compat
	（docker）# echo "3.0 (native)" > debian/source/format
	（docker）# cat > debian/rules <<EOF
	#!/usr/bin/make -f
	
	include /usr/share/linux-packaging-snippets/kernel-snippet.mk
	
	%:
		dh \$@
	EOF
	（docker）# chmod +x debian/rules

现在编辑`debian/kernel-info.mk`以匹配您的内核设置。

大多数默认设置对于使用Pie工具链构建Android内核已经足够，所以您可能需要更改设备特定的设置（例如供应商、名称、cmdline、defconfig以及各种偏移量）。

通过使用unpackbootimg解压已经构建的`boot.img`，可以找到所有偏移量。

内核信息选项
-------------------

unpackbootimg的语法如下

	（docker）# unpackbootimg --boot_img boot.img

或对于AOSP版本的unpackbootimg

	（docker）# unpackbootimg -i boot.img

### kernel-info.mk条目

* `KERNEL_BASE_VERSION`是内核版本，可以在内核源代码根目录的Makefile中查看。

例如

```
VERSION = 4
PATCHLEVEL = 14
SUBLEVEL = 221
```

将是4.14.221

* `KERNEL_DEFCONFIG`是在arch/YOURARCH/configs中找到的defconfig文件名

* `KERNEL_IMAGE_WITH_DTB`决定是否在内核中包含dtb文件。如果这个选项被设置，也需要设置`KERNEL_IMAGE_DTB`。如果没有设置，将尝试找到它。

* `KERNEL_IMAGE_DTB`是dtb文件的路径，可以在arch/YOURARCH/boot/dts/SOC/中找到

* `KERNEL_IMAGE_WITH_DTB_OVERLAY`决定是否构建dtbo文件。如果这个选项被设置，也需要设置`KERNEL_IMAGE_DTB_OVERLAY`。如果没有设置，将尝试找到它。

* `KERNEL_IMAGE_DTB_OVERLAY`是dtbo文件的路径，可以在arch/YOURARCH/boot/dts/SOC/中找到

所有这些值都可以通过使用unpackbootimg解压启动镜像来查看

* `KERNEL_BOOTIMAGE_CMDLINE`对应于“命令行参数”或“BOARD_KERNEL_CMDLINE”`console=tty0`和`droidian.lvm.prefer`应该附加到cmdline。确保从cmdline中删除任何`systempart`条目。

* `KERNEL_BOOTIMAGE_PAGE_SIZE`对应于“页面大小”或“BOARD_PAGE_SIZE”

* `KERNEL_BOOTIMAGE_BASE_OFFSET`对应于“基础”或“BOARD_KERNEL_BASE”

* `KERNEL_BOOTIMAGE_KERNEL_OFFSET`对应于“内核加载地址”或“BOARD_KERNEL_OFFSET”

* `KERNEL_BOOTIMAGE_INITRAMFS_OFFSET`对应于“ramdisk加载地址”或“BOARD_RAMDISK_OFFSET”

* `KERNEL_BOOTIMAGE_SECONDIMAGE_OFFSET`对应于“第二引导加载地址”或“BOARD_SECOND_OFFSET”

* `KERNEL_BOOTIMAGE_TAGS_OFFSET`对应于“内核标签加载地址”或“BOARD_TAGS_OFFSET”

* `KERNEL_BOOTIMAGE_DTB_OFFSET`对应于“dtb地址”或“BOARD_DTB_OFFSET”

虽然这个选项仅在内核头版本2中需要。否则可以注释掉。

* `KERNEL_BOOTIMAGE_VERSION`与内核头版本相关。搭载Android 8及以下版本的设备是0，Android 9是1，Android 10是2，Android 11是2，GKI设备是3。

* 对于三星设备`DEVICE_VBMETA_IS_SAMSUNG`必须设置为1。

* `BUILD_CC`对于大多数搭载Android 9及以上版本的设备是clang，但如果您的内核使用`clang`构建失败，您可以尝试将值更改为`aarch64-linux-android-gcc-4.9`以使用gcc构建。

* 如果您的设备需要一个不在构建系统中包含的工具链，您可以手动下载工具链并将路径添加到`BUILD_PATH`。

* `DEB_BUILD_FOR`和`KERNEL_ARCH`应根据设备架构进行更改。

### 启用自动启动分区闪存

安装构建的包意味着内核及其模块已经就位——但用户仍应将其闪存到他们的启动分区。

如果您希望启用自动闪存（通过[flash-bootimage](https://github.com/droidian/flash-bootimage))，
可以通过将`FLASH_ENABLED`设置为1（这是默认值）来实现。

如果您的设备不支持A/B更新，请确保将`FLASH_IS_LEGACY_DEVICE`设置为1。

请注意，您需要指定一些设备信息，以便`flash-bootimage`在设备上运行时可以交叉检查：

* `FLASH_INFO_MANUFACTURER`：`ro.product.vendor.manufacturer`
Android属性的值。在运行中的Droidian系统上，您可以使用

	（设备）$ sudo android_getprop ro.product.vendor.manufacturer

获取它

* `FLASH_INFO_MODEL`：`ro.product.vendor.model`
Android属性的值。在运行中的Droidian系统上，您可以使用

	（设备）$ sudo android_getprop ro.product.vendor.model

获取它

* `FLASH_INFO_CPU`：`/proc/cpuinfo`中的相关信息。

如果`FLASH_INFO_MANUFACTURER`或`FLASH_INFO_MODEL`未定义（它们都需要用于与Android属性检查），`flash-bootimage`
将检查`FLASH_INFO_CPU`。

如果没有指定特定于设备的信息，内核升级将失败。

F(x)tec Pro1的一个示例：

```
FLASH_ENABLED = 1
FLASH_INFO_MANUFACTURER = Fxtec
FLASH_INFO_MODEL = QX1000

FLASH_INFO_CPU = Qualcomm Technologies, Inc MSM8998
```

### 控制文件

如果出现python2的包错误，您可以将`python2`的Build-Depends条目替换为`python-is-python3`。

您还可以将任何可能需要的额外包添加到Build-Depends。

### initramfs钩子

如果initramfs有任何问题（例如，unl0kr在加密启用时显示黑屏），可以向打包中添加脚本，这些脚本将被包含在ramdisk中。

[这个打包](https://github.com/droidian-devices/linux-android-fxtec-pro1x/blob/bookworm/debian/initramfs-overlay/scripts/halium-hooks) 可以作为一个参考。

以下添加了一个在启动时在initramfs中运行的函数。
任何有助于系统启动的命令都可以添加到脚本中。

### 构建规则

如果在内核编译阶段之后（在打包期间）构建失败，可以忽略错误。

构建序列可以通过`override_dh_sequence`更改（确保用您自己的序列替换序列）

例如

```
override_dh_dwz：
override_dh_strip：
override_dh_makeshlibs：
```

可以添加到`rules`的末尾。

建议稍后回来修复构建错误，而不是忽略它们，因为它们可能会在未来导致各种问题。

内核适配
-----------------

至少需要在您的defconfig中启用这些选项

```
CONFIG_DEVTMPFS=y
CONFIG_VT=y
CONFIG_NAMESPACES=y
CONFIG_MODULES=y
CONFIG_DEVPTS_MULTIPLE_INSTANCES=y
CONFIG_USB_CONFIGFS_RNDIS=y
CONFIG_USB_CONFIGFS_RMNET_BAM=y
CONFIG_USB_CONFIGFS_MASS_STORAGE=y
CONFIG_INIT_STACK_ALL_ZERO=y
CONFIG_ANDROID_PARANOID_NETWORK=n
CONFIG_ANDROID_BINDERFS=n
```

通常`CONFIG_NAMESPACES`会启用所有命名空间选项，如果没有，应该添加所有这些选项

```
CONFIG_SYSVIPC=y
CONFIG_PID_NS=y
CONFIG_IPC_NS=y
CONFIG_UTS_NS=y
```

在初始启动成功之后，应该为其他组件启用各种选项。

对于蓝牙，需要这些选项

```
CONFIG_BT=y
CONFIG_BT_HIDP=y
CONFIG_BT_RFCOMM=y
CONFIG_BT_RFCOMM_TTY
CONFIG_BT_BNEP=y
CONFIG_BT_BNEP_MC_FILTER=y
CONFIG_BT_BNEP_PROTO_FILTER=y
CONFIG_BT_HCIVHCI=y
```

对于Waydroid

```
CONFIG_SW_SYNC_USER=y
CONFIG_NET_CLS_CGROUP=y
CONFIG_CGROUP_NET_CLASSID=y
CONFIG_VETH=y
CONFIG_NETFILTER_XT_TARGET_CHECKSUM=y
CONFIG_ANDROID_BINDER_DEVICES="binder,hwbinder,vndbinder,anbox-binder,anbox-hwbinder,anbox-vndbinder"
```

对于Plymouth（启动动画）

```
# CONFIG_FB_SYS_FILLRECT未设置
# CONFIG_FB_SYS_COPYAREA未设置
# CONFIG_FB_SYS_IMAGEBLIT未设置
# CONFIG_FB_SYS_FOPS未设置
# CONFIG_FB_VIRTUAL未设置
```

为了便于调试，可以启用pstore在每次启动时获取日志

```
CONFIG_PSTORE=y
CONFIG_PSTORE_CONSOLE=y
CONFIG_PSTORE_RAM=y
CONFIG_PSTORE_RAM_ANNOTATION_APPEND=y
```

如果您启用了pstore，您可能会在恢复中从`/sys/fs/pstore`找到线索。

您可以使用menuconfig确保所有选项及其依赖项都已启用。

	（docker）# mkdir -p out/KERNEL_OBJ && make ARCH=arm64 O=out/KERNEL_OBJ/ your_defconfig && make ARCH=arm64 O=out/KERNEL_OBJ/ menuconfig

修改defconfig后，将`out/KERNEL_OBJ/.config`复制到`arch/YOURARCH/configs/your_defconfig`。

或者，可以在`kernel-info.mk`中设置`KERNEL_CONFIG_USE_FRAGMENTS = 1`，在构建时将defconfig片段包含在您的defconfig中。

defconfig片段应放置在内核源代码根目录中名为droidian的目录中。[这个源代码](https://github.com/droidian-devices/linux-android-fxtec-pro1x/tree/bookworm/droidian) 可以作为一个参考。

可以启用`KERNEL_CONFIG_USE_DIFFCONFIG`以使用python脚本`diffconfig`比较片段和主要defconfig。

要使用diffconfig，应该提供diff文件，如下所示

`KERNEL_PRODUCT_DIFFCONFIG = diff_file`

如果LXC启动失败，您可以在设备首次启动后使用`lxc-checkconfig`检查可能需要的其他选项。

编译
---------

现在`kernel-info.mk`已经被修改，剩下的就是实际编译内核了。

首先，（重新）创建`debian/control`文件：

	（docker）# rm -f debian/control
	（docker）# debian/rules debian/control
	
现在一切都已就绪，您可以使用`releng-build-package`开始构建：

	（docker）# RELENG_HOST_ARCH="arm64" releng-build-package
	
当进行交叉编译时，需要`RELENG_HOST_ARCH`变量。

如果一切顺利，您将在`$PACKAGES_DIR`中找到结果包。

DTB/DTBO编译失败
----------------------------

如果DTB/DTBO编译失败，尝试使用外部dtc编译器而不是内核树中预存在的编译器是个好主意。

要做到这一点，在构建控制文件后，将`device-tree-compiler`添加到Build-Depends，然后在`debian/rules`中的include行后添加

`BUILD_COMMAND := $(BUILD_COMMAND) DTC_EXT=/usr/bin/dtc`

获取启动镜像
------------------------

启动镜像包含在`linux-bootimage-VERSION-VENDOR-DEVICE`包中。

您可以通过使用`dpkg-deb`提取包来获取boot.img，
或者直接从编译产物中获取（`out/KERNEL_OBJ/boot.img`）。

内核镜像已经嵌入了Droidian initramfs。

确保保存所有您的`linux-*.deb`包，因为您将在本指南中进一步需要它们。

提交更改
------------------

当您对内核感到满意时，一定要将您的更改以及Debian打包提交到git仓库：

* debian/source/
* debian/control
* debian/rules
* debian/compat
* debian/kernel-info.mk

……然后推送您的`bookworm`分支，供其他人享受。
