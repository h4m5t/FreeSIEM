# 搭建MISP

## 一、介绍

MISP 是一款开源软件解决方案，用于收集、存储、分发和共享有关网络安全事件分析和恶意软件分析的网络安全指标和威胁。MISP 是由事件分析师、安全和 ICT 专业人员或恶意软件逆向者设计的，旨在支持他们的日常运营，从而有效地共享结构化信息。

MISP 的目标是促进安全界内外的结构化信息共享。MISP 提供的功能不仅支持信息交换，还支持网络入侵检测系统 (NIDS)、LIDS 以及日志分析工具、SIEM 对所述信息的使用。

官网：

https://github.com/MISP/MISP

https://www.misp-project.org/download/



## 二、安装

### 2.1使用安装脚本

curl https://raw.githubusercontent.com/MISP/MISP/2.4/INSTALL/INSTALL.sh -o misp_install.sh -k


wget -O /tmp/INSTALL.sh https://raw.githubusercontent.com/MISP/MISP/2.4/INSTALL/INSTALL.sh
bash /tmp/INSTALL.sh -A



多次尝试安装失败，显示sha1校验失败，未能解决。

在家中自己电脑上尝试，有时校验成功，有时校验失败。

使用安装脚本失败，考虑使用镜像部署。

### 2.2使用镜像

从官网下载OVA镜像，

https://vm.misp-project.org/

VMware安装即可。
```
For the MISP web interface -> admin@admin.test:admin
For the system -> misp:Password1234
```

```
sudo -u www-data /var/www/MISP/app/Console/cake Baseurl https://a.b.c.d
sudo systemctl restart apache2
```

MISP.password1

找到API TOken

```
eybqhsnylpJ6gHyx0xBYc11MrnvIL3oVETm4NC6v
```


### 2.3docker安装

搭建之后的初始用户名和密码为：

MISP初始密码

[admin@admin.test](mailto:admin@admin.test)

admin

```
misp:    image: coolacid/misp-docker:core-latest    restart: unless-stopped    depends_on:       - misp_mysql    ports:      - "0.0.0.0:80:80"      - "0.0.0.0:443:443"    volumes:      - "./server-configs/:/var/www/MISP/app/Config/"      - "./logs/:/var/www/MISP/app/tmp/logs/"      - "./files/:/var/www/MISP/app/files"      - "./ssl/:/etc/nginx/certs"    environment:      - MYSQL_HOST=misp_mysql      - MYSQL_DATABASE=mispdb      - MYSQL_USER=mispuser      - MYSQL_PASSWORD=misppass      - MISP_ADMIN_EMAIL=mispadmin@lab.local      - MISP_ADMIN_PASSPHRASE=mispadminpass      - MISP_BASEURL=localhost      - TIMEZONE=Europe/London      - "INIT=true"               - "CRON_USER_ID=1"         - "REDIS_FQDN=redis"      - "HOSTNAME=https://192.168.76.148"    networks:      - SOC_NET   misp_mysql:    image: mysql/mysql-server:5.7    restart: unless-stopped    volumes:      - mispsqldata:/var/lib/mysql       environment:      - MYSQL_DATABASE=mispdb      - MYSQL_USER=mispuser      - MYSQL_PASSWORD=misppass      - MYSQL_ROOT_PASSWORD=mispass    networks:      - SOC_NET  redis:    image: redis:5.0.6    networks:      - SOC_NET  misp-modules:    image: coolacid/misp-docker:modules-latest    environment:      - "REDIS_BACKEND=redis"    depends_on:      - redis      - misp_mysql    networks:      - SOC_NET    volumes:  miniodata:  cassandradata:  elasticsearchdata:  cortexdata:  thehivedata:  mispsqldata: networks:    SOC_NET:          driver: bridge
```
