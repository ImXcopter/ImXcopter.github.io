# 使用官方脚本一键安装 Docker | 安装 Docker Compose

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

可在此命令后附带 `--mirror` 参数设置镜像源，以提高国内服务器下载 docker 的速度。

如使用阿里云镜像：

```bash
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```

## 设置开机自启

```bash
systemctl start docker
systemctl enable docker
```

## 重启服务

```bash
systemctl daemon-reload
systemctl restart docker
```
