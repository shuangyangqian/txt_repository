桥接网络：(交换机和桥是一个概念)
虚拟机的虚拟网卡和物理机的物理网卡通过一个虚拟交换机vnet0桥接，这样虚拟机和物理机在网络拓扑上相当于同等地位，需要把虚拟机和物理机配置成同样的网段。虚拟机就和现实中的一个物理主机一样，虚拟交换机vnet0就和而现实中的一台交换机的作用一样。


Nat网络：
虚拟机获得了在局域网内部使用的局域网IP，这种IP是不能直接访问互联网的。但是如果虚拟机想要访问互联网，需要通过一个具有NAT地址转换功能的路由器，将虚拟机的IP通过地址转换成一个互联网IP，这样虚拟机可以访问外网，但是外网不可以访问虚拟机。

internal模式：
内网模式，虚拟机与外网完全断开，只实现虚拟机与虚拟机之间的内部网络模式。
虚拟机与主机之间不能互相访问，不在同一个网段。虚拟机与其他主机之间也不可相互访问。虚拟机与虚拟机之间可以相互访问。

hostonly模式：
Host-Only模式其实就是NAT模式去除了虚拟NAT设备，然后使用VMware Network Adapter VMnet1虚拟网卡连接VMnet1虚拟交换机来与虚拟机通信的，Host-Only模式将虚拟机与外网隔开，使得虚拟机成为一个独立的系统，只与主机相互通讯。
http://www.linuxidc.com/Linux/2016-09/135521p3.htm

virbr0：
virbt0是KVM默认会创建的一个桥，目的是为了连接其上的虚拟机的虚拟网卡，为其提供上网功能。默认会分配一个IP：192.168.122.1，并且为其上所连的虚拟机提供DHCP服务。
#brctl show  #查看当前网桥上所连接的设备
#virsh domiflist $vmname  #查看该虚拟机上的网络设备
#cat /var/lib/libvirt/dnsmasq/default.leases  #查看都给哪些设备自动分配了IP地址

OVS：
openvswitch，是一个虚拟交换机软件，主要用在虚拟环境中。
一个交换机(桥)工作在数据链路层，通过MAC地址来进行数据的转发和传递，可以将两个LAN进行连接。
#apt-get install openvswitch-switch  #在ubuntu上安装OVS
#ovs-vsctl add-br br-test            #创建一个name为br-test的网桥
#ip link set br-test up              #使这个网络设备(网卡)生效
交换机创建成功了，如果使得网络设备连上这个交换机，这个交换机必须还得有端口。
#ovs-vsctl add-port br-test port     #为交换机创建端口
通常的交换机都有一个管理端口，可以telnet登陆进行相应的配置。相应的通过OVS创建出来的网桥也有这个功能，上面在创建OVS网桥的时候产生一个虚拟网卡(口)，给该虚拟网卡配置IP就相当于给交换机的管理端口配置了IP，和正常的交换机一样了。

添加名为br0的网桥
ovs-vsctl add-br br0
删除名为br0的网桥
ovs-vsctl del-br br0
列出所有网桥
ovs-vsctl list-br
判断网桥br0是否存在
ovs-vsctl br-exists br0
列出挂接到网桥br0上的所有网络接口
ovs-vsctl list-ports br0
将网络接口eth0挂接到网桥br0上
ovs-vsctl add-port br0 eth0
删除网桥br0上挂接的eth0网络接口
ovs-vsctl del-port br0 eth0
列出已挂接eth0网络接口的网桥
ovs-vsctl port-to-br eth0



openstack中的三种网络模式：

1.Flat（扁平）： 所有实例桥接到同一个虚拟网络，需要手动设置网桥。
2.FlatDHCP： 与Flat（扁平）管理模式类似，这种网络所有实例桥接到同一个虚拟网络，扁平拓扑。不同的是，正如名字的区别，实例的ip提供dhcp获取（nova-network节点提供dhcp服务），而且可以自动帮助建立网桥。
3.VLAN： 为每个项目提供受保护的网段（虚拟LAN）。