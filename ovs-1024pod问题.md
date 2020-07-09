#### 当前1.5T服务器，创建1024个pod

![image-20200708173450539](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200708173450539.png)

ovs-vsctl为fptest-br上的pod创建port

当前host的pod数量限制：

> **# ps aux | grep kube**
>
> /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --cgroup-driver=cgroupfs --network-plugin=cni --max-pods=1200

#### 大部分问题

```
RunPodSandbox from runtime service failed: rpc error: code = DeadlineExceeded desc = context deadline exceeded
CreatePodSandbox for pod "uk1028_default(80c6391c-c0fd-11ea-81f9-0022198f8cc6)" failed: rpc error: code = DeadlineExceeded desc = context deadline exceeded
Error syncing pod 80c6391c-c0fd-11ea-81f9-0022198f8cc6 ("uk1028_default(80c6391c-c0fd-11ea-81f9-0022198f8cc6)"), skipping: failed to "CreatePodSandbox" for "uk1028_default(80c6391c-c0fd-11ea-81f9-0022198f8cc6)" with CreatePodSandboxError: "CreatePodSandbox for pod \"uk1028_default(80c6391c-c0fd-11ea-81f9-0022198f8cc6)\" failed: rpc error: code = DeadlineExceeded desc = context deadline exceeded"
```

> 类似关于kubelet的问题
>
> https://stackoverflow.com/questions/54150235/pod-from-statefulset-stuck-in-containercreating-state-failedcreatepodsandbox

容器里只有pause的，没有run.sh的。

![image-20200708174216190](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200708174216190.png)

应当正常分配add-port10次才正常，此时kubelet日志中只有三次add-port操作，有共1024个pod

#### ovs-vsctl add-port应该做什么

> https://blog.csdn.net/vonzhoufz/article/details/19981911

ovsctl这个应用程序主要职责是根据用户的命令和ovsdb沟通，**将配置信息更新到数据库中，而vswitchd会在需要重新配置的时候和ovsdb打交道，而后和内核datapath通信执行真正的动作**（通过netlink传递）。这里规定了命令的语法格式（vsctl_command_syntax ）以及所支持的所有命令，这里主要看add-port相关的。OVS_VPORT_CMD_NEW到达内核后的处理流程如下。

![img](https://img-blog.csdn.net/20140226183542578?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdm9uemhvdWZ6/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

> https://ask.openstack.org/en/question/123218/how-many-ports-are-there-can-be-created-in-an-ovs-switch/



### 1.5T运行1100个pod最后实验：

![image-20200708222352098](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200708222352098.png)

> https://stackoverflow.com/questions/32496080/how-many-virtual-interfaces-can-be-added-to-default-lxcbr0-bridge-provided-by-lx

作为二层网络交换机，**在linux brigde上，ports有限制数目，为1024，而openvswitch是没有的，**下午在验证ports数量的时候，应该可以发现绰绰有余。

![image-20200708223852626](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200708223852626.png)

所以创建到1024时，IP数目应该是够用的，实验前cni对ip池的分配为64*16(2-17)共1024个，如果1024个没有成功创建，是ip池中部分地址与容器的绑定出现问题，部分ip没有让出自己的被使用权。

实验前调研认定，无法创建1100及以上的pod并不是kubelet和ovs的ports(vethxxxxxxx)不够的问题，此时实验环境已经达到ip池的上限，此时拓宽IP池容量。

设置/etc/cni/net.d/10-aaa.conflist

```
root@user-NF8480M5:/etc/cni/net.d# vi 10-aaa.conflist
{
  "name": "ovscni",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "type": "ovsbridge",
        "bridge": "fptest-br",
        "hairpinMode": true,
        "isDefaultGateway": true,
        "ip_start": "2",
        "ip_end": "20",
        "log_path": "/tmp/ovsbridge.log",
        "debug": true
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}

```

当前ip池地址数量为64*19=1216，最大pod容量是1200。

> 打开/var/lib/cni/networks/ovscni
>
> cni ipam host-local
>
> https://cloud.tencent.com/developer/article/1470997
>
> https://www.cnblogs.com/YaoDD/p/6418785.html

由于新建了在1024个pod的基础上新建了pod，ip地址的扩展从

10.244.64.x开始，由原来的2-17每个网段扩展2个直到所有pod可以分配到ip地址。

![image-20200709110216138](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200709110216138.png)

可以看到，fptest-br开始正常添加port，ovs虚拟网络设备资源无问题，只要cni拥有足够地址即可使用。

> kubectl get pod -o wide | grep Running | grep uk | wc -l

查看通过kubelet原生创建的前缀为“uk"且成功运行的pods，为1100个。

![image-20200709110555017](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200709110555017.png)