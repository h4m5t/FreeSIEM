# 安装Kali Purple

## 一、介绍

参考：https://www.kali.org/blog/kali-linux-2023-1-release/

https://gitlab.com/kalilinux/kali-purple/documentation/-/wikis/home

今年是Kali发布的10周年，Kali官方也推出了一款新的防御系统，Kali Purple。

Kali不仅是进攻，而且开始防守。

Kali Purple 从一个概念验证开始，演变成一个框架，然后是一个平台*（就像今天的 Kali 一样）*。目标是让每个人都能获得企业级安全性。



**What is in Kali Purple?**

On a higher level, Kali Purple consists of:

- A reference architecture for the ultimate SOC In-A-Box; perfect for:
  - Learning
  - Practicing SOC analysis and threat hunting
  - Security control design and testing
  - Blue / Red / Purple teaming exercises
  - anKali spy vs. spy competitions ( bare knuckle Blue vs. Red )
  - Protection of small to medium size environments
- Over 100 defensive tools, such as:
  - [Arkime](https://pkg.kali.org/pkg/arkime) - Full packet capture and analysis
  - [CyberChef](https://pkg.kali.org/pkg/cyberchef) - The cyber swiss army knife
  - `Elastic Security` - Security Information and Event Management
  - [GVM](https://www.kali.org/tools/gvm/) - Vulnerability scanner
  - [TheHive](https://pkg.kali.org/pkg/thehive) - Incident response platform
  - `Malcolm` - Network traffic analysis tool suite
  - [Suricata](https://pkg.kali.org/pkg/suricata) - Intrusion Detection System
  - [Zeek](https://pkg.kali.org/pkg/zeek) - (another) Intrusion Detection System *(both have their use-cases!)*
  - *…and of course all the usual [Kali tools](https://www.kali.org/tools/)*
- Defensive tools [documentations](https://gitlab.com/kalilinux/kali-purple/documentation)
- [Pre-generated image](https://www.kali.org/get-kali/)
- Kali Autopilot - an attack script builder / framework for automated attacks
- Kali Purple Hub for the community to share:
  - Practice pcaps
  - Kali Autopilot scripts for blue teaming exercises
- [Community Wiki](https://gitlab.com/kalilinux/kali-purple/documentation/-/wikis/home)
- A defensive menu structure according to NIST CSF (National Institute of Standards and Technology Critical Infrastructure Cybersecurity):
  - Identify
  - Protect
  - Detect
  - Respond
  - Recover
- Kali Purple [Discord](https://discord.kali.org/) channels for community collaboration and fun
- And theme: installer, menu entries & Xfce!

…And this is just the beginning of our journey.

![img](https://gitlab.com/kalilinux/kali-purple/documentation/-/raw/main/pictures/Kali-Purple-03-Architecture.png)



## 二、搭建

打开官网https://www.kali.org/

点击下载镜像文件：KaliPurple

VMware启动，安装即可。

新发布的版本中不包含Elastic,OpenCTI等系统，需要自己安装。



## 三、使用说明

### 3.1系统界面

全新的界面和菜单栏



### 3.2防御体系

根据安全防御体系，所有安全工具分为五类：

- IDENTIFY
- PROTECT
- DETECT
- RESPOND
- RECOVER

我们可以结合需要，根据此分类扩展需要的组件。



### 3.3安装其他组件

#### 安装spiderfoot

```
wget https://github.com/smicallef/spiderfoot/archive/v4.0.tar.gz
ls
tar zxvf spiderfoot-4.0.tar.gz
cd spiderfoot-4.0
ls
pip3 install -r requirements.txt
ifconfig
ls
python3 ./sf.py -l 127.0.0.1:5001
```

#### 安装Sn1per

```
git clone https://github.com/1N3/Sn1per
ping baidu.com
tar zxvf Sn1per-9.1.tar.gz
cd Sn1per-9.1
ls
bash install.sh
sniper
sniper --help
sniper -t 172.29.249.240 -m web
```

#### 安装Elastic stack

参考：https://gitlab.com/kalilinux/kali-purple/documentation/-/blob/main/301_kali-purple/installation.txt

```
# Elastic stack installation

# 1. Install dependencies:
# ------------------------

sudo apt-get install curl
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/elastic-archive-keyring.gpg
echo "deb https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-8.x.list
sudo bash -c "export HOSTNAME=kali-purple.kali.purple; apt-get install elasticsearch -y"

# take note of "elastic" user password


# 2. Convert to single-node setup (or replace fqdn name in initial_master_nodes list with IP address):
# -----------------------------------------------------------------------------------------------------
sudo sed -e '/cluster.initial_master_nodes/ s/^#*/#/' -i /etc/elasticsearch/elasticsearch.yml
echo "discovery.type: single-node" | sudo tee -a /etc/elasticsearch/elasticsearch.yml


# 3. Install Kibana:
# ------------------
sudo apt install kibana
sudo /usr/share/kibana/bin/kibana-encryption-keys generate -q
# Add keys to /etc/kibana/kibana.yml
echo "server.host: \"kali-purple.kali.purple\"" | sudo tee -a /etc/kibana/kibana.yml
# Ensure kli-purple.kali.purple is only mapped to 192.168.253.5 in /etc/hosts in order to bind Kibana to that interface
sudo systemctl enable elasticsearch kibana --now



# 4. Enroll Kibana:
# -----------------
sudo /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana

# open browser and navigate to http://192.168.253.5:5601
# enter username=elastic and password as displayed after installation
# paste token from above

sudo /usr/share/kibana/bin/kibana-verification-code

# enter verification code into Kibana when prompted



# 4.Enable HTTPS for Kibana:
# --------------------------

/usr/share/elasticsearch/bin/elasticsearch-certutil ca
/usr/share/elasticsearch/bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12 --dns kali-purple.kali.purple,elastic.kali.purple,kali-purple --out kibana-server.p12
sudo openssl pkcs12 -in /usr/share/elasticsearch/kibana-server.p12 -out /etc/kibana/kibana-server.crt -clcerts -nokeys
sudo openssl pkcs12 -in /usr/share/elasticsearch/kibana-server.p12 -out /etc/kibana/kibana-server.key -nocerts -nodes
sudo chown root:kibana /etc/kibana/kibana-server.key
sudo chown root:kibana /etc/kibana/kibana-server.crt
sudo chmod 660 /etc/kibana/kibana-server.key
sudo chmod 660 /etc/kibana/kibana-server.crt

echo "server.ssl.enabled: true" | sudo tee -a /etc/kibana/kibana.yml
echo "server.ssl.certificate: /etc/kibana/kibana-server.crt" | sudo tee -a /etc/kibana/kibana.yml
echo "server.ssl.key: /etc/kibana/kibana-server.key" | sudo tee -a /etc/kibana/kibana.yml
echo "server.publicBaseUrl: \"https://kali-purple.kali.purple:5601\"" | sudo tee -a /etc/kibana/kibana.yml
```

注意，保存安装过程中的密钥，Token等信息。

输入Token,并注册一下。

```
sudo /usr/share/kibana/bin/kibana-verification-code
```

添加Security集成 

注册Fleet

安装Fleet



## 四、参考资料

https://www.kali.org/blog/kali-linux-2023-1-release/

https://gitlab.com/kalilinux/kali-purple/documentation/-/wikis/home

https://www.youtube.com/watch?v=UNxrR4mnOnA
