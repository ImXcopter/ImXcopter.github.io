# 安装Midjourney Proxy小记

项目地址
----

```
https://github.com/trueai-org/midjourney-proxy/blob/main/README.md
```

在一台全新的Debian12的VPS下安装成功，以下是安装过程。
Midjourney Proxy Linux 一键安装脚本：
# 方式1

```
wget -N --no-check-certificate https://raw.githubusercontent.com/trueai-org/midjourney-proxy/main/scripts/linux_install.sh && chmod +x linux_install.sh && bash linux_install.sh
```

# 方式2

```
curl -o linux_install.sh https://raw.githubusercontent.com/trueai-org/midjourney-proxy/main/scripts/linux_install.sh && chmod +x linux_install.sh && bash linux_install.sh
```

根据提示进行操作即可：
选择1. 安装Docker
选择2. 拉取镜像并启动容器

安装完成后，可以安装Nginx和Certbot对项目配置 Nginx 来反向代理 8086 端口并自动获取和续签域名 SSL 证书
-----------------------------------------------------------------

第一步：安装 Nginx 和 Certbot

```
sudo apt update
sudo apt install nginx certbot python3-certbot-nginx -y
```

第二步：配置 Nginx 虚拟主机
1. 创建配置文件：

```
sudo nano /etc/nginx/sites-available/midjourney-proxy.your-domain-name.com
```

2. 写入以下内容：

```
server {
    listen 80;
    server_name midjourney-proxy.your-domain-name.com;

    location / {
        proxy_pass http://127.0.0.1:8086;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

3. 启用该配置并重启 Nginx：

```
sudo ln -s /etc/nginx/sites-available/midjourney-proxy.your-domain-name.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

第三步：使用 Certbot 配置 SSL
运行下面的命令获取证书并自动配置 HTTPS：

```
sudo certbot --nginx -d midjourney-proxy.your-domain-name.com
```

过程说明：
Certbot 会自动修改 Nginx 配置，加上 SSL 和自动重定向 HTTP → HTTPS。
你需要确保 DNS 解析正确（即 midjourney-proxy.your-domain-name.com 已指向你的服务器 IP）。
第四步：确认自动续签设置
Certbot 会自动添加续签任务到 cron 或 systemd timer，你可以验证：

```
sudo systemctl list-timers | grep certbot
```

你也可以手动测试自动续签：

```
sudo certbot renew --dry-run
```

配置完成后浏览器访问：

```
https://midjourney-proxy.your-domain-name.com
```

此时，应该可以安全地通过 SSL 访问，并且请求被代理到本地的 8086 端口。
如你使用的是防火墙（如 ufw），记得放行 80 和 443 端口：

```
sudo ufw allow 80
sudo ufw allow 443
```