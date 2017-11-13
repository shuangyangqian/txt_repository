# VLAN #

LAN表示Local Area Network，本地局域网的意思，LAN中的计算机和网络设备通过Hub和Switch链接在一起。

一个LAN表示一个广播域，即同一个LAN中的所有成员都会收到任意一个成员发出的广播包。

VLAN表示Virtual LAN，一个带有VLAN功能的switch可以将自己的端口划分为多个LAN，位于同一个LAN中的计算机可以收到相互之间发出的广播包，不通lan之间的计算机不可以收到。

换言之，VLAN将一个交换机分割成多个交换机，限制了广播的范围，在二层将计算机隔离到不同的VLAN中。

## 问题抛出 ##

假设有两组计算机，A组合B组，A中的机器可以相互访问，B中的机器也可以相互访问，但是A和B之间的机器不可以相互访问。

一种方法是通过两个交换机，A和B分别连接一个交换机。
另外一种方法是通过一个带有VLAN功能的交换机，将A和B的机器分别放到不同的VLAN中。

**VLAN的隔离指的是二层上的隔离，A和B无法相互访问指的是二层广播包无法跨越VLAN的边界。但是在三层上（比如IP）是可以通过路由器使得A和B互通。**

**现有的交换机基本上都是支持VLAN功能的。**

交换机有两种端口，Access口和Trunk口。

![](http://i.imgur.com/sBy0br8.png)

Access口：

1. Access口被打上VLAN标签，每一个Access口只属于一个VLAN；
2. Access口直接与计算机网卡相连；
3. 不同的VLAN用VLAN ID来区分，VLAN ID的范围是0-4096；
4. 一旦数据从计算机网卡流入Access口，数据就被打上该口所在的VLAN标签

Truntk口：

1. Trunk口上可以通过不同VLAN数据的；
2. 交换机A和交换机B通过Trunk口连接后，A和B中具有相同VLAN ID的计算机可以相互通信；
3. 不同的VLAN数据包在通过Trunk口到达对方交换机的过程中始终带着自己的VLAN标签。

## 在KVM虚拟化环境中实现VLAN ##

![](http://i.imgur.com/pUVnJh0.png)

如上图所示：eth0是Host上的物理网卡，有一个子设备eth0.10与之相连；eth0.10相当于VLAN设备，并且VLAN ID为10；eth0.10挂在名为brvlan10的Linux Bridge上，虚拟机VM1的虚拟网卡也挂在brvlan10上。

经过上面的配置，Host通过软件实现了一个具备VLAN功能的交换机，上面定义了VLAN10。eth0.10、brvlan10和vnet10分别与VLAN10的Access口相连。eth0相当于一个Trunk口。VM1通过vnet0发出来的数据包会被打上VLAN10的标签。

eth0.10的作用是：定义了VLAN10。

brvlan10的作用是：连接到brvlan10上的其他网络设备都会自动的添加到VLAN10中。

当添加多个VLAN的时候，网络拓扑入下图：

![](http://i.imgur.com/7UaZ98x.png)


上图中显示VM1属于VLAN10，VM2属于VLAN20.

VLAN设备总是成对出现，一个母设备（eth0）可以有多个子设备（eth0.10，eth0.20...)，一个子设备只能有一个母设备与之对应。
