# immortalwrt x86/64软路由物理安装

## 硬件资源准备

- x86 电脑一枚（最好多网口）内存 2G/硬盘 2G 只做路由够用了，如有其他需求则自己加配置
- U 盘一个 2G 以上即可
- 鼠标键盘屏幕
- 能联网的电脑一台（下载工具/固件用，已下载则忽略）

## 镜像下载

### 方式1下载

<https://firmware-selector.immortalwrt.org/>

### 方式2下载

[Index of (root) / releases / 24.10.2 / targets / x86 / 64 /](https://downloads.immortalwrt.org/releases/24.10.2/targets/x86/64/)

![ImmortalWrt 固件下载页面](/static/2025/2025-06-28-immortalwrt-x86-install_001.png)

截至发文的 ImmortalWrt 的最新版本为 24.10.2。

这里需要说明下，x86 总体有三个镜像类型，常用 2 个类型也就是 ext4 和 squashfs，这两个版本又分了 efi 版本和非 efi 版本。

- ext4 和 squashfs 区别是 squashfs 可以使用类似路由的重置还原功能，ext4 则没用（也可以用其他方法达到相似效果）。
- efi 和非 efi 这个主要是硬件支持情况，如果硬件支持建议选非 efi 的配置更容易。

本文以 squashfs 和非 efi 版为例，那就是下载 `generic-squashfs-combined.img.gz`。

## 制作PE系统

使用 U 盘制作一个 PE 系统，注意，U 盘的内容会被格式化，注意备份 U 盘数据。推荐使用 WEPE，具体制作步骤请自行搜索。

把上面准备好的 PE U 盘插入装软路由的 x86 设备，开机 bios 设置 U 盘启动。

进入 PE 后先打开 diskgenius 工具，将你的待写入盘所有分区删除然后保存（注意删除所有分区后不要新建分区）。

## 两种方式将 immortalwrt 镜像写入硬盘

### 方式1

下载写盘工具 [physdiskwrite](https://m0n0.ch/wall/physdiskwrite.php)

然后打开 cmd 窗口【开始→cmd】

```batch
X:\physdiskwrite.exe -u X:\generic-squashfs-combined.img.gz
```

`X:` 是你 U 盘的盘符，根据启动后的盘符修改（另外可以打开 cmd 窗口后直接鼠标拖文件到 cmd 窗口就自动出现路径）。

上面命令回车后会让你选择设备，注意选择你软路由的硬盘，比如 `0` 然后回车，回车后会提示写入会导致之前的数据丢失，输入 `yes` 即可开始写入。

### 方式2

使用 [Roadkil's Disk Image](https://roadkil.net/program.php?ProgramID=12&Action=NewOSID&DownloadVersion=12&Installer=NO) 工具直接写入硬盘，无需命令行。

写入完成后重启，bios 设置开机启动项为硬盘，并拔掉 U 盘。启动好以后即可享用 ImmortalWrt。

## 提示

- 默认 Lan 地址 `192.168.1.1`
- 默认账号：`root`
- 默认密码：空
- 多个网卡默认 eth0 为 LAN 口，eth1 为 WLAN 口，其他口默认没启用

以上信息可通过编辑 `/etc/config/network` 配置文件修改。
