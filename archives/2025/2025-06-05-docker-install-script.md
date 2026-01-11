# 使用官方脚本一键安装docker|安装docker-compose

## 使用官方安装脚本自动安装

安装命令如下：

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

也可以执行以下命令：

```bash
curl -fsSL https://get.docker.com | bash -s docker
```

## 设置国内镜像源

可在此命令后附带 `--mirror` 参数设置镜像源，以提高国内服务器下载 docker 的速度。

如使用阿里云镜像：

```bash
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```

## 启动和设置 Docker

设置开机自启：

```bash
systemctl enable docker
```

启动 Docker：

```bash
systemctl start docker
```

重启服务：

```bash
systemctl daemon-reload
systemctl restart docker
```
