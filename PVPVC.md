# Volume

在Docker的设计实现中，容器中的数据是临时的，即当容器被销毁时，其中的数据将会丢失。如果需要持久化数据，需要使用Docker数据卷挂载宿主机上的文件或者目录到容器中。在Kubernetes中，当Pod重建的时候，数据是会丢失的，Kubernetes也是通过数据卷挂载来提供Pod数据的持久化的。Kubernetes数据卷是对Docker数据卷的扩展，Kubernetes数据卷是Pod级别的，可以用来实现Pod中容器的文件共享。目前，Kubernetes支持的数据卷类型如下：

> 　1)    EmptyDir　        2)    HostPath　       3)    GCE Persistent Disk          4)    AWS Elastic Block Store　　5)    NFS　　　　6)    iSCSI　　　7)    Flocker　　　　8)    GlusterFS　　　　9)    RBD　　　　10)  Git Repo　　　11)  Secret　　　　12)  Persistent Volume Claim　　　13)  Downward API

## 本地数据卷 基础Pod在node上挂载数据卷

https://www.cnblogs.com/nj-duzi/p/14105180.html

EmptyDir、HostPath这两种类型的数据卷，只能最用于本地文件系统。**本地数据卷中的数据只会存在于一台机器上，所以当Pod发生迁移的时候，数据便会丢失**。该类型Volume的用途是：Pod中容器间的文件共享、共享宿主机的文件系统。

emptyDir与hostPath属于节点级别的卷类型，emptyDir的生命周期与Pod资源相同，而使用了hostPath卷的Pod一旦被重新调度至其他节点，那么它将无法在使用此前的数据。因此，这两张类型都不具有持久性。要想使用持久类型的存储卷，就得使用网络存储系统，如NFS，Ceph、GlusterFS等，或者云端存储，如awsElasticBlockStore、gcePersistentDisk。

## PV PVC（NFS/Ceph/Rook/LocalVolume）

![img](https://www.kubernetes.org.cn/img/2018/06/%E5%9B%BE%E7%89%871.png)

> PV—资源池—NFS等
>
> PVC—“分享给谁用” =>请求PV
>
> pv不是一个namespace资源 pv是跨namespace的共享对象，pvc是有namespace特征的

理解每个存储系统是一件复杂的事情，特别是对于普通用户来说，**有时候并不需要关心各种存储实现，只希望能够安全可靠地存储数据**。Kubernetes中提供了Persistent Volume和Persistent Volume Claim机制，这是存储消费模式。**Persistent Volume是由系统管理员配置创建的一个数据卷（目前支持HostPath、GCE Persistent Disk、AWS Elastic Block Store、NFS、iSCSI、GlusterFS、RBD）**，它代表了某一类存储插件实现；而对于普通用户来说，**通过Persistent Volume Claim可请求并获得合适的Persistent Volume**，而无须感知后端的存储实现。**Persistent Volume和Persistent Volume Claim的关系其实类似于Pod和Node，Pod消费Node资源，Persistent Volume Claim则消费Persistent Volume资源**。Persistent Volume和Persistent Volume Claim相互关联，有着完整的生命周期管理：

​				1)    准备：系统管理员规划或创建一批Persistent Volume；

　　　　2)    绑定：用户通过创建Persistent Volume Claim来声明存储请求，Kubernetes发现有存储请求的时候，就去查找符合条件的Persistent Volume（最小满足策略）。找到合适的就绑定上，找不到就一直处于等待状态；

　　　　3)    使用：创建Pod的时候使用Persistent Volume Claim；

　　　　4)    释放：当用户删除绑定在Persistent Volume上的Persistent Volume Claim时，Persistent Volume进入释放状态，此时Persistent Volume中还残留着上一个Persistent Volume Claim的数据，状态还不可用；

　　　　5)    回收：是否的Persistent Volume需要回收才能再次使用。回收策略可以是人工的也可以是Kubernetes自动进行清理（仅支持NFS和HostPath）

## SC

**存储类对象的名称非常重要，用户通过名称类请求特定的存储类。管理员创建存储类对象时，会设置类的名称和其它的参数，存储类的对象一旦被创建，将不能被更新。管理员能够为PVC指定一个默认的存储类。**

存储类有一个供应者的参数域，此参数域决定PV使用什么存储卷插件。参数必需进行设置：

| **存储卷**           | **内置供应者** | **配置例子**                                                 |
| -------------------- | -------------- | ------------------------------------------------------------ |
| AWSElasticBlockStore | ✓              | [AWS](https://kubernetes.io/docs/concepts/storage/storage-classes/#aws) |
| AzureFile            | ✓              | [Azure File](https://kubernetes.io/docs/concepts/storage/storage-classes/#azure-file) |
| AzureDisk            | ✓              | [Azure Disk](https://kubernetes.io/docs/concepts/storage/storage-classes/#azure-disk) |
| CephFS               | –              | –                                                            |
| Cinder               | ✓              | [OpenStack Cinder](https://kubernetes.io/docs/concepts/storage/storage-classes/#openstack-cinder) |
| FC                   | –              | –                                                            |
| FlexVolume           | –              | –                                                            |
| Flocker              | ✓              | –                                                            |
| GCEPersistentDisk    | ✓              | [GCE](https://kubernetes.io/docs/concepts/storage/storage-classes/#gce) |
| Glusterfs            | ✓              | [Glusterfs](https://kubernetes.io/docs/concepts/storage/storage-classes/#glusterfs) |
| iSCSI                | –              | –                                                            |
| PhotonPersistentDisk | ✓              | –                                                            |
| Quobyte              | ✓              | [Quobyte](https://kubernetes.io/docs/concepts/storage/storage-classes/#quobyte) |
| NFS                  | –              | –                                                            |
| RBD                  | ✓              | [Ceph RBD](https://kubernetes.io/docs/concepts/storage/storage-classes/#ceph-rbd) |
| VsphereVolume        | ✓              | [vSphere](https://kubernetes.io/docs/concepts/storage/storage-classes/#vsphere) |
| PortworxVolume       | ✓              | [Portworx Volume](https://kubernetes.io/docs/concepts/storage/storage-classes/#portworx-volume) |
| ScaleIO              | ✓              | [ScaleIO](https://kubernetes.io/docs/concepts/storage/storage-classes/#scaleio) |
| StorageOS            | ✓              | [StorageOS](https://kubernetes.io/docs/concepts/storage/storage-classes/#storageos) |
| Local                | –              | [Local](https://kubernetes.io/docs/concepts/storage/storage-classes/#local) |

**AccessMode 访问模式，指定node的挂载方式，支持ReadWriteOnce读写挂载一次，ReadOnlyMany多个节点挂载只读模式，ReadWriteMany多个节点挂载读写模式，不同的volume驱动类型支持的模式有所不同，如下**

![img](https://img-blog.csdnimg.cn/20201023172150667.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0NTU2NDE0,size_16,color_FFFFFF,t_70)



已尝试GCE，遇到了找不到节点上的PV来绑定的情况，后期重新尝试

![image-20210114223254814](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210114223254814.png)

## 继续尝试手动编写LocalVolume（no-provisioner的sc，使本地目录挂载目标vm数据卷）

> HostPath: 卷本身不带有调度信息，如果想对每个pod固定在某个节点上，就需要对pod配置nodeSelector等调度信息；
>
> LocalVolume: 卷本身包含了调度信息，使用这个卷的pod会被固定在特定的节点上，这样可以很好的保证数据的连续性。

修改local-storage的bindingmode为immediate后，绑上了第一个pvc进行消费，但出现了scratch这个pvc无法绑定合适pv的问题，describe一下：

![image-20210114224544653](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210114224544653.png)

发现只有一部分绑上了，pod也没有启动成功，查找资料，通过手动方式多创建一个符合对应scratch的pv

![image-20210117191924999](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210117191924999.png)

再次进行绑定。成功创建一个cdi对应的镜像上传时的功能支持pod

![image-20210117192003096](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210117192003096.png)

均成功了，发现pv和pvc遵循了严格的一一对应，尤其在pvc名称和storageclass名称上，且实验发现每一次删除pvc，无法直接再次使用原pvc名称进行镜像上传，必须从初始化开始重新部署pv并进行镜像上传的操作，cdi对应的pod会在上传成功后自动销毁

![image-20210117192020699](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210117192020699.png)

一段时间后，scratch对应的pv解绑，对应的pvc被清除，**不确定此时是否上传成功，而且初步判断scratch是一个辅助cdi数据卷创建的功能支持，根据cdi定义，此时剩下的pvc已经可以作为数据卷被kubevirt虚拟机使用**

![image-20210117192755743](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210117192755743.png)

