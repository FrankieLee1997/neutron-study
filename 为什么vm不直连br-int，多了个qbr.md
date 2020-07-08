## 为什么vm不直连br-int，多了个qbr  
'''
Ideally, the TAP device vnet0 would be connected directly to the integration bridge, br-int.   
Unfortunately, this isn’t possible because of how OpenStack security groups are currently implemented.   
OpenStack uses iptables rules on the TAP devices such as vnet0 to implement security groups,   
and Open vSwitch is not compatible with iptables rules that are applied directly on TAP devices that are connected to an Open vSwitch port.
'''
其实就是说，OpenvSwitch不支持现在的OpenStack的实现方式，因为OpenStack是把iptables规则丢在TAP设备中，以此实现了安全组功能。没办法，所以用了一个折衷的方式，在中间加一层，用Linux Bridge来实现吧，这样，就莫名其妙了多了一个qbr网桥。在qbr上面还存在另一个设备C，这也是一个TAP设备。C通常以qvb开头，C和br-int上的D连接在一起，形成一个连接通道，使得qbr和br-int之间顺畅通信。
