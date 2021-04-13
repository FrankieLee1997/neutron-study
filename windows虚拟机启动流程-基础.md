## 普通文件挂载型，使用指定路径的目录来做磁盘挂载点，不使用k8s的provisionor来做动态规划，已创建好不限大小的storageclass：local-storage，可长期复用：

### 1.新创建两个pv（xxx和xxx-scratch），不与先前pv重复，挂载目录可以放在/data下，每次自增创建目录，目录名可以设置和pv名字一样。apply这些yaml即可。

```
apiVersion: v1
kind: PersistentVolume
metadata:
    name: win10-0303-1
    labels:
        type: local
spec:
    storageClassName: local-storage
    capacity:
        storage: 10Gi
    accessModes:
        - ReadWriteOnce
    hostPath:
        path: /data/win10-0303-1

```

```
apiVersion: v1
kind: PersistentVolume
metadata:
    name: win10-0303-1-scratch
    labels:
        type: local
spec:
    storageClassName: local-storage
    capacity:
        storage: 10Gi
    accessModes:
        - ReadWriteOnce
    hostPath:
        path: /data/win10-0303-1
```

### 2.上传指定iso，pvc设置为和不带scratch后缀的pv一样，自动绑定，pvc大小比镜像略大即可，可长期设为7-8G

virtctl image-upload   --image-path='Win10_20H2_v2_Chinese_x64.iso'   --storage-class local-storage   --pvc-name=win10-0303-1   --pvc-size=7G   --uploadproxy-url=https://10.99.99.248   --insecure   --wait-secs=240

### 3.创建硬盘winhd的pv和pvc，一般指定20G就行，挂载目录不重复。apply这些yaml即可。

```
kind: PersistentVolume
apiVersion: v1
metadata:
    name: winhd-0303-1
    labels:
        type: local
spec:
    storageClassName: local-storage
    capacity:
        storage: 20Gi
    accessModes:
        - ReadWriteOnce
    hostPath:
        path: "/data/winhd-0303-1"
```

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: winhd-0303-1
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: local-storage
```

### 4.部署windows虚拟机实例

yaml如下

```
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  name: win10-vm-0303-1
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/domain: win10
    spec:
      nodeSelector:
        node: node3
      domain:
        cpu:
          cores: 4
        devices:
          disks:
          - bootOrder: 1
            cdrom:
              bus: sata
            name: cdromiso
          - bootOrder: 2
            disk:
              bus: virtio
            name: harddrive
          - bootOrder: 3
            cdrom:
              bus: sata
            name: virtiocontainerdisk
          interfaces:
          - bridge: {}
            name: default
        machine:
          type: q35
        resources:
          requests:
            memory: 16G
      networks:
      - name: default
        pod: {}
      volumes:
      - name: cdromiso
        persistentVolumeClaim:
          claimName: win10-0303-1
      - name: harddrive
        persistentVolumeClaim:
            claimName: winhd-0303-1
      - containerDisk:
          image: kubevirt/virtio-container-disk
        name: virtiocontainerdisk
```

virtctl start/stop <虚拟机名称>，启动一个虚拟机实例。