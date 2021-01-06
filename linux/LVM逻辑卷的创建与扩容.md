# LVM介绍

## LVM简介

全称逻辑卷管理器(Logic Volume Manager)。是在内核中块设备和物理设备之间添加的一个新的抽象层次。通过LVM，可以将几块磁盘（物理卷PV）组合形成一个存储池或卷组(VG)，最终在卷组的基础上再划分逻辑卷。
LVM管理着所有物理卷的物理盘区，维持着逻辑盘区和物理盘区之间的映射。LVM逻辑设备向上层应用提供了和物理磁盘相同的功能，如文件系统的创建和数据的访问等。但LVM逻辑设备不受物理约束的限制，逻辑卷不必是连续的空间，它可以跨越许多物理卷，并且可以在任何时候任意的调整大小。相比物理磁盘来说，更易于磁盘空间的管理。可以通过下图帮助理解。

![image-20200930151208992](https://gitee.com/suhaow/note/raw/master/img/20200930151237.png)

## LVM优缺点

### 优点

- 在系统运行的状态下动态的扩展/减少文件系统的大小
- 文件系统可跨多个磁盘，不受物理磁盘的限制
- 当需要导出整个卷组到另外一台机器更加方便快捷
- 可通过磁盘镜像的方式同步备份重要的数据到其他物理磁盘

### 缺点

- 当卷组中的一个磁盘损坏时，整个卷组都会收到影响
- 当从卷组中移除一个磁盘时必须使用`vgreduce`命令

## 基础概念

### 物理存储介质

指的是系统的存储设置，如`/dev/sda`等，是存储系统最低层的存储单元

### 物理卷

简称`PV(Physical Volume)`。物理卷是指硬盘分区或逻辑上与磁盘分区具有同样同能的设备，是`LVM`的基本存储逻辑块，但和基本的物理存储介绍比较，却包含有与`LVM`相关的管理参数

### 卷组

简称`VG(Volume Group)`。`LVM`卷组类似于非`LVM`系统中的物理硬盘，可以在卷组上创建一个或多个`LVM`分区(逻辑卷)。`LVM`卷组由一个或多个物理卷组成，可以通过增加卷组中的物理卷实现扩容

### 逻辑卷

简称 `LV(Logical Volume)`。`LVM`逻辑卷类似于非`LVM`系统中的硬盘分区，在逻辑卷之上可以建立文件系统

### 物理块

简称PE(Physical Extent)。每一个物理卷被划分为称为PE的基本单元，具有唯一编号的PE是可以被`LVM`寻址的最小单元，PE的大小是可配置的，默认为`4MB`

### 逻辑块

简称`lE(Logical Extent)`。逻辑卷被划分为的最小可寻址的基本单位称为LE。在同一个卷组中，LE的大小和PE是相同的， 并且一一对应。



## 命令汇总

| 功能           | 物理卷管理PV | 卷组管理VG | 逻辑卷管理LV |
| -------------- | ------------ | ---------- | ------------ |
| 扫描 (Scan)    | pvscan       | vgscan     | lvscan       |
| 建立 (Create)  | pvcreate     | vgcreate   | lvcreate     |
| 显示 (Display) | pvdisplay    | vgdisplay  | lvdisplay    |
| 删除 (Remove)  | pvremove     | vgremove   | lvremove     |
| 扩展 (Extend)  | -            | vgextend   | lvextend     |
| 减少 (Reduce)  | -            | vgreduce   | lvreduce     |

---

# LVM的创建与挂载

按照如下步骤操作即可，若有命令不清楚自行`Google`

## 查看分区

```bash
[root@suhw ~]# lsblk 
NAME            MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda               8:0    0  80G  0 disk 
├─sda1            8:1    0   1G  0 part /boot
└─sda2            8:2    0  19G  0 part 
  ├─centos-root 253:0    0  17G  0 lvm  /
  └─centos-swap 253:1    0   2G  0 lvm  [SWAP]
```

通过`lsblk`发现我们目前有两个分区`sda1`和`sda2`，其中`sda1`挂载到了`/boot`目录，而`sda2`分区下则拥有两个逻辑卷，接下来我们新增一个分区开始操作

## 新建分区

```bash
[root@suhw ~]# fdisk /dev/sda
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Command (m for help): m
Command action
   a   toggle a bootable flag
   b   edit bsd disklabel
   c   toggle the dos compatibility flag
   d   delete a partition
   g   create a new empty GPT partition table
   G   create an IRIX (SGI) partition table
   l   list known partition types
   m   print this menu
   n   add a new partition
   o   create a new empty DOS partition table
   p   print the partition table
   q   quit without saving changes
   s   create a new empty Sun disklabel
   t   change a partition's system id
   u   change display/entry units
   v   verify the partition table
   w   write table to disk and exit
   x   extra functionality (experts only)

Command (m for help): n
Partition type:
   p   primary (2 primary, 0 extended, 2 free)
   e   extended
Select (default p): p
Partition number (3,4, default 3): 3
# 指定第一个分区，使用默认值 直接回车
First sector (41943040-167772159, default 41943040): 
Using default value 41943040
# 指定最后一个分区，使用+size{K,M,G} 的形式让fdisk 自己算
Last sector, +sectors or +size{K,M,G} (41943040-167772159, default 167772159): +4G
Partition 3 of type Linux and of size 4 GiB is set

```

通过`fdisk`中的`n`可进行新增分区的操作，最终就新建了大小为4G的`/dev/sda3`分区，接下来将分区格式设置为 LVM。



## 设置LVM分区格式

```bash
# t 对应操作为 change a partition's system id
Command (m for help): t
# 分区序号，对刚新建的 sda3 进行操作
Partition number (1-3, default 3): 3

# 8e 对应的就是 Linux LVM， 可敲入 L 显示所有的system id
Hex code (type L to list all codes): 8e
Changed type of partition 'Linux' to 'Linux LVM'

Command (m for help): p

Disk /dev/sda: 85.9 GB, 85899345920 bytes, 167772160 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000d8731

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     2099199     1048576   83  Linux
/dev/sda2         2099200    41943039    19921920   8e  Linux LVM
/dev/sda3        41943040    50331647     4194304   8e  Linux LVM

```

通过`p`打印出的结果可看出此时我们新建的`/dev/sda3`分区已经设置为了`Linux LVM`格式



## 保存结果

```bash
Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.

WARNING: Re-reading the partition table failed with error 16: Device or resource busy.
The kernel still uses the old table. The new table will be used at
the next reboot or after you run partprobe(8) or kpartx(8)
Syncing disks.
```

`w`为保存退出，若想要放弃修改，敲入`q`即可

## 同步系统分区信息

在上述的修改完后，内核仍然使用的是旧的分区表，若想要通知内核更新分区信息，只有当重启后 或 运行 `partprobe` 或 `kpartx` 命令才可以。

```bash
[root@suhw ~]# partprobe 
```

更新完成后，通过`lsblk`查看就会多出一个 4G的`sda3`分区

```bash
[root@suhw ~]# lsblk 
NAME            MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda               8:0    0  80G  0 disk 
├─sda1            8:1    0   1G  0 part /boot
├─sda2            8:2    0  19G  0 part 
│ ├─centos-root 253:0    0  17G  0 lvm  /
│ └─centos-swap 253:1    0   2G  0 lvm  [SWAP]
└─sda3            8:3    0   4G  0 part 

```

此时我们的物理磁盘分区设置完毕，接下来就是和`LVM`相关的操作

## 新建物理卷PV

将刚才新建的磁盘分区创建为物理卷

```bash
[root@suhw ~]# pvcreate /dev/sda3
  Physical volume "/dev/sda3" successfully created.
  
# 查看物理卷信息
[root@suhw ~]# pvdisplay 
  --- Physical volume ---
  PV Name               /dev/sda2
  VG Name               centos
  PV Size               <19.00 GiB / not usable 3.00 MiB
  Allocatable           yes (but full)
  PE Size               4.00 MiB
  Total PE              4863
  Free PE               0
  Allocated PE          4863
  PV UUID               eJxxnr-dqUB-oZlr-j6Hr-MUWl-oJs6-JUBALe
   
  "/dev/sda3" is a new physical volume of "4.00 GiB"
  --- NEW Physical volume ---
  PV Name               /dev/sda3
  VG Name               
  PV Size               4.00 GiB
  Allocatable           NO
  PE Size               0   
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               1Xed3b-fkNW-utT5-xY7b-26e2-6soa-VfGEOP

```

## 新建卷组VG

命令格式

```
vgcreate VG_new PV ...
```

新建名为`suhwvg`的卷组，并将物理卷`/dev/sda3` 加入到` suhwvg `中

```bash
[root@suhw ~]# vgcreate suhwvg /dev/sda3
  Volume group "suhwvg" successfully created
```

查看结果创建成功

```bash
[root@suhw ~]# vgscan
  Reading volume groups from cache.
  Found volume group "centos" using metadata type lvm2
  Found volume group "suhwvg" using metadata type lvm2
```

查看卷组VG的详细信息

```bash
  [root@suhw ~]# vgdisplay 

  --- Volume group ---
  VG Name               suhwvg
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  1
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                0
  Open LV               0
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <4.00 GiB
  PE Size               4.00 MiB
  Total PE              1023
  Alloc PE / Size       0 / 0   
  Free  PE / Size       1023 / <4.00 GiB
  VG UUID               mVeVDv-UwuJ-pzMG-KEBb-95gW-YoKh-aww8uX

```

至此`PV`和`VG`都已准备就绪，在VG基础上划分逻辑卷即可

## 新建逻辑卷

在` suhwvg `卷组中新增一个大小为2G的名为 `suhwlv `的逻辑卷

```bash
[root@suhw ~]# lvcreate -L 2G -n suhwlv suhwvg
WARNING: xfs signature detected on /dev/suhwvg/suhwlv at offset 0. Wipe it? [y/n]: y
  Wiping xfs signature on /dev/suhwvg/suhwlv.
  Logical volume "suhwlv" created.
```

查看逻辑卷详情

```bash
  [root@suhw ~]# lvdisplay 
  
  --- Logical volume ---
  LV Path                /dev/suhwvg/suhwlv
  LV Name                suhwlv
  VG Name                suhwvg
  LV UUID                j8VX5e-fltA-MWVx-CGrU-BuDs-w4c1-96pC79
  LV Write Access        read/write
  LV Creation host, time suhw, 2020-09-30 10:58:46 +0800
  LV Status              available
  # open                 0
  LV Size                2.00 GiB
  Current LE             512
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:2

```

## 格式化逻辑卷

逻辑卷新建完成后，接下来将刚才新增的`suhwlv`逻辑卷格式化为`xfs`格式

```bash
[root@suhw ~]# mkfs.xfs /dev/suhwvg/suhwlv 
meta-data=/dev/suhwvg/suhwlv     isize=512    agcount=4, agsize=131072 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=524288, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

```

## 设置永久挂载

新建一个目录

```bash
[root@suhw ~]# mkdir /home/suhw
```

编辑 `/etc/fstab` 文件，末尾新增一行

```bash
/dev/suhwvg/suhwlv      /home/suhw      xfs     defaults        0 0
```

每一列分别代表的含义如下：

1. 需要挂载的设备名
2. 文件系统的挂载点
3. 挂载文件系统类型
4. 挂载所需参数，一般使用default
5. 文件系统是否需要dump
6. 是否需要开机进行`fsck` 检查（一般根分区设置为1，/boot分区设置为2， 0代表不需要）

使用`mount`命令进行挂载，`-a`参数即可根据`/etc/fstab`中的配置进行挂载

```bash
[root@suhw ~]# mount -a
```

## 查看挂载结果

```bash
[root@suhw ~]# df -hT
Filesystem                Type      Size  Used Avail Use% Mounted on
/dev/mapper/centos-root   xfs        17G 1013M   16G   6% /
devtmpfs                  devtmpfs  3.9G     0  3.9G   0% /dev
tmpfs                     tmpfs     3.9G     0  3.9G   0% /dev/shm
tmpfs                     tmpfs     3.9G   41M  3.8G   2% /run
tmpfs                     tmpfs     3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sda1                 xfs      1014M  145M  870M  15% /boot
tmpfs                     tmpfs     783M     0  783M   0% /run/user/0
/dev/mapper/suhwvg-suhwlv xfs       2.0G   33M  2.0G   2% /home/suhw

```

可以看出新建的逻辑卷的文件系统格式为`xfs`，大小为2G，并且挂载到了`home/suhw`。至此挂载成功，后续我们可以直接对该逻辑卷进行扩容/减少等操作。



---

# LVM的扩容

我们使用的某个文件系统空间不足时，扩容是常有的事，接下来将之前的逻辑卷扩容到8G。

首先需要检查卷组可用空间是否够我们对逻辑卷进行扩容操作，够的话直接使用`lvextend`即可，不够还需增大卷组中的可用空间后再使用`lvextend`对逻辑卷扩容。整体流程大致如下：

![image-20200930150202544](https://gitee.com/suhaow/note/raw/master/img/20200930150213.png)

## 获取文件系统信息

```bash
[root@suhw ~]# lsblk 
NAME              MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda                 8:0    0  80G  0 disk 
├─sda1              8:1    0   1G  0 part /boot
├─sda2              8:2    0  19G  0 part 
│ ├─centos-root   253:0    0  17G  0 lvm  /
│ └─centos-swap   253:1    0   2G  0 lvm  [SWAP]
└─sda3              8:3    0   4G  0 part 
  └─suhwvg-suhwlv 253:2    0   2G  0 lvm  /home/suhw
```

我们要调整的逻辑卷为` suhwvg-suhwlv`

## 获取逻辑卷信息

```bash
[root@suhw ~]# lvdisplay 
--- Logical volume ---
  LV Path                /dev/suhwvg/suhwlv
  LV Name                suhwlv
  VG Name                suhwvg
  LV UUID                j8VX5e-fltA-MWVx-CGrU-BuDs-w4c1-96pC79
  LV Write Access        read/write
  LV Creation host, time suhw, 2020-09-30 10:58:46 +0800
  LV Status              available
  # open                 1
  LV Size                2.00 GiB
  Current LE             512
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:2

```

## 获取卷组信息

```bash
[root@suhw ~]# vgdisplay 
--- Volume group ---
  VG Name               suhwvg
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  2
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               1
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <4.00 GiB
  PE Size               4.00 MiB
  Total PE              1023
  Alloc PE / Size       512 / 2.00 GiB
  Free  PE / Size       511 / <2.00 GiB #一个PE默认4M
  VG UUID               mVeVDv-UwuJ-pzMG-KEBb-95gW-YoKh-aww8uX

```

## 扩容卷组

卷组可用大小为`511*4=2044M`，我们需要4G 显然不够，所以再添加一个分区到该卷组中

```bash
[root@suhw ~]# fdisk /dev/sda
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): n
Partition type:
   p   primary (3 primary, 0 extended, 1 free)
   e   extended
Select (default e): p
Selected partition 4
First sector (50331648-167772159, default 50331648): 
Using default value 50331648
Last sector, +sectors or +size{K,M,G} (50331648-167772159, default 167772159): +8G
Partition 4 of type Linux and of size 8 GiB is set

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.

WARNING: Re-reading the partition table failed with error 16: Device or resource busy.
The kernel still uses the old table. The new table will be used at
the next reboot or after you run partprobe(8) or kpartx(8)
Syncing disks.
[root@suhw ~]# partprobe 
```

确认磁盘分区新建成功后，新建对应的物理卷，并加入到对应的卷组中

```bash
[root@suhw ~]# lsblk 
NAME              MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda                 8:0    0  80G  0 disk 
├─sda1              8:1    0   1G  0 part /boot
├─sda2              8:2    0  19G  0 part 
│ ├─centos-root   253:0    0  17G  0 lvm  /
│ └─centos-swap   253:1    0   2G  0 lvm  [SWAP]
├─sda3              8:3    0   4G  0 part 
│ └─suhwvg-suhwlv 253:2    0   2G  0 lvm  /home/suhw
└─sda4              8:4    0   8G  0 part 
[root@suhw ~]# pvcreate /dev/sda4
  Physical volume "/dev/sda4" successfully created.

[root@suhw ~]# vgextend suhwvg /dev/sda4
  Volume group "suhwvg" successfully extended
```

加到 `suhwvg` 卷组后， 再次查看卷组详情就会发现可用空间已经增加

```bash
 [root@suhw ~]# vgdisplay 

 --- Volume group ---
  VG Name               suhwvg
  System ID             
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  3
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               1
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               11.99 GiB
  PE Size               4.00 MiB
  Total PE              3070
  Alloc PE / Size       512 / 2.00 GiB
  Free  PE / Size       2558 / 9.99 GiB
  VG UUID               mVeVDv-UwuJ-pzMG-KEBb-95gW-YoKh-aww8uX

```

## 逻辑卷扩容

此时卷组中可用空间9.9G，够我们再扩4G，使用`lvextennd`操作即可。

注：`-r` 直接自动扩展文件系统大小

```bash
[root@suhw ~]# lvextend -L +4G /dev/suhwvg/suhwlv -r
  Size of logical volume suhwvg/suhwlv changed from 2.00 GiB (512 extents) to 6.00 GiB (1536 extents).
  Logical volume suhwvg/suhwlv successfully resized.
meta-data=/dev/mapper/suhwvg-suhwlv isize=512    agcount=4, agsize=131072 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=524288, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 524288 to 1572864

```

## 查看结果

```bash
[root@suhw ~]# df -hT
Filesystem                Type      Size  Used Avail Use% Mounted on
/dev/mapper/centos-root   xfs        19G  1.1G   18G   6% /
devtmpfs                  devtmpfs  4.1G     0  4.1G   0% /dev
tmpfs                     tmpfs     4.2G     0  4.2G   0% /dev/shm
tmpfs                     tmpfs     4.2G   43M  4.1G   2% /run
tmpfs                     tmpfs     4.2G     0  4.2G   0% /sys/fs/cgroup
/dev/sda1                 xfs       1.1G  152M  912M  15% /boot
tmpfs                     tmpfs     821M     0  821M   0% /run/user/0
/dev/mapper/suhwvg-suhwlv xfs       6.5G   35M  6.4G   1% /home/suhw

```

---

# 参考

- https://www.yisu.com/zixun/3865.html
- https://wizardforcel.gitbooks.io/vbird-linux-basic-4e/content/61.html

