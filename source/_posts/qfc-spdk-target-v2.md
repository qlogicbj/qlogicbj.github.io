---
title: 利用SPDK配合QLogic HBA创建FC target系统
date: 2020-09-08 13:09:00
tags:
- SPDK
- FC
- target
- QLogic
---




SPDK是最近比较热门的存储软件平台，用户态和轮询机制可以最大限度的保证驱动的性能。QLogic的HBA卡是目前市场上主流的FC控制器，服务器端的市场份额多年超过50%，而它在存储侧的市场份额则压倒性的领先。

对于最新的SPDK平台，QLogic也在2018年放出了它的代码，而且最大的特点是同时支持传统的FCP和最新的FC-NVMe协议。 本文介绍了使用SPDK 配合QLE26xx 16G HBA 和QLE27xx 32G HBA, 搭建存储系统的方法。

## 软件准备
> * [SPDK 20.01.1](https://github.com/spdk/spdk/archive/v20.01.1.tar.gz)
> * [DPDK 19.11](http://fast.dpdk.org/rel/dpdk-19.11.tar.gz)
> * QFC 4.5.3a

### 1. 安装准备
首先解压qfc 压缩包，解开后有3个目录，分别为qfc-spdk, qfc 和 umq 。暂时不动它们，稍后进行操作。

解压SPDK的压缩包，进入其中的spdk-20.01.1/app目录。将刚才的解出的 qfc-spdk 目录拷贝至此。 进入spdk-20.01.1/lib目录，将刚才解出的qfc，umq目录拷贝至此。


```
     cd spdk-20.01.1/
     cd app/
     cp -R /home/qfc-spdk-4.5.3a/qfc-spdk/ .
   
     cd spdk-20.01.1/lib
     cp -R /home/qfc-spdk-4.5.3a/qfc .
     cp -R /home/qfc-spdk-4.5.3a/umq/ .


```


然后回到spdk-20.01.1目录，将DPDK压缩包解压。

```
 tar zxvf /home/dpdk-19.11.tar.gz

```
<!--more-->

### 2. 编译DPDK代码
打补丁
```
#cd spdk-20.01.1
#patch -p1 < app/qfc-spdk/spdk-20.01.1.patch
```
这段代码补丁的主要作用是：
* 把qfc-spdk 驱动加入spdk 平台
* 把QLogic HBA卡的PCI IDs 加入到uio_pci_generic 驱动

然后修改 spdk-20.01.1/CONFIG 文件
```
Set CONFIG_NVMEFC=y
```

编译DPDK，假设是使用x86_64平台：
```
# cd dpdk-19.11
# make install T=x86_64-native-linuxapp-gcc DESTDIR=. EXTRA_CFLAGS=-fPIC 

```



### 3. 编译SPDK代码
先设定qfc-spdk的工作模式，编辑 app/qfc-spdk/Makefile，将模式改为qfc_tgt：

	 APP = qfc_tgt
	#APP = qfc_app

然后编译代码
```
# cd spdk-20.01.1
# ./configure --with-dpdk=./dpdk-19.11/x86_64-native-linuxapp-gcc
# make CONFIG_QFC=y
```

这个步骤里面，如果Linux OS系统是最小化安装的，会提示缺少一些库文件，比如uuid, numactl 之类的。通过yum 或者apt-get 补上就好了。不过在CentOS 系统当中，有CUnit-devel 这个库是官方yum源没有提供，可以从 pkgs.org 上面直接下载。

### 4.设置QFC SPDK 参数
上面编译工作都完成之后，就可以启动SPDK了。启动SPDK之前，首先编辑修改一下 **qfc.conf.in** 这个配置文件，文件位于 app/qfc-spdk 目录。

里面的注释已经比较清楚了，如果是简单测试，其实修改下面几个部分即可：

```
[QFC]

  # default transport for physical port(s)
  # FCP or NVME
  Transport FCP
#这里确定采用FC还是FC-NVMe

[Malloc]
  NumberOfLuns 8
  LunSizeInMB 32
#这里设置malloc内存盘，默认8个卷，每个大小32MB, 也可以使用真实磁盘或者NVMe盘。分别在[aio]，[nvme] 部分设置。


[PortalGroup1]
  Portal FC1 21000024ff5ba06a
  Portal FC2 21000024ff5ba06b

#portalgroup 用来绑定HBA卡的WWN，如果不知道具体wwn，可以在SPDK启动之后，查看系统信息获得，然后再回来修改这里。

[InitiatorGroup4]
  InitiatorName ALL

#Initiatorgroup用来映射服务器的WWN，可以不限制，就是ALL，或者指定具体WWN。


[TargetNode1]
  TargetName disk1
  TargetAlias "Data Disk1"
  Mapping PortalGroup1 InitiatorGroup4
  LUN0 Malloc4
  LUN1 Malloc5
  LUN2 Malloc6
  LUN3 Malloc7
  QueueDepth 64

# 这里映射所有的LUN,端口，和服务器主机。必须至少有一个LUN0

```
其他不需要的部分可以直接注释掉。





### 5. 启动SPDK

首先运行脚本，注册HBA卡：
```
# cd spdk-20.01.1	
# sudo scripts/setup.sh
```

这个动作每次设备重启之后，都需要重新注册一下。

然后设置系统Huge page，可以通过DPDK 当中 usertools 目录里面的 dpdp-setup.sh 脚本来进行，需要预留至少6GB的内存huge page.

```
# cd spdk-20.01.1
# ./dpdk-19.11/usertools/dpdk-setup.sh
```

在这个菜单方式的界面，选择huge page设置，设置的时候，输入内存块的数量，每个块是2MB，所以如果需要8GB的huge page就输入4096。



最后通过命令行启动SPDK：

```
./app/qfc-spdk/qfc_tgt -c ./app/qfc-spdk/qfc.conf.in
```

如果在运行中看到如下报错：
```
umq: 0: umq_load_file_fw:6260: filestat error /lib/firmware/ql2700_fw.bin
umq: 0: umq_init_hba:3020: Initialization failed.
umq: 0: umq_ctrlr_init:1386: Unable to initialize HBA.
```

则说明在系统的 /lib/firmware 目录当中缺少firmware 文件，请把ql2700_fw.bin 文件拷贝到这个目录中。


我在第一次运行SPDK 之后，会看到这行信息：

```
umq: 3: umq_handle_aen:149: Target port Link Up: nn-0x20000024ff7799ab:pn-0x21000024ff7799ab

```

因此就知道我目前的HBA卡使用的端口，它的WWPN和WWNN号码，把WWPN 替换刚才 qfc.conf.in 文件当中 [PortalGroup1] 模块的内容，把信息改为：

```
[PortalGroup1]
#  Portal FC1 21000024ff5ba06a
#  Portal FC2 21000024ff5ba06b

  Portal FC1 21000024ff7799ab


```


然后重启spdk，即可正常启动FCP Target。 如果想测试FC-NVME，还是修改qfc.conf.in 对应模块即可。




### 6. 服务器host端设置
如果是采用FCP，现在服务器端的HBA卡应该已经可以扫描到4个32M大小的LUN了。lsblk命令应该可以看见sd[X] 的设备，或者/dev目录也能看到。如果看不见，卸载再重新加载一下qla2xxx模块。

如果是采用FC-NVMe，则是看到4个nvme设备，可以通过nvme list 命令或者lsblk，或者/dev目录查看。Linux 系统需要qla2xxx 驱动模块版本在10.0以上。如果还是8或者9，可以从QLogic网站直接下载升级安装。