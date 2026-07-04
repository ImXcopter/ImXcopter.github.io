# 一键 DD Windows 脚本

精简版 DD Windows 重装脚本，适合需要快速把 VPS 重装为常用 Windows 镜像的场景。

本脚本基于 `fcurrk/reinstall` 的 `NewReinstall.sh` 改造，只保留常用 Windows 镜像、自定义镜像、网络检测、DHCP/手动 IP、镜像目录前缀校验和 DD 安装流程；已去除 CN 模式和不使用的系统菜单。

## 安装前提组件

建议先进入 `screen`，避免 SSH 断开导致操作中断。

### Debian / Ubuntu

```bash
apt-get install -y xz-utils openssl gawk file wget screen && screen -S os
```

### RedHat / CentOS

```bash
yum install -y xz openssl gawk file glibc-common wget screen && screen -S os
```

### 异常处理

如果依赖安装或下载出现异常，可以先刷新软件源缓存。

**RedHat / CentOS:**

```bash
yum makecache && yum update -y
```

**Debian / Ubuntu:**

```bash
apt update -y && apt dist-upgrade -y
```

## 使用方法

```bash
wget -O DDinstall.sh https://raw.githubusercontent.com/ImXcopter/DDinstall/main/DDinstall.sh && chmod +x DDinstall.sh && bash DDinstall.sh
```

如果 VPS 的 CA 证书环境过旧，可以使用：

```bash
wget --no-check-certificate -O DDinstall.sh https://raw.githubusercontent.com/ImXcopter/DDinstall/main/DDinstall.sh && chmod +x DDinstall.sh && bash DDinstall.sh
```

![](/static/2023/2023-04-27-windows-dd-script_001.png)

脚本启动后会先询问是否使用 DHCP 自动配置网络。

输入 `Y` 表示重装后的系统使用 DHCP 自动获取 IP。输入 `N` 表示使用静态网络配置。脚本会尝试自动检测当前 VPS 的 IP、网关和掩码，并要求确认；如果检测不正确，可以手动输入正确的网络信息。

DD 重装后能否联网，很大程度取决于这里的网络配置是否正确。

## 镜像目录前缀

为了避免把真实镜像目录公开在 GitHub，脚本里只保留镜像文件名，不写死完整下载地址。

运行脚本后会要求输入镜像文件所在目录：

```text
Image file URL directory prefix (example: https://xxx.com/private-dd/):
```

你需要输入包含目录路径的 URL，例如：

```text
https://example.com/private-dd/
```

脚本会自动拼接文件名：

```text
https://example.com/private-dd/Disk_Windows_Server_2022_DataCenter_CN_v2.12.vhd.gz
https://example.com/private-dd/zh-cn_windows10_ltsc.xz
https://example.com/private-dd/guajibao-winsrv2022-data-x64-cn-efi.vhd.gz
```

规则：

- 必须以 `http://` 或 `https://` 开头
- 必须包含目录路径，不能只填 `https://example.com`
- 末尾没有 `/` 会自动补齐
- 脚本会检查 9 个内置镜像文件是否都存在
- 任意文件无法访问，会提示缺失文件并要求重新填写

## 系统选择

![](/static/2023/2023-04-27-windows-dd-script_002.png)

当前脚本菜单已精简为以下选项：

```text
1  Windows Server 2022 [X64-Legacy-cxthhhhh]
2  Windows Server 2022 [X64-UEFI-cxthhhhh]
3  Windows 10 LTSC [X64-Legacy-teddysun]
4  Windows 10 LTSC [X64-UEFI-teddysun]
5  Windows 10 LTSC Lite [X64-Legacy-nat.ee]
6  Windows 10 LTSC Lite [X64-Legacy-aliyun-nat.ee]
7  Windows 10 LTSC Lite [X64-UEFI-nat.ee]
8  Windows Server 2022 Lite [X64-Legacy-nat.ee]
9  Windows Server 2022 Lite [X64-UEFI-nat.ee]
99 Custom image
0  Exit
```

选择系统后，脚本会显示对应默认密码，确认后开始调用 `InstallNET.sh` 执行 DD。

## 系统密码列表

| 序号 | 系统名称 | 默认密码 |
|------|----------|----------|
| 1 | Windows Server 2022 | `cxthhhhh.com` |
| 2 | Windows Server 2022 UEFI | `cxthhhhh.com` |
| 3 | Windows 10 LTSC | `Teddysun.com` |
| 4 | Windows 10 LTSC UEFI | `Teddysun.com` |
| 5 | Windows 10 LTSC Lite | `nat.ee` |
| 6 | Windows 10 LTSC Lite 阿里云专用 | `nat.ee` |
| 7 | Windows 10 LTSC Lite UEFI | `nat.ee` |
| 8 | Windows Server 2022 Lite | `nat.ee` |
| 9 | Windows Server 2022 Lite UEFI | `nat.ee` |
| 99 | 自定义镜像 | 由镜像本身决定 |

## 需要准备的文件名

你的镜像目录下需要放置这些文件：

```text
Disk_Windows_Server_2022_DataCenter_CN_v2.12.vhd.gz
Disk_Windows_Server_2022_DataCenter_CN_v2.12_UEFI.vhd.gz
zh-cn_windows10_ltsc.xz
zh-cn_win10_ltsc_uefi.xz
guajibao-win10-ent-ltsc-2021-x64-cn.vhd.gz
guajibao-win10-ent-ltsc-2021-x64-cn-aliyun.vhd.gz
guajibao-win10-ent-ltsc-2021-x64-cn-efi.vhd.gz
guajibao-winsrv2022-data-x64-cn.vhd.gz
guajibao-winsrv2022-data-x64-cn-efi.vhd.gz
```

## 自定义镜像

选择 `99 Custom image` 时，可以直接输入完整镜像 URL，不使用上面的固定文件名列表。

适合临时测试其他 DD 镜像，或者不想使用内置 9 个文件名的情况。

## 注意事项

1. DD 重装会覆盖当前系统磁盘，运行前请备份重要数据。
2. 建议确认 VPS 控制台可用，避免网络配置错误后无法恢复。
3. 系统名称后带 Lite 的均为精简版，没有的是完整版。
4. UEFI 机器请选择带 `UEFI` 的镜像。
5. 阿里云机器如需 Windows 10 LTSC Lite，优先选择阿里云专用版。
6. `cxthhhhh`、`teddysun`、`nat.ee` 为镜像来源代称。
7. 系统默认密码会在选择相应序号后提示，请注意记录。
8. 部分云厂商网络配置特殊，如果自动检测到的子网掩码明显不正确，需要手动修正。
9. 脚本和镜像来源请自行审查，公开脚本不保证第三方镜像安全性和版权状态。

## 报错处理

如果出现类似 `Error! grub.cfg` 的问题，可以尝试：

```bash
mkdir /boot/grub2 && grub-mkconfig -o /boot/grub2/grub.cfg
```

然后重新执行脚本。

## Hetzner 机器 DD Windows

1. 进入救援（Rescue）模式。
2. 根据系统提示用户名和密码，进入救援系统。用 root 用户 SSH 登录后执行：

```bash
wget -O- DD download URL | gunzip | dd of=/dev/sda
```

注意：`DD download URL` 为 DD 镜像下载链接，`/dev/sda` 为第一启动盘。

3. 等待程序跑完后输入 `reboot` 重启服务器。
4. 重启后即可尝试远程桌面连接。

如需激活 Windows，可使用沧水的 KMS 服务：https://kms.cangshui.net/

## 特别感谢

感谢 MoeClub、Vicer、cxt、hiCasper、Minijer 等前辈脚本和思路。本脚本只是按个人使用场景做了精简、重排和交互优化。

**版权申明：** 以上脚本和系统镜像均来源于网络整理，请自行判断安全性和版权状态。如有侵犯权益，请联系处理。
