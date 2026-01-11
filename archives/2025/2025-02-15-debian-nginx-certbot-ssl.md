# Debian安装nginx 并配置带自动更新 SSL 证书的完整安装流程

本文档介绍如何在系统上配置 Nginx 反向代理，安装 SSL 证书，并设置自动更新证书任务。

## 1. 安装 Nginx 和 Certbot

首先，更新软件包列表，然后安装 Nginx 和 Certbot 及其 Nginx 插件：

```bash
sudo apt update
sudo apt install nginx certbot python3-certbot-nginx
```

## 2. 创建 Nginx 配置文件

在 `/etc/nginx/sites-available/` 目录下为你的域名创建一个 Nginx 配置文件：

```bash
sudo nano /etc/nginx/sites-available/your_domain_name.com
```

## 3. 添加基础配置

在打开的编辑器中，添加如下配置内容，将 `your_domain_name.com` 替换成你的真实域名，同时确保后端代理地址正确：

```nginx
server {
    listen 80;
    server_name your_domain_name.com;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

保存并退出编辑器。

## 4. 创建符号链接

将新配置启用，通过创建符号链接到 `sites-enabled` 目录：

```bash
sudo ln -s /etc/nginx/sites-available/your_domain_name.com /etc/nginx/sites-enabled/
```

## 5. 测试并重启 Nginx

测试 Nginx 配置文件是否正确，然后重启 Nginx 服务使配置生效：

```bash
sudo nginx -t
sudo systemctl restart nginx
```

## 6. 申请 SSL 证书

使用 Certbot 为你的域名申请 SSL 证书，并自动修改 Nginx 配置以启用 HTTPS：

```bash
sudo certbot --nginx -d your_domain_name.com
```

根据提示完成交互式配置。

## 7. 设置自动更新证书的定时任务

首先检查是否已有 certbot 自动更新任务：

```bash
systemctl list-timers | grep certbot
```

如果没有自动更新任务，则启用并启动 certbot 定时任务（默认每天检查两次，只在证书快过期时进行更新）：

```bash
sudo systemctl enable certbot.timer
sudo systemctl start certbot.timer
```

## 8. 验证定时任务是否正常

可以使用下面的命令测试自动更新是否工作：

```bash
sudo certbot renew --dry-run
```

同时也可以查看定时任务和更新服务的状态：

```bash
# 查看定时器状态
systemctl status certbot.timer

# 查看更新服务状态
systemctl status certbot.service
```

## 手动测试证书更新

若需要手动测试并强制更新 SSL 证书，可运行以下命令：

```bash
sudo certbot renew --force-renewal
```

---

通过以上步骤，你的系统已经具备：

- 配置了 Nginx 反向代理
- 安装并启用了 SSL 证书
- 设置了自动更新证书功能

这样可以确保证书在快过期时能够自动更新，保障 HTTPS 访问的正常运行。
