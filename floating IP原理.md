## 一期浮动IP总结：

![image-20210331153909308](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210331153909308.png)

port2出去的报文，做SNAT，从port2进来的报文做DNAT；

https://blog.csdn.net/wangjinruifly/article/details/79838515?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.control&dist_request_id=&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.control

在构造虚拟路由后，在linux的ip netns中，可以找到对应的一个qrouter并注明了其内部vlan id，其中构建了qg和qr两个接口，分别连接到外网和内网，以及一个loopback接口。qr代表着router_interface，初始用一个外部ip代表着qg，表示出external_gateway，随着浮动ip的分配，它们一一加入到qg下面，并与关联的内网网段形成映射关系。

如博客中：

```
# ip netns exec qrouter-0c5fb528-70e7-49ef-9142-8535560492d9 ip address
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: qg-ac69635d-39@if19: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether fa:16:3e:84:81:b3 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    # 这个是创建虚拟路由后指定的external_gateway，即头图的port3，帮助找到外部网关
    inet 161.55.96.104/24 brd 161.55.96.255 scope global qg-ac69635d-39
       valid_lft forever preferred_lft forever
    # 这个是新分配给instance的，指定外部网段中的一个可用ip
    inet 161.55.96.107/32 brd 161.55.96.107 scope global qg-ac69635d-39
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe84:81b3/64 scope link 
       valid_lft forever preferred_lft forever
3: qr-9a787464-ac@if20: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default qlen 1000
    link/ether fa:16:3e:7f:9d:9b brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.2.1/24 brd 192.168.2.255 scope global qr-9a787464-ac
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe7f:9d9b/64 scope link 
       valid_lft forever preferred_lft forever
```

node2中：

```
root@node2:/home/node2# ip netns exec qrouter-f3e24a2d ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: qr-ca1ce06a@if26: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether fa:16:3e:a4:d0:aa brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.244.1.1/24 brd 10.244.1.255 scope global qr-ca1ce06a
       valid_lft forever preferred_lft forever
3: qg-30dc8ca5@if27: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether fa:16:3e:eb:33:06 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 222.91.160.61/24 brd 222.91.160.255 scope global qg-30dc8ca5
       valid_lft forever preferred_lft forever
    inet 222.91.160.59/32 brd 222.91.160.59 scope global qg-30dc8ca5
       valid_lft forever preferred_lft forever
    inet 222.91.160.60/32 brd 222.91.160.60 scope global qg-30dc8ca5
       valid_lft forever preferred_lft forever
```

分配浮动ip后，qg上多了一个外部ip的信息，并可以查看iptables的更新。

```
root@node2:/home/node2# ip netns exec qrouter-f3e24a2d iptables -t nat -S
-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P OUTPUT ACCEPT
-P POSTROUTING ACCEPT
-N neutron-l3-agent-OUTPUT
-N neutron-l3-agent-POSTROUTING
-N neutron-l3-agent-PREROUTING
-N neutron-l3-agent-float-snat
-N neutron-l3-agent-snat
-N neutron-postrouting-bottom
-A PREROUTING -j neutron-l3-agent-PREROUTING
-A OUTPUT -j neutron-l3-agent-OUTPUT
-A POSTROUTING -j neutron-l3-agent-POSTROUTING
-A POSTROUTING -j neutron-postrouting-bottom
-A neutron-l3-agent-OUTPUT -d 222.91.160.59/32 -j DNAT --to-destination 10.244.1.21
-A neutron-l3-agent-OUTPUT -d 222.91.160.60/32 -j DNAT --to-destination 10.244.1.24
-A neutron-l3-agent-PREROUTING -d 222.91.160.59/32 -j DNAT --to-destination 10.244.1.21
-A neutron-l3-agent-PREROUTING -d 222.91.160.60/32 -j DNAT --to-destination 10.244.1.24
-A neutron-l3-agent-float-snat -s 10.244.1.21/32 -j SNAT --to-source 222.91.160.59
-A neutron-l3-agent-float-snat -s 10.244.1.24/32 -j SNAT --to-source 222.91.160.60
-A neutron-l3-agent-snat -j neutron-l3-agent-float-snat
-A neutron-postrouting-bottom -j neutron-l3-agent-snat
```

从219主机ping到222网关的正常通信：219到222，request包从网卡到实验室网关，回包从222网关到网卡

![image-20210401163826205](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210401163826205.png)

从222主机ping到219网关，**缺少从实验室38网关到网卡的包**，参考一期需要一个external_gateway转发：

![image-20210401163638516](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210401163638516.png)

配置指定路由表信息：

![image-20210401164132621](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210401164132621.png)