# BBR 一键安装脚本 BBR/魔改/暴力/BBRplus/锐速（zeruns 魔改版）

BBR 是 Google 提出的一种新型拥塞控制算法，可以使 Linux 服务器显著地提高吞吐量和减少 TCP 连接的延迟。下面是一个五合一的 TCP 网络加速脚本，其包括了 BBR 原版、BBR 魔改版、暴力 BBR 魔改版、BBR plus、Lotsever（锐速）安装脚本。该脚本由 94ish.me 制作。可用于 KVM/Xen 架构，不兼容 OpenVZ（OVZ）。支持 Centos 6+ / Debian 7+ / Ubuntu 14+，BBR 魔改版不支持 Debian 8。

## 安装教程

**1. 使用 SSH 客户端连接你的 VPS 服务器，运行以下脚本：**

```bash
wget -N --no-check-certificate "https://gist.github.com/zeruns/a0ec603f20d1b86de6a774a8ba27588f/raw/4f9957ae23f5efb2bb7c57a198ae2cffebfb1c56/tcp.sh" && chmod +x tcp.sh && ./tcp.sh
```

如果国内 VPS 无法连接，可使用本站镜像：

```bash
wget -N --no-check-certificate "https://xcopter.cc/src/tcp.sh" && chmod +x tcp.sh && ./tcp.sh
```

**2. 运行完成将出现以下菜单，可根据需要来安装相对应的核心，之后再打开加速功能。**

![菜单界面](/static/2023/2023-03-01-linux-bbr-script_001.png)

以安装 BBR plus 为例，输入数字 2 来安装。重启 VPS 后如图：

![重启后界面](/static/2023/2023-03-01-linux-bbr-script_002.png)

**3. 安装成功，重启 VPS 之后我们重新连接服务器。输入下列指令来启用其 BBR plus：**

```bash
./tcp.sh
```

**4. 按照脚本菜单选项，选择对应安装的功能，来启用加速。**

![功能选项](/static/2023/2023-03-01-linux-bbr-script_003.png)

**5. 如出现如图所示的信息，则表明 BBR 的加速功能已成功打开。**

![成功提示](/static/2023/2023-03-01-linux-bbr-script_004.png)

**6. 如果必须安装或是转换其他版本的加速，必须再次打开脚本来进行卸载。** 卸载完成之后再选择所需的版本进行安装，之后需再次打开脚本来进行功能选择。
