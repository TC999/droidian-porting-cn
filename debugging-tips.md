调试技巧
==============

一套可能有助于您调试Droidian移植问题的建议和修复方法。

目录
-----------------

* [摘要](#摘要)
* [启动前提示](#启动前提示)
* [当系统带有shell启动时的提示](#当系统带有shell启动时的提示)
* [当系统带有Phosh启动时的提示](#当系统带有Phosh启动时的提示)

摘要
-------

在移植新设备时，您可能会（确实会）遇到一些问题。
不太可能在首次启动时一切正常，因此请做好准备，
仔细阅读这份文档两次，享受您的旅程。

标有**（仅限通用rootfs）**的提示仅在通过恢复模式（注意本文档假设使用的是TWRP恢复）刷入通用rootfs.zip安装的Droidian安装中有用/适用。

启动前提示
-------------

首先值得检查这些事情：

### （仅限通用rootfs）检查Droidian rootfs的大小

一些恢复模式可能在调整Droidian rootfs大小时失败。这将反过来
由于缺乏可用的空闲空间，破坏启动过程（以及最终的功能包安装）。

您可以从恢复模式打开shell（要么通过`adb shell`要么使用内置终端）并检查：

	（恢复）$ ls -lth /data/rootfs.img

如果镜像大小不是8GB，您可以尝试手动调整大小：

	（恢复）$ e2fsck -fy /data/rootfs.img
	（恢复）$ resize2fs -f /data/rootfs.img 8G

### （仅限通用rootfs）安装devtools

稳定的Droidian版本需要安装devtools功能包
以获得有助于调试的实用工具，包括SSH服务器和工具要求
通过RNDIS暴露它。

Devtools还将强制systemd日志刷新到磁盘，以便最终
可以在恢复中查找日志。

### （仅限通用rootfs）尝试使用夜间发布版

夜间发布版在rootfs中嵌入了devtools。如果您在稳定版本上安装devtools有困难，考虑尝试使用夜间镜像，这样您就不需要单独安装它。

### 再次检查cmdline的`systempart=`选项

如果您重复使用Ubuntu Touch的相同启动镜像，您可能在内核cmdline中有
`systempart=`选项。这可能导致Android/Ubuntu Touch启动而不是Droidian

与Ubuntu Touch不同，Droidian不使用现有的系统分区作为
rootfs。因此，如果您有`systempart=`内核选项，请删除它并重新编译
您的内核（或者使用yabit或
Android Image Kitchen等工具从启动镜像中删除它）。

如果您的设备立即重新启动，您应该尝试这些几个systemd部分

### initramfs找不到userdata分区
如果设备停留在initramfs（您将在主机的dmesg中看到启动失败），一种可能性是initramfs无法找到userdata分区。

为了解决这个问题，可以在cmdline中添加`datapart=`。

datapart的值应该是`/dev/disk/by-partlabel/userdata`或确切的标签和编号，例如`/dev/sda23`（这取决于设备）。

### （仅限通用rootfs）屏蔽journald

一些设备对systemd-journald有麻烦。您可以尝试通过恢复模式屏蔽它。

请注意，屏蔽journald将禁用日志收集，因此其他问题将更难调试。

	（恢复）$ mkdir /tmp/mpoint
	（恢复）$ mount /data/rootfs.img /tmp/mpoint
	（恢复）$ chroot /tmp/mpoint /bin/bash
	（恢复）$ export PATH=/usr/bin:/usr/sbin
	（恢复）$ systemctl mask systemd-journald

### （仅限通用rootfs）检查systemd日志以寻找线索

如果您没有屏蔽journald，并且已安装devtools（或安装了夜间镜像），您可以检查systemd日志：

	（恢复）$ mkdir /tmp/mpoint
	（恢复）$ mount /data/rootfs.img /tmp/mpoint
	（恢复）$ chroot /tmp/mpoint /bin/bash
	（恢复）$ export PATH=/usr/bin:/usr/sbin
	（恢复）$ journalctl --no-pager

如果您停留在Droidian徽标上，并且RNDIS无法工作，您应该进入您的内核配置以确保`CONFIG_USB_CONFIGFS_RNDIS`已启用。

还应该注意，您应该检查[内核编译](./kernel-compilation.md)指南中的pstore部分。

如果您已安装devtools（或已刷入夜间镜像），但RNDIS仍然无法工作，这些提示可能会帮助您：

### （仅限通用rootfs）屏蔽resolved和timesyncd

一些内核（已知exynos内核有此问题）对systemd为一些
守护进程创建的内核命名空间有麻烦，如timesyncd和resolved。这可能会使
启动过程挂起。

您可以尝试通过恢复模式shell屏蔽这两个服务：

	（恢复）$ mkdir /tmp/mpoint
	（恢复）$ mount /data/rootfs.img /tmp/mpoint
	（恢复）$ chroot /tmp/mpoint /bin/bash
	（恢复）$ export PATH=/usr/bin:/usr/sbin
	（恢复）$ systemctl mask systemd-resolved
	（恢复）$ systemctl mask systemd-timesyncd

**如果这有效，请重新检查您的内核配置。参见上面的部分。**

### （仅限通用rootfs）禁用Halium容器

一些供应商脚本可能与用于设置RNDIS连接的usb-tethering脚本冲突。

您可以使用以下方法禁用Halium容器启动：

	（恢复）$ mkdir /tmp/mpoint
	（恢复）$ mount /data/rootfs.img /tmp/mpoint
	（恢复）$ chroot /tmp/mpoint /bin/bash
	（恢复）$ export PATH=/usr/bin:/usr/sbin
	（恢复）$ nano /etc/systemd/system/lxc@android.service

注释掉ExecStartPre和ExecStart行，添加

`ExecStart=/bin/true`

保存，同步并重新启动

### （仅限通用rootfs）通过WLAN设置连接

您可以尝试预配置您的WLAN设备，并尝试通过WLAN而不是RNDIS进入：

	（恢复）$ mkdir /tmp/mpoint
	（恢复）$ mount /data/rootfs.img /tmp/mpoint
	（恢复）$ chroot /tmp/mpoint /bin/bash
	（恢复）$ export PATH=/usr/bin:/usr/sbin
	（恢复）$ nano /etc/network/interfaces

并在其中放置以下内容：

```
auto wlan0
iface wlan0 inet dhcp
  wpa-ssid SSID
  wpa-psk PASSWORD
```

其中SSID是您的AP SSID，PASSWORD是您的密码。

您也可以根据需要使用静态配置：

```
auto wlan0
iface wlan0 inet static
  wpa-ssid SSID
  wpa-psk PASSWORD
  address <ip>
  netmask <netmask>
  gateway <gw>
```

重新启动，如果一切顺利，您可以尝试使用`ssh droidian@$IP`进入。

如果您使用DHCP配置，您应该检查您的DHCP服务器租赁。

默认密码是`1234`。

### ssh说系统仍在启动

为了使ssh忽略systemd抱怨，可以在恢复中添加[这个服务文件](https://github.com/droidian-mt6765/adaptation-droidian-garden/blob/main/debian/adaptation-garden-configs.ssh-fix.service) 

	（恢复）$ mkdir /tmp/mpoint
	（恢复）$ mount /data/rootfs.img /tmp/mpoint
	（恢复）$ chroot /tmp/mpoint /bin/bash
	（恢复）$ export PATH=/usr/bin:/usr/sbin
	（恢复）$ nano /etc/systemd/system

放入服务文件的内容

	（恢复）$ systemctl enable ssh-fix

当系统带有shell启动时的提示
---------------------------------------------

**在继续之前，请阅读前一节，这些提示可能也有用。**

**请注意，除非特别指明，否则本节中的每个命令都意味着需要root权限**

您可以使用以下命令启动root shell

	（设备）$ sudo -i

### USB网络

要与设备共享主机的网络连接，您可以在主机上使用以下命令

	（主机）$ sudo sysctl net.ipv4.ip_forward=1
	（主机）$ sudo iptables -t nat -A POSTROUTING -o $INTERNET -j MASQUERADE
	（主机）$ sudo iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
	（主机）$ sudo iptables -A FORWARD -i $USB -o $INTERNET -j ACCEPT

$USB是连接到您的设备的接口，很可能是`usb0`

$INTERNET是连接到互联网的接口。

然后在您的设备上运行以下命令

	（设备）$ sudo ip route add default via 10.15.19.100

IP地址`10.15.19.100`是您的主机。运行命令前请仔细检查。

现在要访问互联网，将`/etc/resolv.conf`的内容替换为`nameserver 8.8.8.8`。

### Umount schedtune

Schedtune必须被禁用才能使Droidian工作，但一些设备内核对此有抱怨。
因此，解决方案是在启动时umount schedtune：

	（设备）# mkdir -p /etc/systemd/system/android-mount.service.d
	（设备）# nano /etc/systemd/system/android-mount.service.d/10-schedtune.conf

在那里添加这个：

```
[Service]
ExecStartPre=-/usr/bin/umount -l /sys/fs/cgroup/schedtune
```

保存并重新启动

### 确保Halium容器正在运行

您可以使用

	（设备）# lxc-ls --fancy

来检查Halium容器是否正在运行

### 手动启动Halium容器

如果Halium容器没有运行，您可以尝试手动启动它：

	（设备）# lxc-start -n android --logfile=/tmp/lxclog --logpriority=DEBUG

然后检查`/tmp/lxclog`。

### 重新生成udev规则

如果容器正在运行，但您仍然没有启动到UI，您可以尝试
重新生成udev规则（摘自[UBports halium-9移植说明](https://github.com/ubports/porting-notes/wiki/Halium-9#generating-udev-rules)) 

	（设备）# DEVICE=codename # 替换为您的设备代号
	（设备）# cat /var/lib/lxc/android/rootfs/ueventd*.rc /vendor/ueventd*.rc | grep ^/dev | sed -e 's/^\/dev\///' | awk '{printf "ACTION==\"add\", KERNEL==\"%s\", OWNER=\"%s\", GROUP=\"%s\", MODE=\"%s\"\n",$1,$3,$4,$2}' | sed -e 's/\r//' >/etc/udev/rules.d/70-$DEVICE.rules

然后重新启动。

### 检查if test_hwcomposer工作

尝试`test_hwcomposer`命令可能是一个有用的测试，以检查Android composer是否工作。

请注意，在Halium 10/11设备上，使用客户端如test_hwcomposer或phoc本身后
可能需要重新启动Android composer服务。您可以终止`android.service.composer`进程，它将由
android init自动重新生成。

### 使Phosh服务在启动前等待几秒钟

有时Phosh可能会尝试启动，即使Android composer服务尚未准备好（即使它自己发出信号）。

这是一个错误，您可以通过使phosh服务（启动合成器的服务）等待几秒钟来解决这个问题：

	（设备）# mkdir -p /etc/systemd/system/phosh.service.d/
	（设备）# nano /etc/systemd/system/phosh.service.d/90-wait.conf

并在那里放置这个：

```
# FIXME
[Service]
ExecStartPre=/usr/bin/sleep 5
```

保存并重新启动。

### 供应商未挂载

可能供应商分区没有被init挂载，因此Halium容器无法启动许多服务。

首先检查供应商映像是否已挂载

	（设备）# ls /vendor

如果为空，则它没有被init挂载。尝试手动挂载它，通过在`/dev/disk/by-partlabel`和`/dev/block/bootdevice/by-name`中找到分区。

如果它不在那里，那么您需要在`/dev`中搜索您的供应商分区。

然后进行测试，在`/var/lib/lxc/android/pre-start.sh`下`# Halium 9`部分添加以下内容

````
mkdir -p /var/lib/lxc/android/rootfs/vendor
mount /dev/mmcblk0pYOURVENDOR /vendor
mount --bind /vendor /var/lib/lxc/android/rootfs/vendor
````

并重新启动

请务必稍后回来寻找适当的解决方案。

### lxc@android和phosh已启动但没有输出

如果phosh和lxc@android已在udev规则下启动，但屏幕上没有输出，可能是vndservicemanager崩溃了。

在Halium 11和一些Halium 10设备上，必须使用修补过的vndservicemanager版本。修补过的vndservicemanager版本可以在这里找到[here](https://github.com/droidian-starqlte/adaptation-droidian-starqlte/blob/bookworm/usr/lib/droid-vendor-overlay/bin/vndservicemanager)。

	（设备）# mkdir -p /usr/lib/droid-vendor-overlay/bin/
	（设备）# cp vndservicemanager /usr/lib/droid-vendor-overlay/bin/

重新启动

### 启动时间过长

如果您遇到启动时间过长，您可以尝试检查内核日志并查看哪个进程在阻止系统启动

	（设备）# dmesg

要弄清楚启动过程中哪一部分导致了问题

	（设备）# systemd-analyze

当系统带有Phosh启动时的提示
-----------------------------------------

### 屏幕亮度

在一些高通设备上，屏幕亮度在启动时总是设置为0。为了解决这个问题，亮度可以在每次启动时设置为最大

	（设备）$ echo 2047 > /sys/class/leds/lcd-backlight/brightness

也可以将其设置为服务以在启动时启动，如[这个服务文件](https://github.com/droidian-lavender/adaptation-droidian-lavender/blob/main/debian/adaptation-lavender-configs.brightness.service)。

在联发科设备上，电源按钮在使用一次后将停止工作。

[This hack](https://github.com/droidian-mt6765/garden_power_button_helper) 或类似hack现在可以使用。

它应该在设备本身上编译，或者可以从计算机上交叉编译。

### 从下拉菜单调整亮度

在某些设备上，供应商在启动时为亮度sysfs节点设置了错误的权限，因此Phosh无法访问和修改它。

可以使用[这个](https://github.com/droidian-onclite/adaptation-droidian-onclite/blob/main/debian/adaptation-onclite-configs.brightnessperm.service)服务来设置正确的权限或至少给用户droidian访问sysfs节点的权限

确保根据您的需要替换亮度节点。

### Phosh缩放

Phosh的缩放可能设置错误。应该创建phoc.ini来调整它

	（设备）# mkdir -p /etc/phosh/
	（设备）# nano phoc.ini

并放置[phoc.ini](https://github.com/droidian-lavender/adaptation-droidian-lavender/blob/main/etc/phosh/phoc.ini)

`output:HWCOMPOSER-1`的值应该调整。

### 供应商分区覆盖

要覆盖供应商分区上的文件，可以使用`droid-vendor-overlay`目录

	（设备）# mkdir -p /usr/lib/droid-vendor-overlay

您可以在这里添加您的文件。参考[这个](https://github.com/droidian-devices/adaptation-fxtec-pro1x/tree/bookworm/sparse/usr/lib/droid-vendor-overlay)。

需要注意的是目录结构很重要。

### 系统分区覆盖

要覆盖系统分区上的文件，可以使用`droid-system-overlay`目录

	（设备）# mkdir -p /usr/lib/droid-system-overlay

您可以在这里添加您的文件。参考[这个](https://github.com/droidian-devices/adaptation-fxtec-pro1x/tree/bookworm/sparse/usr/lib/droid-system-overlay)。

需要注意的是目录结构很重要。

### 蓝牙崩溃

蓝牙可能因缺少MAC地址而崩溃。为了使蓝牙服务忽略它，可以使用此hack

	（设备）# mkdir -p /var/lib/bluetooth/
	（设备）# touch /var/lib/bluetooth/board-address

对于适当的解决方案，应该创建一个`droid-get-bt-address.sh`脚本以获取正确的地址。

可以在[here](https://github.com/droidian-sargo/adaptation-droidian-sargo/blob/bookworm/usr/bin/droid/droid-get-bt-address.sh)找到高通的解决方案示例
和[here](https://github.com/droidian-mt6765/adaptation-droidian-garden/blob/main/usr/bin/droid/droid-get-bt-address.sh)找到联发科的解决方案示例。

### 蓝牙在设置中不可用

蓝牙可能因各种原因未能出现在`gnome-control-center`中。

可以使用`bluetoothctl`或`blueman`进行测试

	（设备）$ bluetoothctl
	[bluetooth]# scan on

或安装blueman

	（设备）# apt install blueman

### 音频调整不起作用

在联发科设备上，pulseaudio需要一个自定义配置文件，可以在[here](https://github.com/droidian-mt6765/adaptation-droidian-garden/blob/main/etc/pulse/arm_droid_card_custom.pa)找到。

### 自定义主机名

要在启动时设置自定义主机名，可以创建一个首选主机名文件

	（设备）# mkdir -p /usr/lib/droidian/device/
	（设备）# nano preferred-hostname

并放入您的设备型号或代号，不要有空格。

### 内核模块

`systemd-modules-load`在联发科上不加载模块。[This](https://github.com/droidian-mt6765/adaptation-droidian-garden/blob/main/debian/adaptation-garden-configs.modules.service)服务或类似的实现可以被包含以加载所有正确的模块并使Wi-Fi工作。

### 屏幕上的光标

一些设备有一些HID接口，这些接口不被Droidian使用。这些接口可能被注册为键盘或鼠标等。

因此，您可能会看到屏幕上莫名其妙地出现光标。

要获取有关输入设备的信息，可以安装并使用`libinput-tools`

	（设备）# apt install libinput-tools

列出所有设备

	（设备）# libinput list-devices

监控输入

	（设备）# libinput debug-events

要隐藏此事件节点，可以添加udev规则

`ACTION=="add|change", KERNEL=="event1", OWNER="root", GROUP="system", MODE="0666", ENV{LIBINPUT_IGNORE_DEVICE}="1"`

到`/etc/udev/rules.d/71-hide.rules`。

确保将此行中的事件节点从`event1`调整为您的输入设备并重新启动。

### 加密支持

要测试加密，首先在`/usr/lib/droidian/device/encryption-supported`中创建一个空文件，然后打开加密应用程序。

	（设备）# mkdir -p /usr/lib/droidian/device/
	（设备）# touch /usr/lib/droidian/device/encryption-supported

如果启用加密后设备可以无问题地解锁，则保留该文件。否则，将其删除。
