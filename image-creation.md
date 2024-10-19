图像创建
==============

设备发布
------------

在treble/GSI迁移之前的设备需要构建一个系统镜像，因为默认镜像不会为系统提供所需的库。

目录
-----------------

* [摘要](#摘要)
* [先决条件](#先决条件)
* [构建环境](#构建环境)
* [创建清单文件](#创建清单文件)
* [初始化构建环境](#初始化构建环境)
* [设备树修改](#设备树修改)
* [编译](#编译)
* [获取镜像](#获取镜像)
* [测试系统镜像](#测试系统镜像)
* [打包系统镜像](#打包系统镜像)

摘要
-------

基于Halium的操作系统使用Android驱动程序来利用系统的硬件。这样做可以使移植过程更快更顺利。

它使用一个在后台运行的非常精简版的Android容器。然后使用中间件和各种系统软件的补丁来利用这些加载的驱动程序。

在GSI系统上，可以使用使用供应商分区进行系统固件的系统镜像。但在没有供应商分区的旧设备上（非Treble），必须从头开始构建系统镜像。

目前，这种方法仅适用于Android 9（halium-9.0）的设备。例如，一个发布时带有Android 7但已将Android 9移植到设备的设备。

先决条件
-------------

* 设备内核源代码
* 设备树源代码
* 设备供应商树源代码（如果适用）

构建环境
-----------------

在开始之前，应该指出Android构建系统在构建时将超过50GB，并且在没有任何修改的情况下启动时约为35GB。因此，在开始过程之前，请确保您有足够的存储空间。

首先为Halium/Android构建系统创建一个目录

	（主机）$ mkdir droidian && cd droidian

并为您的Halium版本初始化正确的清单文件（在这种情况下，它将是halium-9.0）

	（主机）$ repo init -u https://github.com/Halium/android -b halium-9.0 --depth=1

现在您已经准备好克隆所有仓库并下载所需仓库的源代码。（这将需要一些时间）

	（主机）$ repo sync -c

如果您的连接速度很快，您可以使用-j增加工作数量

	（主机）$ repo sync -c -j 16

创建清单文件
------------------

下载完所有源代码后，您需要为您的设备编写一个清单文件。
清单文件是一个存储您所有源代码在互联网上的位置以及您本地磁盘上所有源代码位置的文件。

清单文件存储在halium/devices/manifests中，文件名的格式为`vendor_device.xml`，其中device是您设备的代号。
例如，对于小米Redmi Note 7，它将是`xiaomi_lavender.xml`。

您的清单文件应该有这个作为起始代码，并且应该附加所有内容

```
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
    
</manifest>
```

### 远程仓库

可以创建远程仓库以标记每个项目应该从哪个git实例克隆。例如

`<remote name="github" fetch="https://github.com/"  revision="halium-9.0" />`

这意味着每次使用github作为远程仓库时，将在github.com中搜索仓库，并且分支始终是halium-9.0（revision可以从远程中移除并移动到项目条目）

远程的唯一条件是fetch值必须是git实例。

## 项目

项目条目指示要克隆的仓库。例如

`<project path="device/lge/hammerhead" name="droidian-hammerhead/android_device_lge_hammerhead" remote="github" />`

这行表示构建脚本应该查看远程github（即https://github.com/）并克隆到仓库`droidian-hammerhead/android_device_lge_hammerhead`（https://github.com/droidian-hammerhead/android_device_lge_hammerhead）并将其存储在构建系统的device/lge/hammerhead根目录下。

因为我们有revision="halium-9.0"，它还会确保克隆到halium-9.0分支。
revision也可以添加到项目行中而不是远程，如果并非所有仓库都有相同的分支名称。

`<project path="device/lge/hammerhead" name="droidian-hammerhead/android_device_lge_hammerhead" remote="github" revision="halium-9.0" />`

在添加了您设备的所有条目（内核、设备、供应商，可能还有其他条目，如适用于三星设备的slsi）后，您可以进入下一步。

### 移除项目

remove-project条目用于移除构建系统中的目录或文件。例如

`<remove-project name="hybris-patches" />`

这行移除了hybris-patches目录。然后可以被另一个项目替换

`<project path="hybris-patches" name="Halium/hybris-patches" revision="halium-9.0-arm32" />`

在这个例子中，它被替换为适用于具有32位CPU的设备的arm32版本的hybris补丁，因为默认的hybris-patches是arm64。

初始化构建环境
----------------------------------

完成清单文件后，您可以使用设置脚本将您的设备设置为构建目标。

确保在您的droidian目录根目录下运行这些命令

	（主机）$ halium/devices/setup CODENAME

将CODENAME替换为您的代号

然后应用构建系统的hybris补丁

	（主机）$ hybris-patches/apply-patches.sh --mb

现在我们要做的就是设置环境变量以拥有一个准备就绪的构建环境

	（主机）$ source build/envsetup.sh
	（主机）$ breakfast CODENAME

将CODENAME替换为您的代号

设备树修改
-------------------------

对于Halium 9，我们需要将系统镜像构建为system-as-root

`BOARD_BUILD_SYSTEM_ROOT_IMAGE := true`

由于这个更改使android根变为只读，我们可能需要为一些重要的分区提供挂载点，如固件、持久性与系统镜像一起，为此我们可以使用这行

```
BOARD_ROOT_EXTRA_FOLDERS := \
 /firmware \
 /dsp \
 /persist
```

您可能需要根据您的设备添加更多的挂载点。成功启动后执行`ls -la /`并添加对应于损坏的符号链接的文件夹。

您还可能需要注释掉java（jar）库和可执行文件的构建，因为我们的环境中不需要这些。

编译
---------

要构建系统镜像

	（主机）$ mka e2fsdroid
	（主机）$ mka systemimage

获取镜像
-------------------

如果一切顺利，没有任何错误，您应该能够在`out/target/product/CODENAME`中找到所有的镜像，其中CODENAME是您的代号。
您应该看到`system.img`。

我们不会在这里使用创建的`halium-boot.img`，但应该使用内核编译指南来构建适当的启动镜像。

测试系统镜像
----------------------------

要在Droidian中测试此镜像，首先需要将其从稀疏镜像转换为原始镜像

	（主机）$ simg2img system.img system-raw.img

现在它可以安全地移动到我们的Droidian安装中。

假设设备已经启动到Droidian，可以使用scp移动镜像。

	（主机）$ scp system-raw.img droidian@10.15.19.82:~/

现在ssh到您的设备

	（主机）$ ssh droidian@10.15.19.82

并将镜像移动到`/var/lib/lxc/android/`

	（设备）$ sudo cp system-raw.img /var/lib/lxc/android/android-rootfs.img

如果设备尚未启动，可以从恢复模式修改rootfs，通过将其挂载（`/data/rootfs.img`）并复制镜像

	（主机）$ adb push system-raw.img /data/

现在从恢复模式shell

	（恢复）$ mkdir /tmp/mpoint
	（恢复）$ mount /data/rootfs.img /tmp/mpoint
	（恢复）$ cp /data/system-raw.img /tmp/mpoint/var/lib/lxc/android/android-rootfs.img

在系统镜像经过测试并确认工作后，您可以继续进行[创建rootfs](./rootfs-creation.md)

打包系统镜像
-----------------------------

您必须打包您的系统镜像以便在构建rootfs时稍后使用[Rootfs创建](./rootfs-creation)

这里的主要注意事项是，在构建带有系统镜像的debian软件包时，它应该有`android-system-gsi-28`作为冲突软件包。

您还应确保在安装过程中将系统镜像移动到`/var/lib/lxc/android/android-rootfs.img`。
