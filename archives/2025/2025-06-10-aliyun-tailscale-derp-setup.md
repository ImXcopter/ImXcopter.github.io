# 阿里云安装Tailscale DERP小记

## 为什么需要 Derp 服务

这是由于 Tailscale 机制决定的。Tailscale 只会在需要与 peer 建立连接的时候才会尝试打洞，而且最开始的流量一定是会经过 DERP 中转服务器。

**优点**：懒加载机制无需预先维护与其他节点的任何打洞连接，无需预先维护任何状态。

**缺点**：每次通过 Tailscale 创建虚拟连接时，初始所创建的连接其延迟很高，这会极大的影响使用体验，Tailscale 极其依赖中继节点。

当然官方是有提供默认的一系列 Derp 服务器，可以通过 `tailscale netcheck` 命令来查看：

```text
Report:
        * UDP: true
        * IPv4: yes, xxxx
        * IPv6: no, but OS has support
        * MappingVariesByDestIP: true
        * HairPinning: false
        * PortMapping:
        * CaptivePortal: false
        * Nearest DERP: lax
        * DERP latency:
                - lax: 165.8ms (Los Angeles)
                - sfo: 173ms   (San Francisco)
                - tok: 173.8ms (Tokyo)
                - sea: 175.6ms (Seattle)
                - den: 181.5ms (Denver)
                - dfw: 184.1ms (Dallas)
                - ord: 200.7ms (Chicago)
                - hkg: 207.5ms (Hong Kong)
                - hnl: 209ms   (Honolulu)
                - nyc: 213.5ms (New York City)
                - mia: 220.6ms (Miami)
                - iad: 223.6ms (Ashburn)
                - par: 229ms   (Paris)
                - lhr: 230.2ms (London)
                - tor: 231.1ms (Toronto)
                - sin: 234.8ms (Singapore)
                - fra: 238.3ms (Frankfurt)
                - ams: 248.9ms (Amsterdam)
                - nue: 253.4ms (Nuremberg)
                - mad: 266.3ms (Madrid)
                - waw: 266.6ms (Warsaw)
                - blr: 271.1ms (Bangalore)
                - syd: 322ms   (Sydney)
                - sao: 330.8ms (São Paulo)
                - dbi: 348.1ms (Dubai)
                - jnb: 384.2ms (Johannesburg)
                - nai: 393.9ms (Nairobi)
```

可以看到基本上都是国外的服务器，这也就意味着，如果打洞失败，走 DERP 服务器中转的话，会导致延迟非常之高。

## 搭建 Derp 服务

以下为一些硬性要求：

- 需要能够公网访问。这是为了让各个 Tailscale 节点可以直接访问到该 DERP 服务器
- 需要运行 HTTPS 服务。本质上是为了在传输数据给 DERP 服务器时数据可以通过 TLS 加密。
- 分配 80 端口来运行 HTTP 服务。
- 需要额外暴露两个端口来运行 HTTPS 和 STUN 服务。
- 必须允许 ICMP 流量的出入。

### 安装运行依赖

大多数干净的系统执行第一步就会执行不下去，这个是需要依赖 Go 环境，而 Tailscale 的 Go 版本总是会依赖最新的 Go。

以下提供当前的 Go 1.24.5 的安装，其他版本或者更新的版本可以到 Go 官网下载安装：<https://go.dev/doc/install>

```bash
wget https://go.dev/dl/go1.25.5.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.25.5.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
go version
go env -w GOPROXY=https://goproxy.cn,direct
```

上面执行完，就安装好了 Go 环境，同时指定了 Go 代理地址，因为阿里云服务器下载 github 包大概率会失败。

### Derper 官方安装

官方提供的安装步骤就两步：

1. 安装 Derper 程序

```bash
go install tailscale.com/cmd/derper@latest
```

2. 安装到系统路径

```bash
sudo cp /root/go/bin/derper /usr/local/bin/
```

3. 检查 derper 是否正确安装：

```bash
which derper
```

如果返回 `/usr/local/bin/derper`，说明 derper 安装成功并可以执行。这样，derper 将被安装到系统的标准路径 `/usr/local/bin/`，你应该能够在任何地方通过 `derper` 命令运行。

### 用IP代替域名

只要用 IP 生成一个自签名证书即可，如果直接尝试 `openssl x509` 可不太行。

需要将 IP 地址作为证书的 Subject Alternative Name (SAN) 添加到证书中。

因为前面已经安装了 Go 的运行环境了，所以我这里提供一个 Go 版本的生成方案：

```bash
mkdir -p ~/certdir
cd ~/certdir
```

将下面这段代码复制，然后 `vim main.go` 粘贴后改成你的 IP 保存。

```go
package main

import (
	"crypto/ecdsa"
	"crypto/elliptic"
	"crypto/rand"
	"crypto/x509"
	"crypto/x509/pkix"
	"encoding/pem"
	"fmt"
	"log"
	"math/big"
	"net"
	"os"
	"time"
)

func main() {
	// IP地址，替换为你自己的服务器实际IP
	ip := "127.0.0.1"

	// 生成密钥对
	priv, err := ecdsa.GenerateKey(elliptic.P384(), rand.Reader)
	if err != nil {
		log.Fatalf("Failed to generate private key: %v", err)
	}

	// 生成证书模板
	template := x509.Certificate{
		SerialNumber:          big.NewInt(12345),
		Subject:               pkix.Name{Organization: []string{"Example Org"}},
		NotBefore:             time.Now(),
		NotAfter:              time.Now().Add(365 * 24 * time.Hour * 100),
		KeyUsage:              x509.KeyUsageKeyEncipherment | x509.KeyUsageDigitalSignature,
		ExtKeyUsage:           []x509.ExtKeyUsage{x509.ExtKeyUsageServerAuth},
		BasicConstraintsValid: true,
		DNSNames:              []string{ip},
		IPAddresses:           []net.IP{net.ParseIP(ip)}, // 指定 IP 地址
	}

	// 使用私钥签发证书
	certDER, err := x509.CreateCertificate(rand.Reader, &template, &template, &priv.PublicKey, priv)
	if err != nil {
		log.Fatalf("Failed to create certificate: %v", err)
	}

	// 保存证书文件
	certFile, err := os.Create(fmt.Sprintf("%s.crt", ip))
	if err != nil {
		log.Fatalf("Failed to create cert file: %v", err)
	}
	defer certFile.Close()
	err = pem.Encode(certFile, &pem.Block{Type: "CERTIFICATE", Bytes: certDER})
	if err != nil {
		log.Fatalf("Failed to write cert to file: %v", err)
	}

	// 保存私钥文件
	keyFile, err := os.Create(fmt.Sprintf("%s.key", ip))
	if err != nil {
		log.Fatalf("Failed to create key file: %v", err)
	}
	defer keyFile.Close()
	privBytes, err := x509.MarshalECPrivateKey(priv)
	if err != nil {
		log.Fatalf("Failed to marshal private key: %v", err)
	}
	err = pem.Encode(keyFile, &pem.Block{Type: "EC PRIVATE KEY", Bytes: privBytes})
	if err != nil {
		log.Fatalf("Failed to write key to file: %v", err)
	}

	fmt.Println("Certificate and key saved successfully!")
}
```

然后执行：

```bash
go run main.go
```

之后就会在当前目录下生成以你填的 IP 命名的 `.crt` 和 `.key` 密钥。

这时，官方的第二步就可以改成：

```bash
sudo derper -a :8888 -http-port -1 -stun-port 3478 -hostname x.x.x.x --certmode manual -certdir ~/certdir
```

或者：

```bash
sudo /usr/local/bin/derper -a :8888 -http-port -1 -stun-port 3478 -hostname 127.0.0.1 --certmode manual -certdir ~/certdir
```

**参数解释**：

- `x.x.x.x` 换成你的 IP
- `-1` 表示禁用了 http 的端口
- `8888` 为 https 的端口
- `3478` 为 stun 端口

当你能看到下面这段日志输出，就说明启动成功了。

```text
2025/01/11 11:36:28 derper: serving on :8888 with TLS
2025/01/11 11:36:28 running STUN server on [::]:3478
```

不要结束服务，此时你自己的电脑需要连接到 Tailscale 的机器上输入 `curl --insecure "https://x.x.x.x:8888"` 或者直接浏览器访问 `https://x.x.x.x:8888` 看到下面这串输出，就说明 DERP 搭建成功了。

```text
DERP

This is a Tailscale DERP server.

It provides STUN, interactive connectivity establishment, and relaying of end-to-end encrypted traffic
for Tailscale clients.

...
```

## 注册自己的 Derp 服务到列表中

如果你用的 Tailscale 官方服务，到 <https://login.tailscale.com/admin/acls/file> 添加你的 Derp 服务器。在网页上保存好 ACL 后，ACL 会立即下发到各个 Tailscale 节点里。随便找个节点运行 netcheck，可以发现 DERP 成功添加。

这里给出参考配置：

```json
{
	// Declare static groups of users. Use autogroups for all users or users with a specific role.
	// "groups": {
	//  	"group:example": ["alice@example.com", "bob@example.com"],
	// },

	// Define the tags which can be applied to devices and by which users.
	// "tagOwners": {
	//  	"tag:example": ["autogroup:admin"],
	// },

	// Define grants that govern access for users, groups, autogroups, tags,
	// Tailscale IP addresses, and subnet ranges.
	"grants": [
		// Allow all connections.
		// Comment this section out if you want to define specific restrictions.
		{"src": ["*"], "dst": ["*"], "ip": ["*"]},

		// Allow users in "group:example" to access "tag:example", but only from
		// devices that are running macOS and have enabled Tailscale client auto-updating.
		// {"src": ["group:example"], "dst": ["tag:example"], "ip": ["*"], "srcPosture":["posture:autoUpdateMac"]},
	],

	// Define postures that will be applied to all rules without any specific
	// srcPosture definition.
	// "defaultSrcPosture": [
	//      "posture:anyMac",
	// ],

	// Define device posture rules requiring devices to meet
	// certain criteria to access parts of your system.
	// "postures": {
	//      // Require devices running macOS, a stable Tailscale
	//      // version and auto update enabled for Tailscale.
	// 	"posture:autoUpdateMac": [
	// 	    "node:os == 'macos'",
	// 	    "node:tsReleaseTrack == 'stable'",
	// 	    "node:tsAutoUpdate",
	// 	],
	//      // Require devices running macOS and a stable
	//      // Tailscale version.
	// 	"posture:anyMac": [
	// 	    "node:os == 'macos'",
	// 	    "node:tsReleaseTrack == 'stable'",
	// 	],
	// },

	// Define users and devices that can use Tailscale SSH.
	"ssh": [
		// Allow all users to SSH into their own devices in check mode.
		// Comment this section out if you want to define specific restrictions.
		{
			"action": "check",
			"src":    ["autogroup:member"],
			"dst":    ["autogroup:self"],
			"users":  ["autogroup:nonroot", "root"],
		},
	],

	// Test access rules every time they're saved.
	// "tests": [
	//  	{
	//  		"src": "alice@example.com",
	//  		"accept": ["tag:example"],
	//  		"deny": ["100.101.102.103:443"],
	//  	},
	// ],
	"derpMap": {
		"Regions": {
			"901": {
				"RegionID":   901,
				"RegionCode": "aliyun-bj",
				"RegionName": "Aliyun BeiJing",
				"Nodes": [
					{
						"Name":             "901",
						"RegionID":         901,
						"DERPPort":         8888,
						"STUNPort":         3478,
						"IPv4":             "x.x.x.x",
						"InsecureForTests": true,
					},
				],
			},
		},
	},
}
```

### 检查自定义 Derp 是否生效

在 cmd 下输入 `tailscale netcheck` 检查你的客户端是否能够正常检索到自己的 Derp 服务器：

```bash
tailscale netcheck

Report:
* UDP: true
* IPv4: yes, xxxxx
* IPv6: no, but OS has support
* MappingVariesByDestIP: true
* HairPinning: false
* PortMapping:
* CaptivePortal: false
* Nearest DERP: Aliyun Shenzhen
* DERP latency:
- ali-sz: 13ms (Aliyun Shenzhen)
- lax: 165.8ms (Los Angeles)
- sfo: 173ms (San Francisco)
- tok: 173.8ms (Tokyo)
- sea: 175.6ms (Seattle)
- den: 181.5ms (Denver)
- den: 181.5ms (Denver)
- den: 181.5ms (Denver)
- den: 181.5ms (Denver)
- den: 181.5ms (Denver)
- dfw: 184.1ms (Dallas)
- ord: 200.7ms (Chicago)
- den: 181.5ms (Denver)
- dfw: 184.1ms (Dallas)
- ord: 200.7ms (Chicago)
- hkg: 207.5ms (Hong Kong)
- hnl: 209ms (Honolulu)
- nyc: 213.5ms (New York City)
- mia: 220.6ms (Miami)
- iad: 223.6ms (Ashburn)
- par: 229ms (Paris)
- lhr: 230.2ms (London)
- tor: 231.1ms (Toronto)
- sin: 234.8ms (Singapore)
- fra: 238.3ms (Frankfurt)
- ams: 248.9ms (Amsterdam)
- nue: 253.4ms (Nuremberg)
- mad: 266.3ms (Madrid)
- waw: 266.6ms (Warsaw)
- blr: 271.1ms (Bangalore)
- syd: 322ms (Sydney)
- sao: 330.8ms (São Paulo)
- dbi: 348.1ms (Dubai)
- jnb: 384.2ms (Johannesburg)
- nai: 393.9ms (Nairobi)
```

## Derp 服务安全

前面跑起来的 Derp 服务，其他人只要知道了 IP 和端口是可以访问的。

因此需要额外进行一些处理。你需要在 Derp 服务器上安装 Tailscale 客户端

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

然后运行

```bash
tailscale up
```

注册到你的 Tailscale 中。

之后命令行会提示在浏览器登录 https://login.tailscale.com/a/xxxxxxxx，登录成功后出现 "Success" 即注册成功：

```text
To authenticate, visit:

https://login.tailscale.com/a/xxxxxxxx

Success.
```

之后在启动 Derper 时添加参数 `--verify-clients` 这个只允许你的 Tailscale 中的机器才能访问。

完整的命令如下：

```bash
derper -a :8888 -http-port -1 -stun-port 3478 -hostname x.x.x.x --certmode manual -certdir ~/certdir --verify-clients
```

## 将 Derper 注册成服务，自动启动

将文件命名为：`derper.service`

```ini
[Unit]
Description=DERP Relay Server
After=network.target
Wants=network.target

[Service]
User=root
Restart=always
ExecStart=/usr/local/bin/derper -a :8888 -http-port -1 -stun-port 3478 -hostname x.x.x.x --certmode manual -certdir /root/certdir --verify-clients
RestartPreventExitStatus=1

[Install]
WantedBy=multi-user.target
```

保存到 `/etc/systemd/system/derper.service`

```bash
# 重新加载 systemd 配置
sudo systemctl daemon-reexec
sudo systemctl daemon-reload

# 启动 derper 服务
sudo systemctl start derper.service

# 设置开机自启
sudo systemctl enable derper.service

# 查看状态
sudo systemctl status derper.service
```

**说了这么多，总结下来就这么几步：**

1. 安装 Go 环境！
2. 安装 Derper 程序
3. 安装 Tailscale 程序并且 `tailscale up` 注册到 Tailscale 网络中
4. 制作 derper.service 启动程序
5. 启动 derper 服务并且注册到开机自启动中！

### 以下为 Aliyun ECS Debian12.12 64bit 下安装成功的具体执行的命令顺序：

#### 一、修改主机名

1. 设置主机名

```bash
hostnamectl set-hostname aliyun-beijing
```

2. 配置 `/etc/hosts`（非常重要）

```bash
cat > /etc/hosts <
```

3. 重启生效

```bash
reboot
```

#### 二、安装 Go 1.25.5（手动方式）

1. 更新本地的软件包索引

```bash
apt update
```

2. 手动下载 Go 安装包，文件名：

```text
go1.25.5.linux-amd64.tar.gz
```

通过 SFTP 上传到 `/root` 目录

3. 解压并安装

```bash
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.25.5.linux-amd64.tar.gz
```

4. 配置 Go 环境变量

```bash
export PATH=$PATH:/usr/local/go/bin
```

验证：

```bash
go version
```

5. 配置 Go 国内代理

```bash
go env -w GOPROXY=https://goproxy.cn,direct
```

#### 三、安装并准备 derper

1. 编译 derper

```bash
go install tailscale.com/cmd/derper@latest
```

2. 安装到系统路径

```bash
cp ~/go/bin/derper /usr/local/bin/
```

验证：

```bash
which derper
```

#### 四、准备证书与配置目录

1. 创建证书目录

```bash
mkdir -p ~/certdir
cd ~/certdir
```

2. 手动上传证书与代码

通过 SFTP 上传以下文件到：

```text
/root/certdir/
```

（包含 main.go 及证书文件）

3. 测试运行

```bash
go run main.go
```

4. 切换回上级目录

```bash
cd ..
```

#### 五、安装 Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

启动并登录：

```bash
tailscale up
```

#### 六、配置 derper systemd 服务

1. 上传服务文件 通过 SFTP 上传：

```text
/etc/systemd/system/derper.service
```

2. 重新加载 systemd

```bash
systemctl daemon-reexec
systemctl daemon-reload
```

3. 启动服务

```bash
systemctl start derper.service
```

4. 设置开机自启

```bash
systemctl enable derper.service
```

5. 查看运行状态

```bash
systemctl status derper.service
```

本文转载并修改自：<https://bigorange.work/p/88ab7236/>
