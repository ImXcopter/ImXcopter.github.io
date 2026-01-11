# 一键 DD Windows 脚本

一键 DD 脚本，支持性好，更智能更全面，支持国内外各种 VPS 重装，特别是对国内各种访问国外资源慢的 VPS 安装有奇效。

## 更新说明

| 日期 | 更新内容 |
|------|----------|
| 20230313 | 新增强制 CN模式，Using CN Mode选择Y将默认为国内主机安装，国外主机请选择N |
| 20220724 | 更新及增加大量系统镜像，41合1脚本，请注意看说明 |
| 20220703 | 新增自定义SSH端口（9-16） |
| 20220613 | 新版自定义密码支持特殊字符.!$@#&% |
| 20220428 | 修复MoeClub新版DD过程中卡住的BUG，修复Centos7下出现 Error! Not Found grub. 的错误提示，新增支持 xz 压缩格式的 DD 系统镜像包 |
| 20220406 | CN系统镜像已失效，国内主机使用一键脚本1-25选项需要较长时间，推荐使用99的自定义系统镜像 |
| 20211120 | 更新MoeClub新版,依赖更少,支持原版自定义密码安装,体验版可能有Bug |
| 20210909 | 支持debian11 |
| 20210511 | 发现很多人不知道怎么DD甲骨文,使用支持uefi的镜像包即可,ARM机器选择带ARM镜像 |
| 20210509 | 更新部分windows镜像，修正一处小问题 |
| 20210127 | 更换部分windows镜像 |
| 20210109 | 更新支持Ubuntu20.04安装，更新几个windows镜像 |
| 20200708 | 更新自动为CN主机使用国内镜像源 |

## 安装前提组件

### Debian/Ubuntu

```bash
apt-get install -y xz-utils openssl gawk file wget screen && screen -S os
```

### RedHat/CentOS

```bash
yum install -y xz openssl gawk file glibc-common wget screen && screen -S os
```

### 异常处理

如果出现异常，请刷新 Mirrors 缓存或更换镜像源。

**RedHat/CentOS:**

```bash
yum makecache && yum update -y
```

**Debian/Ubuntu:**

```bash
apt update -y && apt dist-upgrade -y
```

## 使用方法

```bash
wget --no-check-certificate -O NewReinstall.sh https://raw.githubusercontent.com/fcurrk/reinstall/master/NewReinstall.sh && chmod a+x NewReinstall.sh && bash NewReinstall.sh
```

如为 CN 主机（部分主机商已不能使用），可能出现报错或不能下载脚本的问题，可执行以下命令开始安装：

```bash
wget --no-check-certificate -O NewReinstall.sh https://cdn.jsdelivr.net/gh/fcurrk/reinstall@master/NewReinstall.sh && chmod a+x NewReinstall.sh && bash NewReinstall.sh
```

###本站镜像

```bash
wget --no-check-certificate -O NewReinstall.sh https://xcopter.cc/down/NewReinstall.sh && chmod a+x NewReinstall.sh && bash NewReinstall.sh
```

![](/static/2023/2023-04-27-windows-dd-script_001.png)

输入 Y 确认 DD 后主机自动获取 IP，N 则自行设置 IP。输入 N 后会自动检测出主机现用 IP，如果正确可以按 Y 确认使用，如不正确则按 N 自行按正确的输入。

![](/static/2023/2023-04-27-windows-dd-script_002.png)

41 合一的系统一键 DD 选择界面，输入 99 则使用自定义镜像。以上系统密码不为默认密码的均为网络收集，如有疑虑使用自己的自定义镜像。

## 41 合 1 系统密码列表

| 序号 | 系统名称 | 默认密码 |
|------|----------|----------|
| 1 | CentOS 7.7 (已关闭防火墙及SELinux) | Pwd@CentOS |
| 2 | CentOS 7 | cxthhhhh.com |
| 3 | CentOS 7 (支持ARM64、UEFI) | cxthhhhh.com |
| 4 | CentOS 8 | cxthhhhh.com |
| 5 | Rocky 8 | cxthhhhh.com |
| 6 | Rocky 8 (支持UEFI) | cxthhhhh.com |
| 7 | Rocky 8 (支持ARM64、UEFI) | cxthhhhh.com |
| 8 | CentOS 9 | cxthhhhh.com |
| 9 | CentOS 6 (官方源原版) | Minijer.com |
| 10 | Debian 11 (官方源原版) | Minijer.com |
| 11 | Debian 10 (官方源原版) | Minijer.com |
| 12 | Debian 9 (官方源原版) | Minijer.com |
| 13 | Debian 8 (官方源原版) | Minijer.com |
| 14 | Ubuntu 20.04 (官方源原版) | Minijer.com |
| 15 | Ubuntu 18.04 (官方源原版) | Minijer.com |
| 16 | Ubuntu 16.04 (官方源原版) | Minijer.com |
| 17 | Windows Server 2022 | cxthhhhh.com |
| 18 | Windows Server 2022 (支持UEFI) | cxthhhhh.com |
| 19 | Windows Server 2019 | cxthhhhh.com |
| 20 | Windows Server 2016 | cxthhhhh.com |
| 21 | Windows Server 2012 | cxthhhhh.com |
| 22 | Windows Server 2008 | cxthhhhh.com |
| 23 | Windows Server 2003 | cxthhhhh.com |
| 24 | Windows 10 LTSC | Teddysun.com |
| 25 | Windows 10 LTSC (支持UEFI) | Teddysun.com |
| 26 | Windows 7 x86 Lite | nat.ee |
| 27 | Windows 7 x86 Lite (阿里云专用) | nat.ee |
| 28 | Windows 7 x64 Lite | nat.ee |
| 29 | Windows 7 x64 Lite (支持UEFI) | nat.ee |
| 30 | Windows 10 LTSC Lite | nat.ee |
| 31 | Windows 10 LTSC Lite (阿里云专用) | nat.ee |
| 32 | Windows 10 LTSC Lite (支持UEFI) | nat.ee |
| 33 | Windows Server 2003 Lite (C盘默认10G) | WinSrv2003x86-Chinese |
| 34 | Windows Server 2008 Lite | nat.ee |
| 35 | Windows Server 2008 Lite (支持UEFI) | nat.ee |
| 36 | Windows Server 2012 Lite | nat.ee |
| 37 | Windows Server 2012 Lite (支持UEFI) | nat.ee |
| 38 | Windows Server 2016 Lite | nat.ee |
| 39 | Windows Server 2016 Lite (支持UEFI) | nat.ee |
| 40 | Windows Server 2022 Lite | nat.ee |
| 41 | Windows Server 2022 Lite (支持UEFI) | nat.ee |
| 99 | 自定义镜像 | - |

## 注意事项

1. 系统名称后带 Lite 的均为精简版，没有的是完整版。
2. [X64-Legacy-cxthhhhh] 代表系统为 AMD64 位，支持传统 BIOS 启动，cxthhhhh 定制的系统镜像。
3. ARM64 代表系统支持 ARM64 位。
4. UEFI 代表系统支持最新的 UEFI 启动，如甲骨文全部都是这种。
5. aliyun 代表阿里云专用系统镜像。
6. cxthhhhh、teddysun、nat.ee 均为三位制作系统镜像的大佬代称。
7. 系统密码会在选择相应序号后提示，请注意记录。
8. 经测试在谷歌云原版系统基础上 DD 会出现自动获取的子网掩码为 255.255.255.255，如出现这种情况需要手工输入改正为正确的如 255.255.255.0，否则会安装完成主机可能会离线。
9. 阿里云因使用了特殊的驱动，DD 安装 Windows 系统选择阿里云专用版。
10. netcup 的 vps 请选择镜像编号 40。
11. Oracle Cloud（甲骨文云）可选择支持 UEFI 的镜像，注意基础系统最好选择 Ubuntu，如原系统是 CentOS 可能无法成功，注意如是 ARM 机器注意选择同时支持 ARM64 和 UEFI 的镜像。
12. 9-16 项安装原版系统，可自定义密码，密码要求 8-16 位，以英文字母或数字开头，可以是大小写英文字母、数字及 7 个特殊字符.!$@#&% 的任意组合。

## 报错处理

报错 Error! grub.cfg. 解决办法：

```bash
mkdir /boot/grub2 && grub-mkconfig -o /boot/grub2/grub.cfg
```

## Hetzner 机器 DD Windows

1. 进入救援（Rescue）模式。
2. 根据系统提示用户名和密码，进入救援系统。用 root 用户 ssh 进去后执行以下的命令：

```bash
wget -O- DD download URL | gunzip | dd of=/dev/sda
```

注意：DD download URL（为 DD 镜像的下载链接）/dev/sda 为第一启动盘

3. 等待程序跑完...
4. 输入 reboot 命令重启服务器。
5. 重启后，就可以启用远程桌面了。

如需激活 Windows，可使用沧水的 KMS 服务：https://kms.cangshui.net/

## 特别感谢

特别感谢：Vicer、cxt、hiCasper 等各位技术大佬的脚本，站长只是脚本的"搬运工"。

**版权申明：** 以上所有脚本、系统均为网络收集，站长不对资源的安全及版权纠纷负责，如有侵犯您的权益欢迎联系。

转载自：https://git.beta.gs/
