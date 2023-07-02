# 一、介绍

Linux distro for threat hunting, enterprise security monitoring, and log management

Security Onion includes a native web interface with built-in tools analysts use to respond to alerts, hunt for evil, catalog evidence into cases, monitor grid performance, and much more. Additionally, third-party tools, such as Elasticsearch, Logstash, Kibana, Suricata, Zeek (formerly known as Bro), Wazuh, Stenographer, CyberChef, NetworkMiner, and many more are included.

Security Onion is a free and open platform for threat hunting, enterprise security monitoring, and log management. It includes our own interfaces for alerting, dashboards, hunting, PCAP, and case management. It also includes other tools such as Playbook, osquery, CyberChef, Elasticsearch, Logstash, Kibana, Suricata, and Zeek.

目前已更新至2.4版本，使用Elastic Fleet架构。

# 二、搭建

## 2.1下载镜像

https://github.com/Security-Onion-Solutions/securityonion/blob/master/VERIFY_ISO.md

## 2.2安装镜像

注意，选择合适的内存大小，系统版本。

1. From the VMware main window, select File >> New Virtual Machine.
2. Select Typical installation >> Click `Next`.
3. Installer disc image file >> SO ISO file path >> Click `Next`.
4. Choose Linux, CentOS 7 64-Bit and click `Next`.
5. Specify virtual machine name and click `Next`.
6. Specify disk size (minimum 200GB), store as single file, click `Next`.
7. Customize hardware and increase Memory and Processors based on the [Hardware Requirements](https://docs.securityonion.net/en/2.3/hardware.html#hardware) section.
8. Network Adapter (NAT or Bridged – if you want to be able to access your Security Onion machine from other devices in the network, then choose Bridged, otherwise choose NAT to leave it behind the host) – in this tutorial, this will be the management interface.
9. Add >> Network Adapter (Bridged) - this will be the sniffing (monitor) interface.
10. Click `Close`.
11. Click `Finish`.
12. Power on the virtual machine and then follow the installation steps for your desired installation type in the [Installation](https://docs.securityonion.net/en/2.3/installation.html#installation) section.



## 2.3开始配置

注意：TAB键是选择，空格键是单击，回车键是确认。

安装时一般选择STANDALONE模式。



初始化过程中需要输入的配置较多，根据提示选择合适的即可。

选择网卡和静态IP。

配置Linux系统的用户名和密码。

配置web接口的邮箱和密码。



选择yes允许so-allow访问web工具、因为安装完成后我们要使用sudo so-allow 来启动web界面



提示输入允许访问的IP范围。

进行初始化：时间漫长，可能需要十几个小时。



## 2.4启动

安装完成后重启，使用Linux系统的用户名和密码登录。

输入sudo so-allow

选择a)



可能还需要再输入一遍允许访问的源IP范围。



使用命令sudo so-status 查看运行服务状态



# 三、使用说明

## 3.1登录

登录后主界面如下：



Kibana界面。



Grafana界面：



playbook界面：



Fleet界面。



# 四、参考资料

虽然是镜像，但安装过程较为繁琐。

可能会因为“内存不足”，“网络不通”，“安装慢”等原因失败。可以看参考文档排查解决。如果安装速度很慢，超过十几个小时，请耐心等待，或重新安装。

https://docs.securityonion.net/en/2.3/

[Vmware workstation12里如何正确快速安装可视化IDS系统Security Onion（图文详解）](https://www.bbsmax.com/A/A7zg3kqlz4/)

[Security-Onion-Solutions/securityonion: Security Onion is a free and open platform for threat hunting, enterprise security monitoring, and log management. It includes our own interfaces for alerting, dashboards, hunting, PCAP, and case management. It also includes other tools such as Playbook, osquery, CyberChef, Elasticsearch, Logstash, Kibana, Suricata, and Zeek.](https://github.com/Security-Onion-Solutions/securityonion)

[Security-Onion-Solutions安全洋葱安装方法_信息安全兴趣爱好者的博客-CSDN博客](https://blog.csdn.net/scxiaotan1/article/details/127326059)

[Install and Setup Security Onion on VirtualBox - [kifarunix.com](http://kifarunix.com)](https://kifarunix.com/install-and-setup-security-onion-on-virtualbox/)

[Basic installation of Security Onion 2.3 – Bjoern Hagedorn](https://bjoern-hagedorn.com/security-onion-2-3/)

[Installation Security Onion](https://www.cyberhuntingguide.net/install-sec-onion.html)

[VMware — Security Onion 2.3 documentation](https://docs.securityonion.net/en/2.3/vmware.html)

https://blog.csdn.net/weixin_43543330/article/details/108310251

https://zhuanlan.zhihu.com/p/540203990?utm_id=0

https://blog.csdn.net/scxiaotan1/article/details/127326059

http://www.hackdig.com/01/hack-248769.htm
