### 1 在物理机上安装openvswitch

#### 1.1 编译成rpm包

```
#yum -y install wget openssl-devel gcc make python-devel openssl-devel kernel-devel kernel-debug-devel autoconf automake rpm-build redhat-rpm-config libtool

#mkdir -p rpmbuild/SOURCES
#wget https://www.openvswitch.org/releases/openvswitch-2.5.10.tar.gz
#cp openvswitch-2.5.10.tar.gz  rpmbuild/SOURCES/
#cd rpmbuild/SOURCES/
#tar zxf openvswitch-2.5.10.tar.gz
#rpmbuild --bb --nocheck openvswitch-2.5.10/rhel/openvswitch.spec
```

#### 1.2 安装并启动openvswitch

```
#cd ../RPMS/
#rpm -ivh openvswitch-2.5.10-1.x86_64.rpm 
#rpm -ivh openvswitch-debuginfo-2.5.10-1.x86_64.rpm
#systemctl start openvswitch
```

### 2 将此物理机上的vm加入到openvswitch网络中

#### 2.1创建ovs网桥

```
#ovs-vsctl add-br ovsbr
```

#### 2.2 编辑vm的XML文件

```
#virsh edit <vm>
```

修改前类似这样,vm是连接到网桥virbr0的，需要修改为刚才创建的网桥ovsbr

```
   <interface type='bridge'>
      <mac address='52:54:00:06:32:bf'/>
      <source bridge='virbr0'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>

```

修改后类似这样

```
<interface type='bridge'>
 <mac address='52:54:00:71:b1:b6'/>
 <source bridge='ovsbr'/>
 <virtualport type='openvswitch'/>
 <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
</interface>
```

#### 2.3 查看openvswitch网络

```
#ovs-vsctl show
```

#### 2.4 文档

```
https://docs.openvswitch.org/en/latest/howto/libvirt/
```

### 3 配置ovsbr的IP地址，并修改物理机路由规则

在将vm加入到ovsbr之前， vm连接的是virbr0

virbr0的IP是192.168.122.1/24

vm的IP是192.168.122.10

将vm从virbr0加入到ovsbr后，从物理机上是无法ping通vm的，原因是ovsbr没有配置IP

```
11: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN qlen 1000
    link/ether 52:54:00:7f:ba:1c brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
12: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc pfifo_fast master virbr0 state DOWN qlen 1000
    link/ether 52:54:00:7f:ba:1c brd ff:ff:ff:ff:ff:ff
22: ovs-system: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN qlen 1000
    link/ether 36:28:6b:c1:fc:e6 brd ff:ff:ff:ff:ff:ff
23: ovsbr: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN qlen 1000
    link/ether 5a:d8:c5:0c:3f:43 brd ff:ff:ff:ff:ff:ff
28: vnet0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master ovs-system state UNKNOWN qlen 1000
    link/ether fe:54:00:69:f7:87 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::fc54:ff:fe69:f787/64 scope link 
       valid_lft forever preferred_lft forever

```

#### 3.1 ovsbr配置IP

```
#ifconfig ovsbr 192.168.122.254/24 up
```

#### 3.2 修改路由规则

此时仍不能访问vm，原因是由于路由规则，之前的192.168.122.0/24网段都是通过virbr0转发的，现在要修改为ovsbr

```
#route del -net 192.168.122.0/24
#route add -net 192.168.122.0/24 gateway ovsbr
```



### 4 不同物理上的vm互相访问

#### 4.1 文档

```
https://docs.openvswitch.org/en/latest/howto/tunneling/
```

#### 4.2 场景

| 物理机IP     | vm名称 | vm IP          | vm tap name on ovs | ovs bridge name | ovs bridge ip   | route                                         |
| ------------ | ------ | -------------- | ------------------ | --------------- | --------------- | --------------------------------------------- |
| 192.168.6.42 | vm5    | 192.168.122.14 | vnet0              | ovsbr           | 192.168.122.254 | route add -net 192.168.122.0/24 gateway ovsbr |
| 192.168.6.42 | vm6    | 192.168.122.15 | vnet1              | ovsbr           | 192.168.122.254 | 同上                                          |
| 192.168.6.43 | vm1    | 192.168.122.10 | vnet0              | ovsbr           | 192.168.122.253 | 同上                                          |
| 192.168.6.43 | vm2    | 192.168.122.11 | vnet1              | ovsbr           | 192.168.122.253 | 同上                                          |

目前vm1和vm2可以互访， vm5和vm6可以互访，现在我们将使用 GRE tunnel来实现2台物理机上vm的互访

#### 4.3 添加GRE Tunnel或VXLAN

在192.168.6.42上添加：

GRE Tunnel

```
#ovs-vsctl add-port ovsbr gre0 -- set interface gre0 type=gre options:remote_ip=192.168.6.43
```

VXLAN Tunnel

```
#ovs-vsctl add-port ovsbr vxlan0 -- set interface vxlan0 type=vxlan options:remote_ip=192.168.6.43
```



在192.168.6.43上添加:

GRE Tunnel

```
#ovs-vsctl add-port ovsbr gre0 -- set interface gre0 type=gre options:remote_ip=192.168.6.42
```

VXLAN Tunnel

```
#ovs-vsctl add-port ovsbr vxlan0 -- set interface vxlan0 type=vxlan options:remote_ip=192.168.6.42
```



查看192.168.6.42上

```
#ovs-ofctl show ovsbr
OFPT_FEATURES_REPLY (xid=0x2): dpid:00006622c9835846
n_tables:254, n_buffers:256
capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
actions: output enqueue set_vlan_vid set_vlan_pcp strip_vlan mod_dl_src mod_dl_dst mod_nw_src mod_nw_dst mod_nw_tos mod_tp_src mod_tp_dst
 2(gre0): addr:fe:0b:96:27:ca:a9
     config:     0
     state:      0
     speed: 0 Mbps now, 0 Mbps max
 4(vnet1): addr:fe:54:00:e3:33:34
     config:     0
     state:      0
     current:    10MB-FD COPPER
     speed: 10 Mbps now, 0 Mbps max
 5(vnet0): addr:fe:54:00:ed:ec:b1
     config:     0
     state:      0
     current:    10MB-FD COPPER
     speed: 10 Mbps now, 0 Mbps max
 LOCAL(ovsbr): addr:66:22:c9:83:58:46
     config:     0
     state:      0
     speed: 0 Mbps now, 0 Mbps max
OFPT_GET_CONFIG_REPLY (xid=0x4): frags=normal miss_send_len=0
```

```
#ovs-vsctl show
f730d99a-0e87-4ae2-a1ce-626e7aac4517
    Bridge ovsbr
        Port ovsbr
            Interface ovsbr
                type: internal
        Port "vnet1"
            Interface "vnet1"
        Port "vnet0"
            Interface "vnet0"
        Port "gre0"
            Interface "gre0"
                type: gre
                options: {remote_ip="192.168.6.43"}
    ovs_version: "2.5.10"
```

```
#route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.10.254  0.0.0.0         UG    100    0        0 em1
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.0.0     0.0.0.0         255.255.0.0     U     100    0        0 em1
192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 ovsbr
```



查看192.168.6.43上

```
#ovs-vsctl show
ece10d3a-41bd-493a-8914-96e502dee08b
    Bridge ovsbr
        Port "vnet1"
            Interface "vnet1"
        Port "gre0"
            Interface "gre0"
                type: gre
                options: {remote_ip="192.168.6.42"}
        Port ovsbr
            Interface ovsbr
                type: internal
        Port "vnet0"
            Interface "vnet0"
    ovs_version: "2.5.10"
```

```
#ovs-ofctl show ovsbr
OFPT_FEATURES_REPLY (xid=0x2): dpid:00005ad8c50c3f43
n_tables:254, n_buffers:256
capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
actions: output enqueue set_vlan_vid set_vlan_pcp strip_vlan mod_dl_src mod_dl_dst mod_nw_src mod_nw_dst mod_nw_tos mod_tp_src mod_tp_dst
 2(gre0): addr:2a:07:02:70:79:b4
     config:     0
     state:      0
     speed: 0 Mbps now, 0 Mbps max
 3(vnet0): addr:fe:54:00:69:f7:87
     config:     0
     state:      0
     current:    10MB-FD COPPER
     speed: 10 Mbps now, 0 Mbps max
 4(vnet1): addr:fe:54:00:98:17:63
     config:     0
     state:      0
     current:    10MB-FD COPPER
     speed: 10 Mbps now, 0 Mbps max
 LOCAL(ovsbr): addr:5a:d8:c5:0c:3f:43
     config:     0
     state:      0
     speed: 0 Mbps now, 0 Mbps max
OFPT_GET_CONFIG_REPLY (xid=0x4): frags=normal miss_send_len=0
```

```
#route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.10.254  0.0.0.0         UG    100    0        0 em1
192.168.0.0     0.0.0.0         255.255.0.0     U     100    0        0 em1
192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 ovsbr
```

