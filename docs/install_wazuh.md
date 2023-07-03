# Wazuh安装

## 介绍

Wazuh 是一个安全平台，可为端点和云工作负载提供统一的 XDR 和 SIEM 保护。该解决方案由一个通用代理和三个中心组件组成：Wazuh 服务器、Wazuh 索引器和 Wazuh 仪表板。

Wazuh is a security platform that provides unified XDR and SIEM protection for endpoints and cloud workloads. The solution is composed of a single universal agent and three central components: the Wazuh server, the Wazuh indexer, and the Wazuh dashboard.

The Wazuh platform helps organizations and individuals protect their data assets through threat prevention, detection, and response. Besides, Wazuh is also employed to meet regulatory compliance requirements, such as PCI DSS or HIPAA, and configuration standards like CIS hardening guides.

官网：https://wazuh.com/

官方文档：https://documentation.wazuh.com/

GitHub仓库：https://github.com/wazuh/wazuh



使用场景：

| [Log data analysis](https://documentation.wazuh.com/current/getting-started/use-cases/log-analysis.html) | [File integrity monitoring](https://documentation.wazuh.com/current/getting-started/use-cases/file-integrity.html) |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [Rootkits detection](https://documentation.wazuh.com/current/getting-started/use-cases/rootkits-detection.html) | [Active response](https://documentation.wazuh.com/current/getting-started/use-cases/active-response.html) |
| [Configuration assessment](https://documentation.wazuh.com/current/getting-started/use-cases/configuration-assessment.html) | [System inventory](https://documentation.wazuh.com/current/getting-started/use-cases/system-inventory.html) |
| [Vulnerability detection](https://documentation.wazuh.com/current/getting-started/use-cases/vulnerability-detection.html) | [Cloud security](https://documentation.wazuh.com/current/getting-started/use-cases/cloud-security.html) |
| [Container security](https://documentation.wazuh.com/current/getting-started/use-cases/container-security.html) | [Regulatory compliance](https://documentation.wazuh.com/current/getting-started/use-cases/regulatory-compliance.html) |



## 部署方式

本次采用单节点部署。对于较大环境，建议采用分布式部署，多节点集群配置，从而提供高可用性和负载平衡。

### 硬件要求

| **Agents** | **CPU** | **RAM** | **Storage (90 days)** |
| ---------- | ------- | ------- | --------------------- |
| 1–25       | 4 vCPU  | 8 GiB   | 50 GB                 |
| 25–50      | 8 vCPU  | 8 GiB   | 100 GB                |
| 50–100     | 8 vCPU  | 8 GiB   | 200 GB                |



### 开放端口

| Component       | Port      | Protocol       | Purpose                                        |
| --------------- | --------- | -------------- | ---------------------------------------------- |
| Wazuh server    | 1514      | TCP (default)  | Agent connection service                       |
|                 | 1514      | UDP (optional) | Agent connection service (disabled by default) |
|                 | 1515      | TCP            | Agent enrollment service                       |
|                 | 1516      | TCP            | Wazuh cluster daemon                           |
|                 | 514       | UDP (default)  | Wazuh Syslog collector (disabled by default)   |
|                 | 514       | TCP (optional) | Wazuh Syslog collector (disabled by default)   |
|                 | 55000     | TCP            | Wazuh server RESTful API                       |
| Wazuh indexer   | 9200      | TCP            | Wazuh indexer RESTful API                      |
|                 | 9300-9400 | TCP            | Wazuh indexer cluster communication            |
| Wazuh dashboard | 443       | TCP            | Wazuh web user interface                       |



## OVA镜像部署

### 下载并启动

下载镜像文件：https://documentation.wazuh.com/current/deployment-options/virtual-machine/virtual-machine.html

此镜像是基于CentOS7。

在虚拟化平台（VMware,ESXi,VirtualBox,Hyper-V等）上导入和启动此OVA。

以VMware为例：点击打开虚拟机，选择下载好的OVA镜像，导入。



系统登录：

`root`用户的密码也是`wazuh`。但是，只能使用系统用户通过 SSH 访问虚拟机。禁用 root 用户的 SSH 登录。

```
user: wazuh-user
password: wazuh
```



安装几个工具包(便于后面操作，可以不装，如需用到再装)：

```
yum -y install vim
yum -y install psmisc
yum -y install net-tools
yum -y install lvm2
```

### 配置系统时间

修改时区

```
备份原始时间文件
mv /etc/localtime /etc/localtime.bak
创建软连接，修改为CST时区
ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

### 磁盘扩容

下载好的OVA打开默认是50G，需要扩展磁盘。首先在VMware中调正磁盘大小。

然后在客户机操作系统内部对磁盘重新进行分区和扩展文件系统。

如果新增的磁盘在sdb,需要使用vgdisplay(查看卷组信息)、vgextend(扩展卷组)等命令，因此需要安装lvm2工具，再进行扩容操作，可以参考thehive安装教程中磁盘扩容章节。

参考连接：https://github.com/wazuh/wazuh/issues/11017

**扩容前**：

```
[root@wazuh-server /]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0  100G  0 disk
└─sda1   8:1    0   50G  0 part /
sr0     11:0    1 1024M  0 rom
[root@wazuh-server /]# df -hl
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        3.9G     0  3.9G   0% /dev
tmpfs           3.9G   60K  3.9G   1% /dev/shm
tmpfs           3.9G  8.9M  3.9G   1% /run
tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sda1        50G  6.8G   44G  14% /
tmpfs           799M     0  799M   0% /run/user/1000
```

```
[root@wazuh-server /]# parted /dev/sda  print free
Model: ATA VMware Virtual I (scsi)
Disk /dev/sda: 107GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start   End     Size    Type     File system  Flags
        32.3kB  1049kB  1016kB           Free Space
 1      1049kB  53.7GB  53.7GB  primary  xfs
        53.7GB  107GB   53.7GB           Free Space

[root@wazuh-server /]# fdisk -l

Disk /dev/sda: 107.4 GB, 107374182400 bytes, 209715200 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000a05f8

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1            2048   104857599    52427776   83  Linux
```

**进行扩容**：在此处，我将sda1从50G扩容为100G。

```
#启动一个分区向导
[root@wazuh-server /]# fdisk /dev/sda
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): d
Selected partition 1
Partition 1 is deleted

Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p):
Using default response p
Partition number (1-4, default 1):
First sector (2048-209715199, default 2048):
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-209715199, default 209715199):
Using default value 209715199
Partition 1 of type Linux and of size 100 GiB is set

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.

WARNING: Re-reading the partition table failed with error 16: Device or resource busy.
The kernel still uses the old table. The new table will be used at
the next reboot or after you run partprobe(8) or kpartx(8)
Syncing disks.

[root@wazuh-server /]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0  100G  0 disk
└─sda1   8:1    0   50G  0 part /
sr0     11:0    1 1024M  0 rom

#根据上面的警告信息，需要运行如下命令。
[root@wazuh-server /]# partprobe

[root@wazuh-server /]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0  100G  0 disk
└─sda1   8:1    0  100G  0 part /
sr0     11:0    1 1024M  0 rom

#目前只调整了分区的大小，接下来需要调整文件系统的大小
[root@wazuh-server /]# xfs_growfs /dev/sda1
meta-data=/dev/sda1              isize=512    agcount=6, agsize=2621376 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=13106944, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=5119, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 13106944 to 26214144
```

我建议扩容之后再重启一次操作系统。

查看扩容后的磁盘情况：

```
[root@wazuh-server /]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0  100G  0 disk
└─sda1   8:1    0  100G  0 part /
sr0     11:0    1 1024M  0 rom
[root@wazuh-server /]# df -hl
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        3.9G     0  3.9G   0% /dev
tmpfs           3.9G   60K  3.9G   1% /dev/shm
tmpfs           3.9G  8.9M  3.9G   1% /run
tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sda1       100G  7.1G   93G   8% /
tmpfs           799M     0  799M   0% /run/user/1000
[root@wazuh-server /]# parted /dev/sda  print free
Model: ATA VMware Virtual I (scsi)
Disk /dev/sda: 107GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start   End     Size    Type     File system  Flags
        32.3kB  1049kB  1016kB           Free Space
 1      1049kB  107GB   107GB   primary  xfs
```



### 网络配置

默认情况下，网络为桥接模式。在VMware中改为NAT模式。

VM 将尝试从网络 DHCP 服务器获取 IP 地址。或者，可以通过在 VM 所基于的 CentOS 操作系统中配置适当的网络文件来设置静态 IP 地址。

设置网络：

```
vi /etc/sysconfig/network-scripts/ifcfg-eth0
```

其中，网关，DNS等根据情况自行配置。

```
DEVICE=eth0
ONBOOT=yes
BOOTPROTO=static
TYPE=Ethernet
NM_CONTROLLED=no
IPADDR=192.168.76.149
NETMASK=255.255.255.0
GATEWAY=192.168.76.2
DNS=172.31.1.1
```

重启网卡。

systemctl restart network



### 配置文件

该虚拟映像中包含的所有组件均配置为开箱即用，无需修改任何设置。然而，所有组件都可以完全定制。这些是配置文件位置：

- Wazuh manager: `/var/ossec/etc/ossec.conf`

- Wazuh indexer: `/etc/wazuh-indexer/opensearch.yml`

- Filebeat-OSS: `/etc/filebeat/filebeat.yml`

- Wazuh dashboard:

  > - `/etc/wazuh-dashboard/opensearch_dashboards.yml`
  > - `/usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml`






### web界面登录

打开web界面。用刚才的IP登录。https://192.168.76.149

```
user: admin
password: admin
```



## 安装客户端代理

https://documentation.wazuh.com/current/installation-guide/wazuh-agent/index.html

The Wazuh agent provides [key features](https://documentation.wazuh.com/current/getting-started/components/wazuh-agent.html#agents-modules) to enhance your system’s security.

| Log collector                   | Command execution                       |
| ------------------------------- | --------------------------------------- |
| File integrity monitoring (FIM) | Security configuration assessment (SCA) |
| System inventory                | Malware detection                       |
| Active response                 | Container security                      |
| Cloud security                  |                                         |

注意：

* 如果在具有大量服务器或端点的大型环境中部署 Wazuh，建议使用Puppet、Chef、 SCCM 或Ansible等自动化工具
* 在安装Wazuh agent and the Wazuh manager，要注意版本的兼容性。

安装方式：

* 根据官方教程安装
* 在Wazuh仪表盘->代理，点击部署新代理，根据提示安装。

