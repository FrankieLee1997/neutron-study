## Rook

rook的安装问题，无法在集群中正常部署ceph-mon-a...等窗格存储pod。官网如下：

```
rook-ceph-mon-a-canary-84c7fc67ff-pf7t5         1/1     Running             0          14s
rook-ceph-mon-b-canary-5f7c7cfbf4-8dvcp         1/1     Running             0          8s
```

发现是块存储的问题，目前不太敢乱动磁盘：

![image-20210401162551357](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210401162551357.png)

无空闲fstype的设备块可供使用，官网文档显示，正常fstype（ext4、swap）的存储区域都有其树形父节点，最好有个单独无挂载点的文件系统。

https://blog.csdn.net/c1481118216/article/details/81147402

https://rook.github.io/docs/rook/v1.5/ceph-quickstart.html

## NFS

https://blog.csdn.net/qq_21713697/article/details/90175435

https://blog.csdn.net/aixiaoyang168/article/details/83988253

转向较简单的NFS-server，目前在192和193的/data/share目录，挂载了一个远程可共享、无权限的远程文件系统。

服务器端219.245.185.192，客户端219.245.185.193。

![image-20210401162957232](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210401162957232.png)

NFS支持直接目录挂载/静态PV+PVC挂载/**动态PV挂载（主要需要的是这个，创一个nfs目录尝试）**

![image-20210401170123915](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210401170123915.png)

k8s 默认内部 [provisioner](https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner) 支持列表中，是不支持 NFS 的，如果我们要使用该 provisioner 该怎么办呢？方案就是使用外部 provisioner，这里可参照 [kubernetes-incubator/external-storage](https://github.com/kubernetes-incubator/external-storage) 这个来创建（https://github.com/kubernetes-incubator/external-storage.git只读了，下个源码包解压）：

主要是修改/nfs-client/deploy中的deployment.yaml，修改nfs配置（server ip和挂载目录），然后直接跑class、rbac和deployment的yaml文件，默认namespace依然是default：

```
spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: 219.245.185.192
            - name: NFS_PATH
              value: /data/nfs
      volumes:
        - name: nfs-client-root
          nfs:
            server: 219.245.185.192
            path: /data/nfs
```

![image-20210401171512172](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210401171512172.png)

使用示例pvc尝试，发现可以自动生成pv并版绑定了：

![image-20210401172938619](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210401172938619.png)

![image-20210401172958953](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210401172958953.png)

![image-20210401173014702](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210401173014702.png)

尝试使用virtctl直接创建上传镜像的pvc：

成功了，它们会自动创建pv并匹配

![image-20210401173421573](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210401173421573.png)

![image-20210401173438759](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210401173438759.png)

成功完成持久化：

![image-20210401173645418](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210401173645418.png)

这样在基础版虚拟机启动流程中，可以省略最麻烦的手动创建pv，而且不会产生冗杂的xxx-scratch这个pv，过程简化为：

**上传win镜像至一个不重名pvc ->制作一个winhd的pvc ->创建虚机并启动**

## 针对虚拟硬盘的问题

#### hostdisk形式（不同path即可，节省pv定义）

```
name: harddrive
hostDisk:
  capacity: 50Gi
  path: /data/disk.img
  type: DiskOrCreate
```

winhd形式

通过本地no-provisioner形式创建一个winhd的pvc，但需要手动定义pv和hostpath。

![image-20210407144445449](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210407144445449.png)

```
root@node3:/data/winhd-0302-1# ls
disk.img
```

**上传win镜像至一个不重名pvc ->制作一个winhd的pvc ->创建虚机并启动**

->

**上传win镜像至一个不重名pvc ->使用指定目录存储img->创建虚机并启动**