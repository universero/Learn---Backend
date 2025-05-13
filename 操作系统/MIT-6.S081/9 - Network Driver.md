___
## Lab: networking

In this lab we will write an xv6 device driver for a network interface card (NIC).

使用E1000作为网络设备, xv6作为客户端持有IP 10.0.2.15, 宿主机作为主机, 具有id 10.0.2.2, 发送和接收到的包存在packets.pcap文件中, 使用如下命令可以查看记录的包
```
tcpdump -XXnr packets.pcap
```

相关文件
- kernel/e1000.c 包含了初始化E1000以及空的发送和接受函数
- kernel/e1000_dev.h 包含了E1000定义的寄存器和标识符
- kernel/net.c和kernel/net.h 包含了建议的网络协议栈, 实现了IP, UDP, ARP, 以及mbuf一个灵活的struct来存储packets
- kernel/pic.c 存储了在PCI bus上寻找E1000的代码
