### Ryu控制器在1.5T服务器上的实验：

首先需要一个闲置的浮动ip和一个闲置的蜜罐进行实验，找到1.5T服务器上一个蜜罐”uk38“

```
#docker ps -a | grep uk38
c5233206d62a        535fbb1428e9           "/run.sh"                46 hours ago        Up 46 hours                                   k8s_ubuntu4_uk38_default_8d5111ee-c4f9-11ea-81f9-0022198f8cc6_0
43a4149edeee        k8s.gcr.io/pause:3.1   "/pause"                 46 hours ago        Up 46 hours                                   k8s_POD_uk38_default_8d5111ee-c4f9-11ea-81f9-0022198f8cc6_0
```

由于k8s的基础知识，**pause容器的最主要的作用：创建共享的网络名称空间，以便于其它容器以平等的关系加入此网络名称空间**，pause容器为每个业务容器提供了网络命名空间，接下来我们找到这个内部网络设备的网络信息。

> https://blog.51cto.com/liuzhengwei521/2422120

```
root@user-NF8480M5:~# ifconfig | grep 43a41
veth43a4149e Link encap:以太网  硬件地址 72:af:ee:64:ac:f2  
```

这个硬件地址将是第一个有用的网络信息，对于二层交换而言，内部mac信息很重要，它作为一个port存在于ovs网桥上，为后面的联通起作用，并且我们找到它作为veth pair的id(**ovs-vsctl list interface veth43a4149e**也可以)。

```
root@user-NF8480M5:~# ovs-vsctl show | grep 43a41
        Port "veth43a4149e"
            Interface "veth43a4149e"
root@user-NF8480M5:~# ovs-ofctl show fptest-br | grep veth43a41
 73(veth43a4149e): addr:72:af:ee:64:ac:f2
```

接下来我们需要获取蜜罐的网络信息，包括其ip和mac地址，我们进入创建的”run.sh"容器查看网络信息：

```
root@user-NF8480M5:~# docker exec -it c5233206d62a /bin/bash
root@uk38:/# ifconfig
eth0      Link encap:Ethernet  HWaddr c2:51:d5:f5:d4:70  
          inet addr:10.244.18.2  Bcast:10.244.18.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:241945 errors:0 dropped:1294 overruns:0 frame:0
          TX packets:179 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:29734927 (29.7 MB)  TX bytes:13495 (13.4 KB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

选取实验室实验用的浮动IP之一，比如202.117.54.233，实验demo中可以随机选取一个不与网络设备重复的mac信息。

```
root@user-NF8480M5:~# ifconfig ens3f1
ens3f1    Link encap:以太网  硬件地址 80:61:5f:02:cb:81  
          inet6 地址: fe80::8261:5fff:fe02:cb81/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  跃点数:1
          接收数据包:211035646 错误:0 丢弃:39661 过载:299557 帧数:0
          发送数据包:20132827096 错误:0 丢弃:0 过载:0 载波:0
          碰撞:0 发送队列长度:1000 
          接收字节:93958556419 (93.9 GB)  发送字节:8660115688793 (8.6 TB)
          Memory:94080000-940fffff 
# ovs-ofctl show fptest-br
1102(ens3f1): addr:80:61:5f:02:cb:81
     config:     0
     state:      0
     current:    1GB-FD COPPER AUTO_NEG
     advertised: 10MB-HD 10MB-FD 100MB-HD 100MB-FD 1GB-FD COPPER AUTO_NEG AUTO_PAUSE
     supported:  10MB-HD 10MB-FD 100MB-HD 100MB-FD 1GB-FD COPPER AUTO_NEG AUTO_PAUSE
     speed: 1000 Mbps now, 1000 Mbps max
```

ens3f1是一个通向外网的网卡，届时它的port id 1102将作为通外网的外部端口，可以使其在大网段互通。另外如果控制器尚且没有定义，我们使用**ovs-vsctl set-controller fptest-br tcp:127.0.0.1:6633**配置一个控制器，这样我们完成了基本网络实验网络信息的基本收集。

#### 此时先将流表规则清空

```
root@user-NF8480M5:~# ovs-ofctl del-flows fptest-br
root@user-NF8480M5:~# ovs-ofctl dump-flows fptest-br
NXST_FLOW reply (xid=0x4):
```

#### 此时选取的浮动ip(公用ip)是不能联通的

```
root@user-NF8480M5:/home/user/qyq/dragonflow/main# ping 202.117.54.233
PING 202.117.54.233 (202.117.54.233) 56(84) bytes of data.
^C
--- 202.117.54.233 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1012ms
```

已经定义了一个ryu app来控制服务器上流量的转发，代码里被send_arp函数定义，轮询对一些预先定义好的公网ip发arp包，我们启动这个ryu app如下：

```
root@user-NF8480M5:/home/user/qyq/dragonflow/main# ryu-manager ryu.app.rest_router
loading app ryu.app.rest_router
loading app ryu.controller.ofp_handler
instantiating app None of DPSet
creating context dpset
creating context wsgi
instantiating app ryu.app.rest_router of RestRouterAPI
instantiating app ryu.controller.ofp_handler of OFPHandler
Stream Server loop, port is 6633
Stream Server loop, port is 6653
(65660) wsgi starting up on http://0.0.0.0:8080
switch features ev version=0x4,msg_type=0x6,msg_len=0x20,xid=0x5bf644f0,OFPSwitchFeatures(auxiliary_id=0,capabilities=79,datapath_id=141155694201729,n_buffers=256,n_tables=254)
DPSET: register datapath <ryu.controller.controller.Datapath object at 0x7f2c0aae4890>
[RT][INFO] switch_id=000080615f02cb81: Set SW config for TTL error packet in.
[RT][INFO] switch_id=000080615f02cb81: Set ARP handling (packet in) flow [cookie=0x0]
[RT][INFO] switch_id=000080615f02cb81: Set L2 switching (normal) flow [cookie=0x0]
[RT][INFO] switch_id=000080615f02cb81: Set default route (resubmit to ingress) flow [cookie=0x0]
parse time is 0.656669139862 
[RT][INFO] switch_id=000080615f02cb81: Start cyclic routing table update.
[RT][INFO] switch_id=000080615f02cb81: Join as router.
receive, arp, 202.117.54.233 in fip_maps.keys()
receive, arp, 202.117.54.234 in fip_maps.keys()
receive, arp, 202.117.54.235 in fip_maps.keys()
receive, arp, 219.245.185.203 in fip_maps.keys()
receive, arp, 202.117.54.233 in fip_maps.keys()
receive, arp, 202.117.54.234 in fip_maps.keys()
receive, arp, 202.117.54.235 in fip_maps.keys()
```

由于代码逻辑还没实现，我们针对浮动ip和容器uk38的绑定进行实验，看看实验效果，实验脚本如下：

```sh
#!/bin/bash
pod_mac="c2:51:d5:f5:d4:70"
veth_mac="72:af:ee:64:ac:f2"
# a docker ip
pod_ip="10.244.18.2"
# a floating ip which can be used right now
fip="202.117.54.235"
# fip mac can be found in table "ports" where device_owner is "network:floatingip"
fip_mac="06:c4:b5:80:fc:24"
# a controller on openvswitch
ex_port=1102
veth_num=73

ovs-ofctl add-flow fptest-br table=45,priority=30,ip,in_port=$ex_port,nw_dst=$fip,actions=mod_dl_dst:$pod_mac,mod_dl_src:$veth_mac,Dec_TTL,mod_nw_dst=$pod_ip,resubmit\(,40\)
ovs-ofctl add-flow fptest-br table=50,priority=30,ip,in_port=$ex_port,nw_dst=$pod_ip,actions=output:$veth_num
ovs-ofctl  add-flow  fptest-br table=60,priority=30,ip,nw_src=$pod_ip,actions=mod_dl_src:$fip_mac,mod_dl_dst:38:ad:8e:df:a0:65,dec_ttl,mod_nw_src:$fip,resubmit\(,40\)
ovs-ofctl add-flow fptest-br table=50,priority=30,ip,nw_src=$fip,actions=output:$ex_port

```

先看一看实验效果，然后分析流表规则，启动脚本：

```
root@user-NF8480M5:/home/user/qyq/dragonflow/main# ./fip2.sh 
```

ryu app此时已经捕获到arp请求，产生相应规则，此时该外网已经可以ping通：

```
root@user-NF8480M5:/home/user/qyq/dragonflow/main# ping 202.117.54.235
PING 202.117.54.235 (202.117.54.235) 56(84) bytes of data.
64 bytes from 202.117.54.235: icmp_seq=1 ttl=62 time=11.7 ms
64 bytes from 202.117.54.235: icmp_seq=2 ttl=62 time=0.132 ms
64 bytes from 202.117.54.235: icmp_seq=3 ttl=62 time=0.205 ms
64 bytes from 202.117.54.235: icmp_seq=4 ttl=62 time=0.195 ms
64 bytes from 202.117.54.235: icmp_seq=5 ttl=62 time=0.114 ms
64 bytes from 202.117.54.235: icmp_seq=5 ttl=62 time=84.7 ms (DUP!)
64 bytes from 202.117.54.235: icmp_seq=6 ttl=62 time=0.165 ms
64 bytes from 202.117.54.235: icmp_seq=6 ttl=62 time=0.193 ms (DUP!)
64 bytes from 202.117.54.235: icmp_seq=7 ttl=62 time=0.155 ms
64 bytes from 202.117.54.235: icmp_seq=7 ttl=62 time=0.178 ms (DUP!)
^C
--- 202.117.54.235 ping statistics ---
7 packets transmitted, 7 received, +3 duplicates, 0% packet loss, time 6076ms
rtt min/avg/max/mdev = 0.114/9.785/84.715/25.216 ms
```

查看流表信息，已生成：

```
root@user-NF8480M5:/home/user/qyq/dragonflow/main# ovs-ofctl dump-flows fptest-br
NXST_FLOW reply (xid=0x4): flags=[more]
 cookie=0x0, duration=71.722s, table=0, n_packets=9285, n_bytes=2248077, idle_age=0, priority=10,in_port=1102 actions=resubmit(,10)
 cookie=0x0, duration=71.722s, table=0, n_packets=7, n_bytes=294, idle_age=0, priority=2,arp actions=CONTROLLER:65535
 cookie=0x0, duration=71.722s, table=0, n_packets=0, n_bytes=0, idle_age=71, priority=0 actions=NORMAL
 cookie=0x0, duration=71.722s, table=0, n_packets=11, n_bytes=1001, idle_age=4, priority=1 actions=resubmit(,20)
 cookie=0x0, duration=71.722s, table=10, n_packets=736, n_bytes=44160, idle_age=0, priority=2,arp actions=CONTROLLER:65535
 cookie=0x0, duration=71.722s, table=10, n_packets=6937, n_bytes=1923638, idle_age=0, priority=1037,ip actions=load:0x3->OXM_OF_METADATA[],resubmit(,20)
 cookie=0x0, duration=71.722s, table=10, n_packets=1612, n_bytes=280279, idle_age=0, priority=0 actions=drop
 cookie=0x0, duration=71.722s, table=20, n_packets=0, n_bytes=0, idle_age=71, priority=1037,ip,nw_src=10.244.240.40 actions=resubmit(,30)
 cookie=0x800, duration=71.722s, table=20, n_packets=0, n_bytes=0, idle_age=71, priority=1037,ip,nw_src=10.244.240.41 actions=resubmit(,30)
 cookie=0x1000, duration=71.722s, table=20, n_packets=0, n_bytes=0, idle_age=71, priority=1037,ip,nw_src=10.244.240.42 actions=resubmit(,30)
 cookie=0x1800, duration=71.722s, table=20, n_packets=0, n_bytes=0, idle_age=71, priority=1037,ip,nw_src=10.244.240.43 actions=resubmit(,30)
 cookie=0x2000, duration=71.721s, table=20, n_packets=0, n_bytes=0, idle_age=71, priority=1037,ip,nw_src=10.244.240.44 actions=resubmit(,30)
 cookie=0x2800, duration=71.721s, table=20, n_packets=0, n_bytes=0, idle_age=71, priority=1037,ip,nw_src=10.244.240.45 actions=resubmit(,30)
 cookie=0x3000, duration=71.721s, table=20, n_packets=0, n_bytes=0, idle_age=71, priority=1037,ip,nw_src=10.244.240.46 actions=resubmit(,30)
 cookie=0x3800, duration=71.721s, table=20, n_packets=0, n_bytes=0, idle_age=71, priority=1037,ip,nw_src=10.244.240.47 actions=resubmit(,30)
 cookie=0x4000, duration=71.721s, table=20, n_packets=0, n_bytes=0, idle_age=71, priority=1037,ip,nw_src=10.244.240.48 actions=resubmit(,30)
 cookie=0x4800, duration=71.721s, table=20, n_packets=0, n_bytes=0, idle_age=71, priority=1037,ip,nw_src=10.244.240.49 actions=resubmit(,30)
 cookie=0x5000, duration=71.721s, table=20, n_packets=0, n_bytes=0, idle_age=71, priority=1037,ip,nw_src=10.244.241.40 actions=resubmit(,30)
 cookie=0x5800, duration=71.721s, table=20, n_packets=0, n_bytes=0, idle_age=71, priority=1037,ip,nw_src=10.244.241.41 actions=resubmit(,30)
 cookie=0x6000, duration=71.721s, table=20, n_packets=0, n_bytes=0, idle_age=71, priority=1037,ip,nw_src=10.244.241.42 actions=resubmit(,30)
 cookie=0x6800, duration=71.721s, table=20, n_packets=0, n_bytes=0, idle_age=71, priority=1037,ip,nw_src=10.244.241.43 actions=resubmit(,30)
 ......
```

实验结束后一段时间，由于ryu app的停止，流表规则被清空。

接下来分析一下四条流表规则：

```sh
ovs-ofctl add-flow fptest-br table=45,priority=30,ip,in_port=$ex_port,nw_dst=$fip,actions=mod_dl_dst:$pod_mac,mod_dl_src:$veth_mac,Dec_TTL,mod_nw_dst=$pod_ip,resubmit\(,40\)
# 表45，匹配由veth port为1102且目标ip为202.117.54.235进入表的包，修改目的MAC为容器（蜜罐）的MAC地址，修改源MAC为容器接ovs网络设备的MAC地址，修改目的ip为容器ip，ttl自减，且resubmit到表40
ovs-ofctl add-flow fptest-br table=50,priority=30,ip,in_port=$ex_port,nw_dst=$pod_ip,actions=output:$veth_num
# 表50，匹配由veth port为1102且目标ip为容器IP进入表的包，修改由指定的作为veth pair的id转发出去
ovs-ofctl  add-flow  fptest-br table=60,priority=30,ip,nw_src=$pod_ip,actions=mod_dl_src:$fip_mac,mod_dl_dst:38:ad:8e:df:a0:65,dec_ttl,mod_nw_src:$fip,resubmit\(,40\)
# 表60，匹配源ip为容器ip进入表的包，修改源MAC为浮动ip的MAC地址，修改目的MAC为38:ad:8e:df:a0:65（实验室网关），ttl自减，修改源ip为浮动ip，且resubmit到表40
ovs-ofctl add-flow fptest-br table=50,priority=30,ip,nw_src=$fip,actions=output:$ex_port
# 表50，匹配源ip为浮动ip进入表的包，修改由指定的网卡出口转发出去
```

```
节选部分45到60的openflow流表：
cookie=0x0, duration=52.325s, table=45, n_packets=12, n_bytes=1061, idle_age=1, priority=30,ip,in_port=1102,nw_dst=202.117.54.235 actions=mod_dl_dst:c2:51:d5:f5:d4:70,mod_dl_src:72:af:ee:64:ac:f2,dec_ttl,mod_nw_dst:10.244.18.2,resubmit(,40)
 cookie=0x0, duration=71.727s, table=45, n_packets=6925, n_bytes=1922577, idle_age=0, priority=0 actions=drop
 cookie=0x0, duration=52.320s, table=50, n_packets=12, n_bytes=1061, idle_age=1, priority=30,ip,in_port=1102,nw_dst=10.244.18.2 actions=output:73
 cookie=0x0, duration=52.296s, table=50, n_packets=11, n_bytes=1001, idle_age=4, priority=30,ip,nw_src=202.117.54.235 actions=output:1102
 cookie=0x0, duration=71.727s, table=50, n_packets=0, n_bytes=0, idle_age=71, priority=0 actions=NORMAL
 cookie=0x0, duration=52.303s, table=60, n_packets=11, n_bytes=1001, idle_age=4, priority=30,ip,nw_src=10.244.18.2 actions=mod_dl_src:06:c4:b5:80:fc:24,mod_dl_dst:38:ad:8e:df:a0:65,dec_ttl,mod_nw_src:202.117.54.235,resubmit(,40)
 cookie=0x0, duration=71.727s, table=60, n_packets=0, n_bytes=0, idle_age=71, priority=0 actions=drop
```

缺少内部网络流向的包，这样的包都被转走了：

*cookie=0x0, duration=52.303s, table=60, n_packets=11, n_bytes=1001, idle_age=4, priority=30,ip,nw_src=10.244.18.2 actions=mod_dl_src:**06:c4:b5:80:fc:24**,mod_dl_dst:**38:ad:8e:df:a0:65**,dec_ttl,mod_nw_src:**202.117.54.235**,resubmit(,40)*



ex_port=$1
fip=$2
veth_mac=$3
pod_mac=$4
veth_num=$5
pod_ip=$6

ovs-ofctl add-flow fptest-br table=45,priority=30,ip,in_port=$ex_port,nw_dst=$fip,actions=mod_dl_dst:$pod_mac,mod_dl_src:$veth_mac,Dec_TTL,mod_nw_dst=$pod_ip,resubmit\(,40\)
ovs-ofctl add-flow fptest-br table=50,priority=30,ip,in_port=$ex_port,nw_dst=$pod_ip,actions=output:$veth_num
ovs-ofctl add-flow fptest-br table=60,priority=30,ip,nw_src=$pod_ip,actions=mod_dl_src:06:c4:b5:80:fc:24,mod_dl_dst:38:ad:8e:df:a0:65,dec_ttl,mod_nw_src:$fip,resubmit\(,40\)
ovs-ofctl add-flow fptest-br table=50,priority=30,ip,nw_src=$fip,actions=output:$ex_port

ovs-ofctl add-flow fptest-br table=50,priority=30,ip,in_port=$ex_port

