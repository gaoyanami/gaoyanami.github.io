---
layout:     post
title:      openstack虚拟机集群迁移
date:       2017-09-05
author:     gaoyan
catalog:    true
tags:
    - openstack
    - cinder
    - volume

---
## 一. openstack虚拟机集群迁移

### 1. 环境介绍
现在有两个独立openstack＋ceph集群A和B，现在要把B集群的虚拟机冷迁移至A集群, 主要保证虚拟机环境在迁移后数据完整，网络环境相同

### 2. 信息收集
- A集群，配置和B集群相同网络环境
- B集群，记录一下虚拟机的详细信息，包括IP，flavor,了解虚拟机运行服务

### 3. 卷导出(B集群)
#### 3.1 找到卷id
首先确认卷id，不管是什么方式启动，boot from volume 还是boot from image 都可以找到对应在ceph里的卷信息,这里假设boot from image
那么在ceph中对应的就是虚拟机uuiid
```bash
nova show vm1
```

```bash
rbd -p vms ls | grep $UUID
95995bfc-a6c3-482f-b0ad-dfd7afdef3cd_disk
```

#### 3.2 导出卷


```bash
rbd export vms/95995bfc-a6c3-482f-b0ad-dfd7afdef3cd_disk volume1
```

### 4 卷导入(A集群)
#### 4.1 创建volume
```bash
cinder create --name vm1_volume 100G
```

#### 4.2 导入卷
```bash
rbd import volume1 --dest-pool volumes
```

#### 4.3 替换卷
```bash
rbd -p volumes rm $vm1_volume
rbd rename volumes/volume1 volumes/$vm1_volume
```


### 4.4 设置卷启动
```bash
cinder set-bootable $vm1_volume true
```

#### 4.5 启动云主机
```bash
nova boot --boot-volume 3526da21-e94c-48f0-bef9-39c779462266  --flavor 79602f32-3a52-4cfb-b35c-78d24abef809 --nic net-id=2d9cc667-629d-4ebd-aad0-e56a3b4c1264,v4-fixed-ip=10.10.10.100 vm1
```

最后选择合适的flavor,neutron,fix_id即可


## 总结

以上操作为整个大概操作流程，主要在新平台上创建一个卷，然后导入之前的卷，并且改名字替换，这样我们创建的卷就是带有之前虚拟机数据的新卷，然后在创建虚拟机就可以了
