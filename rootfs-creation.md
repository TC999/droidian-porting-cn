Rootfs 创建
===============

为了使安装过程更简单、更一致，我们构建了特定于设备的rootfs镜像。

目录
-----------------

* [摘要](#摘要)
* [先决条件](#先决条件)
* [包创建](#包创建)
* [构建rootfs](#构建rootfs)
* [自动化夜间镜像](#自动化夜间镜像)

摘要
-------

调试完成后，您应该开始为您的设备构建一个特定的rootfs，以使安装过程更加一致和用户友好。

这个过程涉及创建一个适配包和一个apt仓库。

适配包是一个debian包，包含所有设备特定的配置和所需的自定义二进制文件和脚本，以便设备能够正确运行。

为了能够进行OTA更新，您必须在像GitHub这样的平台或服务器上设置一个apt仓库。

apt仓库的创建不在本教程的范围内。

先决条件
-------------

* 设备特定的文件
* Docker

依赖项
------------

需要在您的主机上安装dpkg-dev gpg git和apt-utils
```
（主机）# apt update && apt install dpkg-dev gpg apt-utils
```

包创建
----------------

首先克隆droidian构建工具仓库

	（主机）$ git clone https://github.com/droidian-releng/droidian-build-tools/  && cd droidian-build-tools/bin

现在为您的设备创建一个模板。当然，替换vendor, codename, ARCH和APIVER

APIVER可以是28、29或30，取决于您作为基础使用的Android版本

ARCH可以是arm64、armhf或amd64

	（主机）$ ./droidian-new-device -v vendor -n codename -c ARCH -a APIVER -r phone -d droidian

由于您将为具有不同架构的设备构建，您必须初始化qemu-user-static以启用docker的仿真

	（主机）$ docker run --rm --privileged multiarch/qemu-user-static --reset -p yes

进入新创建的适配目录

	（主机）$ cd droidian/vendor/codename/packages/adaptation-vendor-codename/

现在将所有设备特定的文件放在`sparse/`下，并具有linux目录结构

确保将您的仓库URL放在`~/droidian-build-tools/bin/droidian/vendor/codename/packages/adaptation-vendor-codename/sparse/usr/lib/adaptation-vendor-model/sources.list.d/community-vendor-device.list`中，格式如下

`deb [signed-by=/usr/share/keyrings/codename.gpg] https://YOURREPO.TLD  bookworm main`

然后将您的gpg文件复制到`~/droidian-build-tools/bin/droidian/vendor/codename/packages/adaptation-vendor-codename/sparse/usr/share/keyrings/codename.gpg`。确保替换codename。

复制完所有文件后构建deb包

	（主机）$ ~/droidian-build-tools/bin/droidian-build-package

确保将`~/droidian-build-tools/bin/droidian/vendor/codename/droidian/apt`中的所有文件复制到您在`~/droidian-build-tools/bin/droidian/vendor/codename/packages/adaptation-vendor-codename/sparse/usr/lib/adaptation-vendor-model/sources.list.d/community-vendor-device.list`中指定的仓库

现在此时您可以在droidian设备上使用`package-sideload-create`来创建这两个deb包的恢复可刷入包。

将deb文件复制到您的个人仓库后，通过将sources list文件添加到您的设备来将该仓库添加到您的设备，然后执行

	（设备）# apt update
	（设备）# package-sideload-create adaptation-droidian-codename.zip adaptation-vendor-codename adaptation-vendor-codename-configs

最后，您应该得到一个zip文件，可以从恢复中刷入，并包含您所有的更改。

同时确保将您的仓库/更改提交到git仓库。

构建rootfs
-------------------

在构建rootfs之前，请确保将您在内核编译过程中构建的`linux-*.deb`包添加到`~/droidian-build-tools/droidian/vendor/codename/packages/adaptation-vendor-codename/droidian/community_devices.yml`作为一个包条目。

或者，您可以尝试将这些包作为依赖项添加到`~/droidian-build-tools/droidian/vendor/codename/packages/adaptation-vendor-codename/debian/control`中的适配包。

首先拉取rootfs-builder docker镜像

	（主机）$ docker pull quay.io/droidian/rootfs-builder:bookworm-amd64

现在在docker容器rootfs-builder中运行debos以构建rootfs。确保替换占位符值

	（主机）$ cd ~/droidian-build-tools/droidian/vendor/codename/droidian
	（主机）$ mkdir images
	（主机）$ docker run --privileged -v $PWD/images:/buildd/out -v /dev:/host-dev -v /sys/fs/cgroup:/sys/fs/cgroup -v $PWD:/buildd/sources --security-opt seccomp:unconfined quay.io/droidian/rootfs-builder:bookworm-amd64 /bin/sh -c 'cd /buildd/sources; DROIDIAN_VERSION="nightly" ./generate_device_recipe.py vendor_codename ARCH phosh phone APIVER && debos --disable-fakemachine generated/droidian.yaml'

如果一切构建正常，您应该在`~/droidian-build-tools/droidian/vendor/codename/packages/adaptation-vendor-codename/droidian/images/`中拥有您的LVM fastboot可刷入rootfs镜像。

自动化夜间镜像
-------------------------

您可以使用GitHub actions自动化构建，每天为您的设备生成带有您更改的新镜像。参考[这个actions yml文件](https://github.com/droidian-onclite/droidian-images/blob/bookworm/.github/workflows/release.yml)作为示例。

您可以通过替换`community_devices.yml`中的值和`apt/`中的文件来复制夜间构建。
