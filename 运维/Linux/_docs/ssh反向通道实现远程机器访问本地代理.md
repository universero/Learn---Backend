___

有时远程的服务器无法访问外网, linux系统上安装clash又不方便, 此时可以尝试用ssh建立一个反向的通道, 将服务器上指定端口的请求发送到本机的端口, 从而实现流量转发的效果, 来实现外网访问

## 建立反向代理通道

在本地机器上执行
`ssh -R 6990:localhost:7890 ubuntu@43.143.27.198

- `ssh -R` 表示建立一个反向 SSH 隧道
- `6990` 是在公网服务器上监听的端口
- `localhost` 本地机器的地址
- `7890` 本地机器代理服务的地址，这里使用的是 clash
- `ubuntu@47.65.55.62` 公网服务器的用户名和 IP 地址
- `-N` 若加上则表示不执行远程命令，仅建立隧道连接
因此上述命令的作用是：将本地的 `7890` 端口映射到公网服务器 `47.65.55.62` 的 `6990` 端口，当公网服务器有请求发送到公网服务器的 `6990` 端口时，它会通过 SSH 隧道转发到本地机器的 `7890` 端口。

在本地机器执行上述命令前先在公网服务器上确认配置了允许端口转发，`vim /etc/ssh/sshd_config` 后找到 `AllowTcpForwarding` 和 `GatewayPorts`，这两个配置需要都设置为 `yes`，一般是注释了，可以在这两者的下方添加如下配置，并执行 `systemctl restart sshd` 命令重启 SSH 服务：

```
AllowTcpForwarding yes
GatewayPorts yes
```

设置代理
```shell
export https_proxy=http://localhost:6990;
export http_proxy=http://localhost:6990;
export all_proxy=socks5://localhost:6990;
```

- `ssh -R` 命令的作用是将远程服务器的端口映射到本地机器的端口，这也可以实现局域网内的服务在公网访问，不仅仅是代理的作用
- 当ssh关闭后服务器也无法继续访问外网

## 简化

可以在本机的.ssh/config 添加RemoteForward选项, 示例如下
然后在服务器上写个脚本, 执行后切换http代理
```
Host u1
    HostName 43.143.27.198
    User root
    IdentityFile ~/.ssh/univero_rsa
    RemoteForward 6990 localhost:7890
```