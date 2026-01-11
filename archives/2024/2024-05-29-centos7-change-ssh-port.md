# CentOS7.9 修改默认SSH端口一键脚本

给脚本修改权限：

```bash
chmod +x ssh_change_port.sh
```

执行脚本：

```bash
./ssh_change_port.sh
```

将下面的脚本保存为ssh_change_port.sh：

```bash
#!/bin/bash

# 修改SSH配置文件
sudo sed -i 's/^#Port 22/Port 22122/' /etc/ssh/sshd_config

# 检查SELinux是否启用，如果启用则添加新的端口
if sestatus | grep -q "SELinux status:.*enabled"; then
    sudo semanage port -a -t ssh_port_t -p tcp 22122
fi

# 调整防火墙配置
sudo firewall-cmd --permanent --add-port=22122/tcp
sudo firewall-cmd --permanent --remove-port=22/tcp
sudo firewall-cmd --reload

# 重启SSH服务
sudo systemctl restart sshd

echo "SSH端口已更改为22122，请使用新的端口连接。"
```
