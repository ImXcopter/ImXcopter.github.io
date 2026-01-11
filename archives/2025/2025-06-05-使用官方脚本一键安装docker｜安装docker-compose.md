使用官方安装脚本自动安装
------------

安装命令如下：

```
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

也可以执行以下命令：

```
curl -fsSL https://get.docker.com | bash -s docker
```

可在此命令后附带--mirror参数设置镜像源，以提高国内服务器下载docker的速度
如使用阿里云镜像:

```
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```

设置开机自启：

```
systemctl start docker
```

启动 Docker：

```
systemctl enable docker
```

重启服务 ：

```
systemctl daemon-reload
systemctl restart docker
```