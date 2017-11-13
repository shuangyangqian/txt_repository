一个network是一个隔离的二层广播域

一个Project（Tenant租户）可以创建多个network

一个subnet的创建需要指定其具有的IP段和掩码

一个network可以创建多个subnet，subnet可以是不同的IP段，但是不能重叠
	有效的配置：
		networkA:   subnetA-a: 10.10.1.0/24 {"start": "10.10.1.1", "end": "10.10.1.50"}
					subnetA-b: 10.10.2.0/24 {"start": "10.10.2.1", "end": "10.10.2.50"}
	无效的配置：
		networkA:   subnetA-a: 10.10.1.0/24 {"start": "10.10.1.1", "end": "10.10.1.50"}
					subnetA-b: 10.10.1.0/24 {"start": "10.10.1.51", "end": "10.10.1.100"}
	判断上述配置有效无效的原则不是看实际的IP是否重叠，而是看subnet的CIDR重叠与否（都是10.10.1.0/24）。
	但是位于不同network的subnet的IP段和CIDR都是可以重叠的。

port可以看成是虚拟交换机上的一个端口，port定义了MAC地址和IP地址，当instance的VIF(Virtual Interface)绑定到port上时，port将MAC地址和IP地址分配给VIF。

subnet和port是一对多的关系。一个port必须属于某个subnet，一个subnet可以有多个port。

Project 1:n Network 1:n subnet 1:n port 1:1 VIF n:1 instance


几种网络模型：
	local： 不会与宿主机的任何物理网卡相连，也不关联任何VLAN ID，对于每一个local network，
			ML2 linux-bridge会创建一个bridge，instance的tap设备会连接到该bridge上。位于同一个local network的instance会连接到相同的bridge上，这样instance之间可以通信了。
			
			创建了一个local network之后，底层会创建两个设备：brqXXX和tapYYY。
			brqXXX对应的local network。tap对应的一个DHCP port。

	flat：	flat network是不带tag的网络，要求宿主机的物理网卡与linux bridge直接相连，也就是说每一个flat 			networ都会单独的站一块物理网卡。
			位于不同的host上的虚拟机，只要连接的是同一个flat网络，那么他们都可以通信。

	vlan:	vlan network 是带 tag 的网络，是实际应用最广泛的网络类型。


DHCP获取IP：
1：当创建 network 并在 subnet 上 enable DHCP 时，网络节点上的 DHCP agent 会启动一个 dnsmasq 进程为该 network 提供 DHCP 服务。
2：dnsmasq 是一个提供 DHCP 和 DNS 服务的开源软件。dnsmasq 与 network 是一对一关系，一个 dnsmasq 进程可以为同一 netowrk 中所有 enable 了 DHCP 的 subnet 提供服务。


namespace：
在二层网络上，VLAN 可以将一个物理交换机分割成几个独立的虚拟交换机。类似地，在三层网络上，Linux network namespace 可以将一个物理三层网络分割成几个独立的虚拟三层网络。


一切准备就绪，instance 获取 IP 的过程如下：
1. cirros-vm1 开机启动，发出 DHCPDISCOVER 广播，该广播消息在整个 flat_net 中都可以被收到。
2. 广播到达 veth tap19a0ed3d-fe，然后传送给 veth pair 的另一端 ns-19a0ed3d-fe。dnsmasq 在它上面监听，dnsmasq 检查其 host 文件，发现有对应项，于是dnsmasq 以  DHCPOFFER 消息将IP（172.16.1.103）、子网掩码（255.255.255.0）、地址租用期限等信息发送给 cirros-vm1。
3. cirros-vm1 发送 DHCPREQUEST 消息确认接受此 DHCPOFFER。
4. dnsmasq 发送确认消息 DHCPACK，整个过程结束。


