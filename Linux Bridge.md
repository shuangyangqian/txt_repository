# Linux Bridge #
当Host上有一台虚拟机，Host上只有一块eth0的物理网卡，如何能使得虚拟机也访问外网？

## 解决方案 ##
给虚拟机分配一块虚拟网卡vnet0，Host上的物理网卡eth0和虚拟机上的vnet0通过Linux Bridge **br0**链接。如下图所示：

![](http://i.imgur.com/lskNfi3.png)

这样vnet0和eth0任何一方接收到数据包的时候，br0都会将数据包转发给对方，这样虚拟机就可以链接外网可以将br0理解为二层交换设备或者Hub。

当增加一台虚拟机的时候，网络拓扑可以表示为下图：

![](http://i.imgur.com/Ub1M0kw.png)

将虚拟机VM2的虚拟网卡也添加到br0上，这样使得VM1和VM2之间可以相互通信，VM1和VM2也可以和外网通信。

## 配置Linux Bridge br0 ##

修改Host配置文件/etc/network/interfaces，配置br0，下图表示了修改之后与修改之前文件interfaces的变化：

![](http://i.imgur.com/8SSOgBm.png)

1. 其中auto br0表示的是将Host通过DHCP获得的ip地址分配给br0；
2. bridge_ports eth0表示的是将eth0绑定到br0上。 

通过上述设置之后可以通过ifconfig查看Host上的ip地址已经分配给br0了。