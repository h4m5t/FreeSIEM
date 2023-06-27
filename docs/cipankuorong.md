Ubuntu LVM磁盘扩容：

扩容前：

```
#df -h

Filesystem                         Size  Used Avail Use% Mounted on
udev                               3.9G     0  3.9G   0% /dev
tmpfs                              796M  1.5M  795M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv  9.8G  8.2G  1.1G  89% /
tmpfs                              3.9G     0  3.9G   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
tmpfs                              3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/loop0                          62M   62M     0 100% /snap/core20/1328
/dev/sda2                          1.4G  307M  998M  24% /boot
/dev/loop2                          44M   44M     0 100% /snap/snapd/14978
/dev/loop1                          68M   68M     0 100% /snap/lxd/21835
tmpfs                              796M     0  796M   0% /run/user/1000


#lsblk

NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
loop0                       7:0    0 61.9M  1 loop /snap/core20/1328
loop1                       7:1    0 67.2M  1 loop /snap/lxd/21835
loop2                       7:2    0 43.6M  1 loop /snap/snapd/14978
sda                         8:0    0   16G  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0  1.4G  0 part /boot
└─sda3                      8:3    0 14.6G  0 part
  └─ubuntu--vg-ubuntu--lv 253:0    0   10G  0 lvm  /
sdb                         8:16   0  484G  0 disk
sr0                        11:0    1 1024M  0 rom



#pvdisplay
  --- Physical volume ---
  PV Name               /dev/sda3
  VG Name               ubuntu-vg
  PV Size               <14.58 GiB / not usable 0
  Allocatable           yes
  PE Size               4.00 MiB
  Total PE              3732
  Free PE               1172
  Allocated PE          2560
  PV UUID               UsKYdO-8NJ3-pFkC-jdyf-4fhP-iAIi-9c075z

  "/dev/sdb" is a new physical volume of "484.00 GiB"
  --- NEW Physical volume ---
  PV Name               /dev/sdb
  VG Name
  PV Size               484.00 GiB
  Allocatable           NO
  PE Size               0
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               2KaSOV-eLvT-qvUY-L3NX-iSvW-zNu2-WfWKlB

```

扩容方法

```
# 创建物理卷
pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created
  
# 将磁盘加到vg_data 逻辑组中
vgextend ubuntu-vg /dev/sdb

#vgdisplay
--- Volume group ---
  VG Name               ubuntu-vg
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
  VG Size               498.57 GiB
  PE Size               4.00 MiB
  Total PE              127635
  Alloc PE / Size       2560 / 10.00 GiB
  Free  PE / Size       125075 / 488.57 GiB
  VG UUID               dAbRG7-k3Z6-EzUs-1cou-4Qll-HNQf-HOub32


# 扩容逻辑卷
lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv

# 查看逻辑卷容量
lvdisplay /dev/mapper/ubuntu--vg-ubuntu--lv

  --- Logical volume ---
  LV Path                /dev/ubuntu-vg/ubuntu-lv
  LV Name                ubuntu-lv
  VG Name                ubuntu-vg
  LV UUID                HAJoth-dCcG-yHoP-UrVH-uzLT-EE2b-zQogUs
  LV Write Access        read/write
  LV Creation host, time ubuntu-server, 2022-07-04 20:17:08 +0800
  LV Status              available
  # open                 1
  LV Size                498.57 GiB
  Current LE             127635
  Segments               2
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:0

# 重新计算逻辑卷大小
resize2fs  /dev/mapper/ubuntu--vg-ubuntu--lv
```

扩容后

```
#df -h

Filesystem                         Size  Used Avail Use% Mounted on
udev                               3.9G     0  3.9G   0% /dev
tmpfs                              796M  1.5M  795M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv  491G  8.2G  463G   2% /
tmpfs                              3.9G     0  3.9G   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
tmpfs                              3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/loop0                          62M   62M     0 100% /snap/core20/1328
/dev/sda2                          1.4G  307M  998M  24% /boot
/dev/loop2                          44M   44M     0 100% /snap/snapd/14978
/dev/loop1                          68M   68M     0 100% /snap/lxd/21835
tmpfs                              796M     0  796M   0% /run/user/1000
overlay                            491G  8.2G  463G   2% /var/lib/docker/overlay2/3e8b6037853535471cd5009da49ad37f089c900c5e74111ccf2d8df0905422c8/merged
overlay                            491G  8.2G  463G   2% /var/lib/docker/overlay2/89a79b752fe3343b51c7b37380c00f4405edb5bceb3dc5415d83b59c00db3ce7/merged
overlay                            491G  8.2G  463G   2% /var/lib/docker/overlay2/cd3c0af98461798c97f693d2ef58c41ef1673a379207b5f3858837c183aaf0d7/merged
overlay                            491G  8.2G  463G   2% /var/lib/docker/overlay2/0c30a836f57be6da2ce0147149e9dc8e649917a5721522fd9032a06faba2e470/merged
```

