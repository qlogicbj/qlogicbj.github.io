---
title: 利用nvmet配合 QLogic 41000/45000系列 RNIC创建 NVME over Fabric target系统
date: 2020-09-10 15:18:24
tags:
- nvmet
- RDMA
- fastlinq
- QLogic
- Marvell

---






NVME 现在正在慢慢替代传统的SCSI 成为主流的存储协议。在网络传输层面，也有了[NVME over Fabric](https://nvmexpress.org/wp-content/uploads/NVMe-over-Fabrics-1.1-2019.10.22-Ratified.pdf) 协议规范。NVMe over Fabric支持把NVMe映射到多个Fabrics传输选项，主要包括FC、InfiniBand、RoCE v2、iWARP和TCP。

但在这众多的传输协议选择当中，谁是最适合的呢？非常有意思的是，在NVME组织的白皮书当中，直接有这样的一段话：

> For instance, inorder to transmit NVMe protocol over a distance, the ideal underlying network or fabric technology will have the following characteristics:

> ·Reliable, credit-based flow control and delivery mechanisms.This type of flow control allows the network or fabric to be self-throttling, providing a reliable connectionthat can guarantee delivery at the hardware level without the need to drop frames or packets due to congestion. **Credit-based flow control is native to Fibre Channel, InfiniBand and PCI Express®transports.**


也就是并没有把支持RDMA 技术的以太网放在NVME over Fabric的理想网络协议当中，虽然以太网这边也是借用了InfiniBand 网络的RDMA， 技术但是对于以太网来讲，流控真的是一个难点，需要交换机和网卡双方面的配合以及大量的复杂的配置过程。虽然在大规模的部署当中，还是存在一些问题，但是简单的小规模的验证测试，则是非常方便。



## 网卡硬件
Marvell 的以太网网卡目前有E3 和E4 两个系列。 E3主要源自当年QLogic 收购的Broadcom的10G 网卡芯片，即57810, 57840系列。 E4则是QLogic后面发布的基于45000（100G能力）和41000（50G能力）两个系列。 QLogic后面被Marvell最终收购，成为Marvell产品线的一员。目前常见的有QL41132 （2x10G），QL41212（2x25G），QL45611（1x100G) 等网卡。 这里所有的E4 系列都可以使用下面的方法进行nvmet的设置。



## 软件准备
> CentOS 7.7 (Kernel 3.10.0-1062)

> Kernel Inbox OFED module

> Kernel Inbox Fastlinq driver (8.37.0.20)


## Target 端配置
### 1. 安装准备
首先确认本地的块设备：

```
[root@localhost ~]# lsblk
NAME            MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda               8:0    0 279.4G  0 disk
├─sda1            8:1    0   200M  0 part /boot/efi
├─sda2            8:2    0     1G  0 part /boot
└─sda3            8:3    0 278.2G  0 part
  ├─centos-root 253:0    0    50G  0 lvm  /
  ├─centos-swap 253:1    0   7.7G  0 lvm  [SWAP]
  └─centos-home 253:2    0 220.5G  0 lvm  /home
sr0              11:0    1  1024M  0 rom
[root@localhost ~]#

```

然后看一下，目前加载了哪些模块与qedr（网卡RDMA 处理模块）相关。


```
[root@localhost ~]# lsmod |grep qedr
qedr                  105558  0
ib_core               255469  13 qedr,rdma_cm,ib_cm,iw_cm,rpcrdma,ib_srp,ib_iser,ib_srpt,ib_umad,ib_uverbs,rdma_ucm,ib_ipoib,ib_isert
qede                  134828  1 qedr
qed                   645293  2 qede,qedr
[root@localhost ~]#
[root@localhost ~]#
```


然后加载nvmet 和nvmet-rdma 模块。

```
[root@localhost ~]# modprobe nvmet
[root@localhost ~]# modprobe nvmet-rdma
[root@localhost ~]#
[root@localhost ~]#
[root@localhost ~]# lsmod |grep nvme
nvmet_rdma             27231  0
nvmet                  51395  1 nvmet_rdma
rdma_cm                59991  7 rpcrdma,ib_srp,nvmet_rdma,ib_iser,ib_srpt,rdma_ucm,ib_isert
ib_core               255469  14 qedr,rdma_cm,ib_cm,iw_cm,rpcrdma,ib_srp,nvmet_rdma,ib_iser,ib_srpt,ib_umad,ib_uverbs,rdma_ucm,ib_ipoib,ib_isert
nvme_fc                33640  1 qla2xxx
nvme_fabrics           20212  1 nvme_fc
nvme_core              63717  2 nvme_fabrics,nvme_fc
[root@localhost ~]#


```
<!--more-->

### 2. 创建块设备用于测试

通常有几种方式来创建nvme的块设备，一种是直接使用系统的nvme盘，另外就是创建内存盘，比如使用 null_blk 或者 brd 模块等等。我们这里使用null_blk，因为单纯测试网络传输的延迟，所以null_blk 最为方便。这个 [null_blk](https://www.kernel.org/doc/html/latest/block/null_blk.html) 的介绍，可以参考的 kernel 手册中的解释。

> The null block device (/dev/nullb*) is used for benchmarking the various block-layer implementations. It emulates a block device of X gigabytes in size. It does not execute any read/write operation, just mark them as complete in the request queue.

我们使用 “nr_devices” 模块以及 “gb” 参数来建立一个2G大小的块设备：nullb0  
 
 ```
 [root@localhost ~]# modprobe null_blk nr_devices=1 gb=2
[root@localhost ~]# lsblk
NAME            MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda               8:0    0 931.5G  0 disk
├─sda1            8:1    0   200M  0 part /boot/efi
├─sda2            8:2    0     1G  0 part /boot
└─sda3            8:3    0 930.3G  0 part
  ├─centos-root 253:0    0    50G  0 lvm  /
  ├─centos-swap 253:1    0  31.4G  0 lvm  [SWAP]
  └─centos-home 253:2    0 848.9G  0 lvm  /home
sr0              11:0    1  1024M  0 rom
nullb0          252:0    0     2G  0 disk
[root@localhost ~]#

```

### 3.创建nvmet target 磁盘参数


进入目录, 此时目录当中内容为空。
```
[root@localhost ~]# cd /sys/kernel/config/nvmet/subsystems/
[root@localhost subsystems]# ls
[root@localhost subsystems]#
```


手动创建一个目录，利用它作为target 磁盘。
```
[root@localhost subsystems]# mkdir nvme-test-1
[root@localhost subsystems]# ls
nvme-test-1
[root@localhost subsystems]#

```

进入目录后，可以发现系统会自动创建出相应的磁盘参数文件。
```
[root@localhost subsystems]# cd nvme-test-1/
[root@localhost nvme-test-1]#
[root@localhost nvme-test-1]# ls
allowed_hosts  attr_allow_any_host  attr_serial  attr_version  namespaces
[root@localhost nvme-test-1]#

```

配置相关磁盘参数，将之前创建的2G 大小null_blk 块设备映射给这个target。

```
[root@localhost nvme-test-1]# echo -n 1 > attr_allow_any_host
[root@localhost nvme-test-1]#
[root@localhost nvme-test-1]# mkdir namespaces/1
[root@localhost nvme-test-1]# echo -n /dev/nullb0 > namespaces/1/device_path
[root@localhost nvme-test-1]# echo -n 1 > namespaces/1/enable
[root@localhost nvme-test-1]#

```


### 4.配置nvmet target 网络端口参数

进入目录，此时目录内容为空。

```
[root@localhost nvme-test-1]# cd /sys/kernel/config/nvmet/ports/
[root@localhost ports]# ls
[root@localhost ports]#

```

创建端口1，系统会自动生成相应的参数文件。

```
[root@localhost ports]# mkdir 1
[root@localhost ports]# cd 1
[root@localhost 1]# ls
addr_adrfam  addr_traddr  addr_treq  addr_trsvcid  addr_trtype  param_inline_data_size  referrals  subsystems
[root@localhost 1]#
[root@localhost 1]#


```

配置网络参数
```
[root@localhost 1]# echo -n 192.168.11.100 > addr_traddr
[root@localhost 1]# echo -n rdma  > addr_trtype
[root@localhost 1]# echo -n 4420  > addr_trsvcid
[root@localhost 1]# echo -n ipv4  > addr_adrfam
[root@localhost 1]#
```

请在上面4个步骤完成之后，再创建软连接：
```
[root@localhost 1]# ln -s /sys/kernel/config/nvmet/subsystems/nvme-test-1/ subsystems/nvme-test-1
[root@localhost 1]#

 
 ```

此时，配置工作就完成了。如果一切正常，应该在dmesg 看到如下信息：

```
 nvmet: adding nsid 1 to subsystem nvme-test-1
 nvmet_rdma: enabling port 1 (192.168.11.100:4420)


```


## Initiator 端配置
### 1.安装nvme-cli 工具

 ```
[root@localhost ~]# yum install nvme-cli
…
 
Installed:
  nvme-cli.x86_64 0:1.8.1-3.el7
Complete!

```

### 2.加载相关模块


```
[root@localhost mnt]# modprobe qedr
[root@localhost mnt]# modprobe nvme-rdma
[root@localhost mnt]#
[root@localhost mnt]# lsmod |grep qedr
qedr                   92709  0
ib_core               242235  15 qedr,rdma_cm,ib_cm,iw_cm,rpcrdma,ib_srp,ib_ucm,nvme_rdma,ib_iser,ib_srpt,ib_umad,ib_uverbs,rdma_ucm,ib_ipoib,ib_isert
qede                  116373  1 qedr
qed                   622744  4 qede,qedf,qedi,qedr
[root@localhost mnt]#
```

### 3.发现远端target 设备
 
```
[root@localhost mnt]# nvme discover -t rdma -a 192.168.11.100 -s 4420

Discovery Log Number of Records 1, Generation counter 1
=====Discovery Log Entry 0======
trtype:  rdma
adrfam:  ipv4
subtype: nvme subsystem
treq:    not specified
portid:  1
trsvcid: 4420
subnqn:  nvme-test-1
traddr:  192.168.11.100
rdma_prtype: not specified
rdma_qptype: connected
rdma_cms:    rdma-cm
rdma_pkey: 0x0000
[root@localhost mnt]#

```
在这个步骤如果出现错误，请检测加载的模块（qedr,nvme-rdma...)和网络的连通性（比如双方是否可以ping 通，防火墙是否关闭等等）。


### 4.连接远端target 设备
 
```
[root@localhost mnt]# nvme connect -t rdma -n nvme-test-1 -a 192.168.11.100 -s 4420

```
此时我们就可以看见这个2G 大小的nvme盘已经连上了。

```
[root@localhost mnt]# lsblk
NAME            MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda               8:0    0 931.5G  0 disk
├─sda1            8:1    0   200M  0 part /boot/efi
├─sda2            8:2    0     1G  0 part /boot
└─sda3            8:3    0 930.3G  0 part
  ├─centos-root 253:0    0    50G  0 lvm  /
  ├─centos-swap 253:1    0  31.4G  0 lvm  [SWAP]
  └─centos-home 253:2    0 848.9G  0 lvm  /home
sr0              11:0    1  1024M  0 rom
nvme0n1         259:0    0     2G  0 disk
[root@localhost mnt]#
[root@localhost mnt]#
```

### 5.下载fio 工具进行性能测试

 
```

[root@localhost ~]# yum install fio
```

性能测试: 带宽
```
[root@localhost ~]# fio --bs=64k --numjobs=16 --iodepth=4 --loops=1 --ioengine=libaio --direct=1 --invalidate=1 --fsync_on_close=1 --randrepeat=1 --norandommap --time_based --runtime=10 --filename=/dev/nvme0n1 --name=read-phase --rw=randread  -group_reporting
```

性能测试：延迟
```
[root@localhost ~]# fio --bs=512 --numjobs=1 --iodepth=1 --loops=1 --ioengine=libaio --direct=1 --invalidate=1 --fsync_on_close=1 --randrepeat=1 --norandommap --time_based --runtime=10 --filename=/dev/nvme0n1 --name=read-phase --rw=randread  -group_reporting
 
 ```
 
 


