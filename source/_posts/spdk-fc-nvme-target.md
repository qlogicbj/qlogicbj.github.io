---
title: 利用SPDK配合QLogic HBA创建FC-NVMe target系统 
date: 2021-10-27 20:01:14
tags:
- SPDK
- FC-NVMe
- target
- QLogic

---

SPDK是最近比较热门的存储软件平台，用户态和轮询机制可以最大限度的保证驱动的性能。

前面的文章已经介绍了如何利用SPDK 构建FCP的target，本文介绍一下，利用SPDK 来构建FC-NVMe的target系统。


## 软件准备
> * [SPDK 20.01.1](https://github.com/spdk/spdk/archive/v20.01.1.tar.gz)
> * [DPDK 19.11](http://fast.dpdk.org/rel/dpdk-19.11.tar.gz)
> * QFC 4.5.4d

### 1. 安装准备
首先解压qfc 压缩包，解开后有3个目录，分别为qfc-spdk, qfc 和 umq 。暂时不动它们，稍后进行操作。

解压SPDK的压缩包，进入其中的spdk-20.01.1/app目录。将刚才的解出的 qfc-spdk 目录拷贝至此。 进入spdk-20.01.1/lib目录，将刚才解出的qfc，ump目录拷贝至此。


```
     cd spdk-20.01.1/
     cd app/
     cp -R /home/qfc-spdk-4.5.4d/qfc-spdk/ .
   
     cd spdk-20.01.1/lib
     cp -R /home/qfc-spdk-4.5.4d/qfc .
     cp -R /home/qfc-spdk-4.5.4d/umq/ .


```


然后回到spdk-20.01.1目录，将DPDK压缩包解压。

```
 tar zxvf /home/dpdk-19.11.tar.gz

```


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

  # Backend application
  #BackendAppFcp spdkscsi:-1
  BackendAppNvme spdknvme:-1

  # default transport for physical port(s)
  # FCP or NVME
  Transport NVME

#这里确定采用FC还是FC-NVMe

  # Unload the kernel driver
  UnbindFromKernel Yes

  #Load fc card firmware from file /lib/firmware/ql2700_fw.bin
  LoadFWfromFile Yes
#这里确认把fw文件放到系统的/lib文件夹当中。

  # Multi Atio
  # To Enable multi atio Set MultiAtioEnable as Yes.
  # Default Multiatio will be disabled.
  # 1. NumberOfATIO (Total number of ATIO FCP + NVMe)
  # 2. NvmeConnectionID_ATIO  (Number of atio queues for NVMe, ConnID based)
  # 3. FCP Queues = NumberOfATIO - NvmeConnectionID_ATIO (Round-Robin)
  # Note:-  NumberofATIO should be less than ReactorMask
  MultiAtioEnable No
#这里设置Multi ATIO 的支持，默认情况是关闭。

  #NumberOfATIO 10
  #NvmeConnectionID_ATIO 5

   NumberOfATIOQ       8
   NumberOfNvmeATIO 4
   FCPQRouting roundrobin
   NVMeQRouting  connid

#这些采用默认设置即可。



[Malloc]
  NumberOfLuns 2
  LunSizeInMB 4096
#这里设置malloc内存盘，这里设置为2个卷，每个大小4096MB, 也可以使用真实磁盘或者NVMe盘。分别在[aio]，[nvme] 部分设置。


[Null]
   # Dev <name> <size_in_MiB> <block_size>
   # Create an 1 terabyte null bdev with 4k block size called Null0
   # For example add to a nvme Subsystem with:
   #   Namespace Null0
  Dev Null0 1048576 4096
  Dev Null1  5120 512
#如果为了测试网络延迟，也可以使用Null 来作为测试卷，这样IO落下之后，也无需处理，相比malloc 更快。



[Subsystem2]
  NQN nqn.2016-06.io.spdk:cnode2
  Listen FC nn-0x20000024ff7799ab:pn-0x21000024ff7799ab

#  Listen FC nn-0x20000124ff5ba06b:pn-0x21000124ff5ba06b
  AllowAnyHost Yes
#Host nqn.2016-06.io.spdk:init
  SN SPDK00000000000002
  Namespace Malloc0
#  Namespace Null1

[Subsystem3]
  NQN nqn.2016-06.io.spdk:cnode3
#  Listen FC nn-0x20000124ff5ba06b:pn-0x21000124ff5ba06b
  Listen FC nn-0x20000024ff7799a9:pn-0x21000024ff7799a9
  AllowAnyHost Yes
#  #Host nqn.2016-06.io.spdk:init
  SN SPDK00000000000003
  Namespace Null1
  

#创建2个Subsystem，一个采用Malloc 内存盘，另一个采用Null 卷。上面的Listen FC 参数非常重要，必须对应你要使用的FC端口的WWN。如果现在不知道端口的WWN参数，没有关系，执行一次QFC 就能够从系统打印知道结果，然后再回来修改conf文件即可。

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

最后通过命令行启动SPDK，这里注意一下，qfc4.5.4 之后采用了一个新的 -g参数：

```
 ./app/qfc-spdk/qfc_tgt -c ./app/qfc-spdk/qfc.conf.in -g

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

因此就知道我目前的HBA卡使用的端口，它的WWPN和WWNN号码，把WWPN 替换刚才 qfc.conf.in 文件当中 [Subsystem] 模块的内容，把信息改为：

```
[Subsystem2]
  NQN nqn.2016-06.io.spdk:cnode2
  Listen FC nn-0x20000024ff7799ab:pn-0x21000024ff7799ab

#  Listen FC nn-0x20000124ff5ba06b:pn-0x21000124ff5ba06b


```


然后重启spdk，即可正常启动FCP Target。 如果想测试FC-NVME，还是修改qfc.conf.in 对应模块即可。




### 6. 服务器host端设置

从10.02.xx qla2xxx驱动开始，我们在编译驱动的时候，可以增加一个参数来进行FC NVMe discovery 服务。但是前提条件是，必须安装nvme-cli 工具。

所以首先yum 安装nvme-cli：

```
# yum install nvme-cli

```

然后从[QLogic驱动下载网站](https://driverdownloads.qlogic.com/)，下载最新的10.x 的Linux驱动。目前官网最新Linux 版本是10.02.06.xx

下载源码之后，首先进行编译安装。编译安装需要系统首先安装kernel源码：
```
# yum install kernel-devel

```

然后解压驱动包：

```
# tar -xzvf qla2xxx-src-10.02.xx.yy.zz-k.tar.gz

# cd qla2xxx-src-10.02.xx.yy.zz-k

```

编译安装新驱动
```
./extras/build.sh initrd

```

这里需要介绍一下，编译安装的时候，有多种参数，可以直接使用 -h参数查看
```
./extras/build.sh -h
```
其中 **install** 参数是单纯编译，但是并不集成到启动镜像initrd image里面，这样你重启服务器之后，系统自动加载的qla2xxx 驱动版本，仍然是初始的inbox驱动版本； 而**initrd** 参数则是编译安装之后，把新驱动合并到initrd image当中，这样你重启服务器之后，系统加载的也是新的qla2xxx驱动版本了。

在安装完驱动之后，仍然使用脚本工具，执行命令参数 install_fcnvme_scripts：

```

   # ./extras/build.sh install_fcnvme_scripts

     Install udev rule for FC-NVME disovery and
     boot time auto-connection service/scripts.

```
这个参数的作用就是配合nvme-cli 工具，自动发现FC-NVMe target目标。

如果是采用FC-NVMe，则是看到2个nvme设备，可以通过nvme list 命令或者lsblk，或者/dev目录查看。

