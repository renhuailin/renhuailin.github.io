---
layout: post
title: "配置LVM"
date: 2015-01-30 12:30:00
author: 任怀林
categories: 
- blog
- other
thumb: linux.png
isCJKLanguage: true
---

1 为虚拟机新加一块disk，用fdisk把它分成lvm的分区。

{% highlight sh %}
root@localhost:~# fdisk /dev/xvdb 
Command (m for help): m
Command action
   a   toggle a bootable flag
   b   edit bsd disklabel
   c   toggle the dos compatibility flag
   d   delete a partition
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
{% endhighlight %}

2 显示当前的分区表
{% highlight sh %}
Command (m for help): p 

Disk /dev/xvdb: 107.4 GB, 107374182400 bytes
255 heads, 63 sectors/track, 13054 cylinders, total 209715200 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0xdd2fff2e

    Device Boot      Start         End      Blocks   Id  System

{% endhighlight %}

#创建一个新的分区

{% highlight sh %}
Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended

#创建一个主分区
Select (default p): 
Using default response p

#按提示来，用默认值
Partition number (1-4, default 1): 
Using default value 1

#按提示来，用默认值
First sector (2048-209715199, default 2048): 
Using default value 2048

#这里把整个盘分成了一个区
Last sector, +sectors or +size{K,M,G} (2048-209715199, default 209715199): 
Using default value 209715199

# 把分区类型转成lvm

Command (m for help): t
Selected partition 1

#如果不记得分区的类型的代码了，可以用L命令显示分区类型列表
Hex code (type L to list codes): L

 0  Empty           24  NEC DOS         81  Minix / old Lin bf  Solaris        
 1  FAT12           27  Hidden NTFS Win 82  Linux swap / So c1  DRDOS/sec (FAT-
 2  XENIX root      39  Plan 9          83  Linux           c4  DRDOS/sec (FAT-
 3  XENIX usr       3c  PartitionMagic  84  OS/2 hidden C:  c6  DRDOS/sec (FAT-
 4  FAT16 <32M      40  Venix 80286     85  Linux extended  c7  Syrinx         
 5  Extended        41  PPC PReP Boot   86  NTFS volume set da  Non-FS data    
 6  FAT16           42  SFS             87  NTFS volume set db  CP/M / CTOS / .
 7  HPFS/NTFS/exFAT 4d  QNX4.x          88  Linux plaintext de  Dell Utility   
 8  AIX             4e  QNX4.x 2nd part 8e  Linux LVM       df  BootIt         
 9  AIX bootable    4f  QNX4.x 3rd part 93  Amoeba          e1  DOS access     
 a  OS/2 Boot Manag 50  OnTrack DM      94  Amoeba BBT      e3  DOS R/O        
 b  W95 FAT32       51  OnTrack DM6 Aux 9f  BSD/OS          e4  SpeedStor      
 c  W95 FAT32 (LBA) 52  CP/M            a0  IBM Thinkpad hi eb  BeOS fs        
 e  W95 FAT16 (LBA) 53  OnTrack DM6 Aux a5  FreeBSD         ee  GPT            
 f  W95 Ext'd (LBA) 54  OnTrackDM6      a6  OpenBSD         ef  EFI (FAT-12/16/
10  OPUS            55  EZ-Drive        a7  NeXTSTEP        f0  Linux/PA-RISC b
11  Hidden FAT12    56  Golden Bow      a8  Darwin UFS      f1  SpeedStor      
12  Compaq diagnost 5c  Priam Edisk     a9  NetBSD          f4  SpeedStor      
14  Hidden FAT16 <3 61  SpeedStor       ab  Darwin boot     f2  DOS secondary  
16  Hidden FAT16    63  GNU HURD or Sys af  HFS / HFS+      fb  VMware VMFS    
17  Hidden HPFS/NTF 64  Novell Netware  b7  BSDI fs         fc  VMware VMKCORE 
18  AST SmartSleep  65  Novell Netware  b8  BSDI swap       fd  Linux raid auto
1b  Hidden W95 FAT3 70  DiskSecure Mult bb  Boot Wizard hid fe  LANstep        
1c  Hidden W95 FAT3 75  PC/IX           be  Solaris boot    ff  BBT            
1e  Hidden W95 FAT1 80  Old Minix      
Hex code (type L to list codes): 8e
Changed system type of partition 1 to 8e (Linux LVM)

#再看一下
Command (m for help): p

Disk /dev/xvdb: 107.4 GB, 107374182400 bytes
255 heads, 63 sectors/track, 13054 cylinders, total 209715200 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0xdd2fff2e

    Device Boot      Start         End      Blocks   Id  System

/dev/xvdb1            2048   209715199   104856576   8e  Linux LVM

#写入分区信息
Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.

#重新加载分区信息
root@localhost:~# partprobe 
root@localhost:~# fdisk -l

Disk /dev/xvda: 32.2 GB, 32212254720 bytes
255 heads, 63 sectors/track, 3916 cylinders, total 62914560 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00020a62

    Device Boot      Start         End      Blocks   Id  System

/dev/xvda1            2048      499711      248832   83  Linux
/dev/xvda2          501758    62912511    31205377    5  Extended
/dev/xvda5          501760    62912511    31205376   8e  Linux LVM

Disk /dev/xvdb: 107.4 GB, 107374182400 bytes
43 heads, 44 sectors/track, 110843 cylinders, total 209715200 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0xdd2fff2e
#这是我们刚创建的lvm分区
    Device Boot      Start         End      Blocks   Id  System
/dev/xvdb1            2048   209715199   104856576   8e  Linux LVM

Disk /dev/mapper/localhost--vg-root: 30.6 GB, 30614224896 bytes
255 heads, 63 sectors/track, 3721 cylinders, total 59793408 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000

Disk /dev/mapper/localhost--vg-root doesn't contain a valid partition table

Disk /dev/mapper/localhost--vg-swap_1: 1337 MB, 1337982976 bytes
255 heads, 63 sectors/track, 162 cylinders, total 2613248 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000

Disk /dev/mapper/localhost--vg-swap_1 doesn't contain a valid partition table

#创建一个lvm物理卷pv
root@localhost:~# pvcreate /dev/xvdb1
  Physical volume "/dev/xvdb1" successfully created

root@localhost:~# pvscan
  PV /dev/xvda5   VG localhost-vg    lvm2 [29.76 GiB / 0    free]
  PV /dev/xvdb1                      lvm2 [100.00 GiB]
  Total: 2 [129.76 GiB] / in use: 1 [29.76 GiB] / in no VG: 1 [100.00 GiB]

#因为我们在安装ubuntu时选择lvm类型，所以已经有了vg了。
root@localhost:~# vgscan
  Reading all physical volumes.  This may take a while...
  Found volume group "localhost-vg" using metadata type lvm2

#把新建的pv添加到vg里吧
root@localhost:~# vgextend localhost-vg /dev/xvdb1
  Volume group "localhost-vg" successfully extended
#看一下vg的详情

root@localhost:~# vgdisplay 
  --- Volume group ---
  VG Name               localhost-vg
  System ID             
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  4
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                2
  Open LV               2
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               129.75 GiB
  PE Size               4.00 MiB
  Total PE              33217
  Alloc PE / Size       7618 / 29.76 GiB
  Free  PE / Size       25599 / 100.00 GiB  <= 这个信息很重要，我们在扩大lv时要用到
  VG UUID               B1b5mq-JMhc-HbPo-ImdF-50pB-fpML-eivSLF

#看看系统里的逻辑卷lv吧
root@localhost:~# lvdisplay 
  --- Logical volume ---
  LV Path                /dev/localhost-vg/root
  LV Name                root
  VG Name                localhost-vg
  LV UUID                YfzzNm-ozhH-k2r2-KucH-p2he-CcVj-SCpuAr
  LV Write Access        read/write
  LV Creation host, time localhost, 2015-01-26 16:36:55 +0800
  LV Status              available

# open                 1

  LV Size                28.51 GiB  <= 大小还没变
  Current LE             7299
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto

- currently set to     256
  Block device           252:0
  
  --- Logical volume ---
  LV Path                /dev/localhost-vg/swap_1
  LV Name                swap_1
  VG Name                localhost-vg
  LV UUID                6Kid6V-tQ1B-TT43-g7at-Ha1d-yR3X-DomZA7
  LV Write Access        read/write
  LV Creation host, time localhost, 2015-01-26 16:36:55 +0800
  LV Status              available
  
  # open                 2
  
  LV Size                1.25 GiB
  Current LE             319
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto

- currently set to     256
  Block device           252:1

#扩大lv的容量,加号后面的值要是vgdisplay命令输出里的Free  PE / Size       25599 / 100.00 GiB 这个值，当然可以小于这个值，也就是不全部分配给这个lv.
root@localhost:~# lvresize -l +25599 /dev/localhost-vg/root
  Extending logical volume root to 128.51 GiB
  Logical volume root successfully resized

#再看看vg,会发现Free PE数量为0了。
root@localhost:~# vgdisplay
  --- Volume group ---
  VG Name               localhost-vg
  System ID             
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  5
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                2
  Open LV               2
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               129.75 GiB
  PE Size               4.00 MiB
  Total PE              33217
  Alloc PE / Size       33217 / 129.75 GiB
  Free  PE / Size       0 / 0   
  VG UUID               B1b5mq-JMhc-HbPo-ImdF-50pB-fpML-eivSLF

#df一下看看。
root@localhost:~# df -h
Filesystem                      Size  Used Avail Use% Mounted on
/dev/mapper/localhost--vg-root   28G  1.3G   26G   5% /
none                            4.0K     0  4.0K   0% /sys/fs/cgroup
udev                            4.8G  4.0K  4.8G   1% /dev
tmpfs                           979M  400K  978M   1% /run
none                            5.0M     0  5.0M   0% /run/lock
none                            4.8G     0  4.8G   0% /run/shm
none                            100M     0  100M   0% /run/user
/dev/xvda1                      236M   37M  187M  17% /boot

#What!为什么这里的容量还不对？因为你只是调整了lv的大小，它上面的文件系统还不知道呢。
#好吧，我们来把lv的容量扩到文件系统里吧。  
root@localhost:~# resize2fs /dev/mapper/localhost--vg-root 
resize2fs 1.42.9 (4-Feb-2014)
Filesystem at /dev/mapper/localhost--vg-root is mounted on /; on-line resizing required
old_desc_blocks = 2, new_desc_blocks = 9
The filesystem on /dev/mapper/localhost--vg-root is now 33687552 blocks long.

#再看看吧，OK!
root@localhost:~# df -h
Filesystem                      Size  Used Avail Use% Mounted on
/dev/mapper/localhost--vg-root  127G  1.3G  120G   2% /
none                            4.0K     0  4.0K   0% /sys/fs/cgroup
udev                            4.8G  4.0K  4.8G   1% /dev
tmpfs                           979M  400K  978M   1% /run
none                            5.0M     0  5.0M   0% /run/lock
none                            4.8G     0  4.8G   0% /run/shm
none                            100M     0  100M   0% /run/user
/dev/xvda1                      236M   37M  187M  17% /boot
{% endhighlight %}
