# BBR一键安装脚本 BBR/魔改/暴力/BBRplus/锐速（zeruns魔改版）

BBR是 Google 提出的一种新型拥塞控制算法,可以使 Linux 服务器显著地提高吞吐量和减少 TCP 连接的延迟。 下面是一个五合一的TCP网络加速脚本，其包括了BBR原版、BBR魔改版、暴力BBR魔改版、BBR plus、Lotsever(锐速)安装脚本。该脚本由94ish.me制作。可用于KVMXen架构，不兼容OpenVZ（OVZ）。支持Centos 6+ / Debian 7+ / Ubuntu 14+，BBR魔改版不支持Debian 8。
安装教程
1.使用SSH客户端连接你的VPS服务器，运行以下脚本。

```
wget -N --no-check-certificate "https://gist.github.com/zeruns/a0ec603f20d1b86de6a774a8ba27588f/raw/4f9957ae23f5efb2bb7c57a198ae2cffebfb1c56/tcp.sh" && chmod +x tcp.sh && ./tcp.sh
```

如果国内VPS无法连接，可使用本站镜像：

```
wget -N --no-check-certificate "https://xcopter.cc/down/tcp.sh" && chmod +x tcp.sh && ./tcp.sh
```

2.运行完成将出现以下菜单，可根据需要来安装相对应的核心，之后再打开加速功能。如图所示：
![](/static/2023/2023-03-01-linux-bbr-script_001.png)
以安装BBR plus为例，输入数字2来安装。重启VPS如图：
![](/static/2023/2023-03-01-linux-bbr-script_002.png)
3.安装成功，重启VPS之后我们重新连接服务器。输入下列指令来启用其BBR plus。

```
./tcp.sh
```

4.按照脚本菜单选项，选择对应安装的功能，来启用加速。
![](/static/2023/2023-03-01-linux-bbr-script_003.png)
5.如出现如图所示的信息，则表明BBR的加速功能已成功打开。
![](/static/2023/2023-03-01-linux-bbr-script_004.png)
6.如果必须安装或是转换其他版本的加速，必须再次打开脚本来进行卸载。卸载完成之后再选择所需的版本进行安装，之后需再次打开脚本来进行功能选择。