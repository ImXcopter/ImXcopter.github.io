安装OpenSSH服务器：首先，您需要在Windows 10 LTSC机器上安装OpenSSH服务器。可以通过“设置”>“系统”>“可选功能”>“添加功能”>在列表中找到并选择“OpenSSH服务器”>点击“安装”来完成此操作。
配置OpenSSH服务器：安装完成后，您需要打开OpenSSH的配置文件，通常位于"C:\ProgramData\ssh\sshd\_config"。在这个文件中，您可以设置许多SSH服务器的参数。例如，如果您想要允许远程桌面转发，您需要将"AllowTcpForwarding"选项设置为"yes"。
启动OpenSSH服务：配置完成后，您需要启动OpenSSH服务。您可以通过在命令提示符中输入"services.msc"，然后在服务列表中找到并启动"OpenSSH SSH Server"服务来完成此操作，同时可添加自启动服务。
在远程计算机上设置SSH客户端：在远程计算机上，您需要设置SSH客户端以连接到您的OpenSSH服务器。在Windows 10中，您可以使用内置的OpenSSH客户端，也可以使用第三方SSH客户端，如PuTTY。在SSH客户端中，您需要输入服务器的IP地址和SSH端口（默认为22），以及您在服务器上的用户名和密码。
启动远程桌面转发：最后，您需要在SSH客户端中启动远程桌面转发。在OpenSSH客户端中，您可以通过在命令行中输入"ssh -L 本地端口:远程计算机的IP地址:远程端口 用户名@服务器IP地址"来完成此操作。例如，如果您想要将本地的3389端口转发到远程计算机的3389端口，您可以输入"ssh -L 3389:localhost:3389 用户名@服务器IP地址"。
参考网址：
https://learn.microsoft.com/zh-cn/windows-server/administration/openssh/openssh\_install\_firstuse