#### 当前1.5T服务器，创建1024个pod

![image-20200708173450539](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200708173450539.png)

ovs-vsctl为fptest-br上的pod创建port

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

应当正常分配add-port10次才正常，

#### ovs-vsctl add-port做了什么

> https://blog.csdn.net/vonzhoufz/article/details/19981911

ovsctl这个应用程序主要职责是根据用户的命令和ovsdb沟通，**将配置信息更新到数据库中，而vswitchd会在需要重新配置的时候和ovsdb打交道，而后和内核datapath通信执行真正的动作**（通过netlink传递）。这里规定了命令的语法格式（vsctl_command_syntax ）以及所支持的所有命令，这里主要看add-port相关的。OVS_VPORT_CMD_NEW到达内核后的处理流程如下。

![img](https://img-blog.csdn.net/20140226183542578?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdm9uemhvdWZ6/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

> https://ask.openstack.org/en/question/123218/how-many-ports-are-there-can-be-created-in-an-ovs-switch/