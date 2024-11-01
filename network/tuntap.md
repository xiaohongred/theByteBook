# 3.5.2 虚拟网络设备 tun 和 tap

tun 和 tap 是 Linux 内核 2.4.x 版本之后引入的虚拟网卡设备，主要用于用户空间（user space）和内核空间（kernel space）双向传输数据。这两种设备的区别与含义为：
- tun 设备工作在网络层，处理的是 IP 数据包。它模拟的是一个网络层接口，允许用户空间程序接收和发送 IP 层的数据包。tun 设备常用于 VPN 应用，将网络流量从一个远程网络重定向到本地虚拟网络接口；
- tap 设备工作在数据链路层，处理的是以太网帧。它模拟的是一个以太网接口，允许用户空间程序发送和接收以太网帧。TAP 设备通常用于虚拟机网络、桥接网络、虚拟交换机等场景，以模拟完整的二层网络通信。


Linux 系统中，内核空间和用户空间之间数据传输有多种方式，字符设备文件是其中一种。tap/tun 对应的字符设备文件为 /dev/net/tun。

当用户空间的程序 open() 字符设备文件时，返回一个 fd 句柄，同时字符设备驱动创建并注册相应的虚拟网卡网络接口，并以 tunX 或 tapX 命名。当用户空间的程序向 fd 执行 read()/write() 时，就可以和内核网络互通了。

tun 和 tap 的工作原理基本相同，只是两者工作的网络层面不一样。如图 3-13所示，笔者以 tun 设备构建 VPN 隧道为例，说明其工作原理：
- 首先，一个普通的用户程序发起一个网络请求；
- 接着，数据包进入内核协议栈时并找路由。大致的路由规则如下:
	```bash
	$ ip route show
	default via 172.12.0.1 dev tun0  // 默认流量经过 tun0 设备
	192.168.0.0/24 dev eth0  proto kernel  scope link  src 192.168.0.3
	```
- tun0 设备字符文件 /dev/net/tun 由 VPN 程序打开。所以，第一步用户程序发送的数据包被 VPN 程序接管。
- VPN 程序对数据包进行封装操作，“封装”是指将一个数据包包装在另一个数据包中，就像将一个盒子放在另一个盒子中一样。封装后的数据再次被发送到内核，最后通过 eth0 接口（也就是图中的物理网卡）发出。

:::center
  ![](../assets/tun.svg)<br/>
 图 3-13 VPN 中数据流动示意图
:::

将一个数据包封装到另一个数据包是构建网络隧道，实现虚拟网络的典型方式。

本书第七章介绍的容器网络插件 Flannel，早期的设计中曾使用 tun 设备实现了 UDP 模式下的跨主机通信，但使用 tun 设备传输数据需要经过两次协议栈，且有多次的封包/解包过程，产生额外的性能损耗。这是后来 Flannel 弃用 UDP 模式的主要原因。
