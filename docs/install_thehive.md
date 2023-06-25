# 搭建 the hive
方法1：官方5.X docker安装。
方法2：非官方docker安装
https://ls111.me/thehive-cortex-misp-installation-using-docker-compose/

方法3：使用OVA镜像https://docs.strangebee.com/resources/demo/

## 使用docker安装
### 参考

https://chinyati.medium.com/the-hive-cortex-through-docker-installation-e50cbadb6cb0

https://wmvalente.medium.com/installing-misp-the-hive-and-cortex-part-5-d8a21c886fa8

https://github.com/TheHive-Project/Docker-Templates/tree/main/docker/thehive4-cortex31-shuffle

### compose.yml
```
version: "3"
services:
  thehive:
    image: strangebee/thehive:5.1
    restart: unless-stopped
    depends_on:
      - cassandra
      - elasticsearch
      - minio
      - cortex
    mem_limit: 1500m
    ports:
      - "0.0.0.0:9000:9000"
    environment:
      - JVM_OPTS="-Xms1024M -Xmx1024M"
    command:
      - --secret
      - "lab123456789"
      - "--cql-hostnames"
      - "cassandra"
      - "--index-backend"
      - "elasticsearch"
      - "--es-hostnames"
      - "elasticsearch"
      - "--s3-endpoint"
      - "http://minio:9002"
      - "--s3-access-key"
      - "minioadmin"
      - "--s3-secret-key"
      - "minioadmin"
      - "--s3-use-path-access-style"
      - "--no-config-cortex"
      #- "--cortex-port"
      #- "9001"
      #- "--cortex-keys"
      #- "k3DZO07qOoIMfNNS5qLloPmMS2PnhMMR"
    volumes:
      - thehivedata:/etc/thehive/application.conf
    networks:
      - SOC_NET

  cassandra:
    image: 'cassandra:4'
    restart: unless-stopped
    mem_limit: 1000m
    ports:
      - "0.0.0.0:9042:9042"
    environment:
      - CASSANDRA_CLUSTER_NAME=TheHive
    volumes:
      - cassandradata:/var/lib/cassandra
    networks:
      - SOC_NET

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.9
    restart: unless-stopped
    mem_limit: 512m
    ports:
      - "0.0.0.0:9200:9200"
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - cluster.name=hive
      - http.host=0.0.0.0
      - "ES_JAVA_OPTS=-Xms256m -Xmx256m"
    volumes:
      - elasticsearchdata:/usr/share/elasticsearch/data
    networks:
      - SOC_NET

  minio:
    image: quay.io/minio/minio
    mem_limit: 512m
    restart: unless-stopped
    command: ["minio", "server", "/data", "--console-address", ":9002"]
    environment:
      - MINIO_ROOT_USER=minioadmin
      - MINIO_ROOT_PASSWORD=minioadmin
    ports:
      - "0.0.0.0:9002:9002"
    volumes:
      - "miniodata:/data"
    networks:
      - SOC_NET

  cortex:
    image: thehiveproject/cortex:3.1.7
    restart: unless-stopped
    environment:
      - job_directory=/opt/cortex/jobs
    volumes:
      - cortexdata:/var/run/docker.sock
      - cortexdata:/opt/cortex/jobs
      - cortexdata:/var/log/cortex
      - cortexdata:/cortex/application.conf
    depends_on:
      - elasticsearch
    ports:
      - "0.0.0.0:9001:9001"
    networks:
      - SOC_NET

  misp:
    image: coolacid/misp-docker:core-latest
    restart: unless-stopped
    depends_on: 
      - misp_mysql
    ports:
      - "0.0.0.0:80:80"
      - "0.0.0.0:443:443"
    volumes:
      - "./server-configs/:/var/www/MISP/app/Config/"
      - "./logs/:/var/www/MISP/app/tmp/logs/"
      - "./files/:/var/www/MISP/app/files"
      - "./ssl/:/etc/nginx/certs"
    environment:
      - MYSQL_HOST=misp_mysql
      - MYSQL_DATABASE=mispdb
      - MYSQL_USER=mispuser
      - MYSQL_PASSWORD=misppass
      - MISP_ADMIN_EMAIL=mispadmin@lab.local
      - MISP_ADMIN_PASSPHRASE=mispadminpass
      - MISP_BASEURL=localhost
      - TIMEZONE=Europe/London
      - "INIT=true"         
      - "CRON_USER_ID=1"   
      - "REDIS_FQDN=redis"
      - "HOSTNAME=https://192.168.76.148"
    networks:
      - SOC_NET

  misp_mysql:
    image: mysql/mysql-server:5.7
    restart: unless-stopped
    volumes:
      - mispsqldata:/var/lib/mysql   
    environment:
      - MYSQL_DATABASE=mispdb
      - MYSQL_USER=mispuser
      - MYSQL_PASSWORD=misppass
      - MYSQL_ROOT_PASSWORD=mispass
    networks:
      - SOC_NET
  redis:
    image: redis:5.0.6
    networks:
      - SOC_NET
  misp-modules:
    image: coolacid/misp-docker:modules-latest
    environment:
      - "REDIS_BACKEND=redis"
    depends_on:
      - redis
      - misp_mysql
    networks:
      - SOC_NET   

volumes:
  miniodata:
  cassandradata:
  elasticsearchdata:
  cortexdata:
  thehivedata:
  mispsqldata:

networks:
    SOC_NET:
          driver: bridge
```



### 遇到的问题
容器冲突：
`docker stop $(docker ps -aq)`

提示端口冲突：

```
Error starting userland proxy: listen tcp4 0.0.0.0:9042: bind: address already in use
```

解决方法：

```
#查询PID
lsof -i :9042

#中止服务
kill PID
```

使用docker-compose安装，Cortex提示无法连接到es,查看docker发现，es一直处于退出状态。

查看es docker log

报错：

```
[root@centos1 Hive&Cortex]# docker logs 7b810d62bb7b
Exception in thread "main" java.lang.RuntimeException: starting java failed with [1]
output:
[0.000s][error][logging] Error opening log file 'logs/gc.log': Permission denied
[0.000s][error][logging] Initialization of output 'file=logs/gc.log' using options 'filecount=32,filesize=64m' failed.
error:
Invalid -Xlog option '-Xlog:gc*,gc+age=trace,safepoint:file=logs/gc.log:utctime,pid,tags:filecount=32,filesize=64m', see error log for details.
Error: Could not create the Java Virtual Machine.
Error: A fatal exception has occurred. Program will exit.
        at org.elasticsearch.tools.launchers.JvmOption.flagsFinal(JvmOption.java:119)
        at org.elasticsearch.tools.launchers.JvmOption.findFinalOptions(JvmOption.java:81)
        at org.elasticsearch.tools.launchers.JvmErgonomics.choose(JvmErgonomics.java:38)
        at org.elasticsearch.tools.launchers.JvmOptionsParser.jvmOptions(JvmOptionsParser.java:135)
        at org.elasticsearch.tools.launchers.JvmOptionsParser.main(JvmOptionsParser.java:86)

```



修改compose文件：

参考：https://blog.csdn.net/qq_18848239/article/details/108158741

```
TAKE_FILE_OWNERSHIP=true
```

打开Cortex9001页面，报错如下：

```
Error:user init not found
```

查看日志发现：

```
[error] o.t.c.s.ErrorHandler - Internal error
java.lang.RuntimeException: HttpResponse(429,Some(StringEntity({"error":{"root_cause":[{"type":"cluster_block_exception","reason":"index [cortex_6] blocked by: [TOO_MANY_REQUESTS/12/disk usage exceeded flood-stage watermark, index has read-only-allow-delete block];"}],"type":"cluster_block_exception","reason":"index [cortex_6] blocked by: [TOO_MANY_REQUESTS/12/disk usage exceeded flood-stage watermark, index has read-only-allow-delete block];"},"status":429},Some(UTF-8))),Map(X-elastic-product -> Elasticsearch, content-type -> application/json; charset=UTF-8, content-length -> 399))
```

解决方法：https://github.com/TheHive-Project/Cortex/issues/187

查找索引

```
curl -s http://localhost:9200/_all/_settings
```

删除索引：

```
curl -X DELETE http://localhost:9200/cortex_6/"
{"acknowledged":true}
```



## step by step安装

参考官方安装文档：

https://docs.thehive-project.org/thehive/installation-and-configuration/installation/step-by-step-guide/

参考安装视频：





在Ubuntu上搭建。我的本机IP是192.168.76.145

注意，Java和Cassandra版本必须要按照官方版本去安装。



## 系统初始配置

```
sudo sed -i 's@//.*archive.ubuntu.com@//mirrors.ustc.edu.cn@g' /etc/apt/sources.list
sudo apt-get update
sudo apt-get install git
sudo apt-get install vim
sudo apt-get install curl
sudo apt-get install net-tools
sudo apt-get install open-vm-tools-desktop #虚拟机安装，方便从宿主机复制命令，安装后重启生效
```



## 安装JVM

Java Virtual Machine

```
apt-get install -y openjdk-8-jre-headless
echo JAVA_HOME="/usr/lib/jvm/java-8-openjdk-amd64" >> /etc/environment
export JAVA_HOME="/usr/lib/jvm/java-8-openjdk-amd64"

```



## 安装Cassandra

Cassandra database

```
curl -fsSL https://www.apache.org/dist/cassandra/KEYS | sudo apt-key add -
echo "deb http://www.apache.org/dist/cassandra/debian 311x main" | sudo tee -a /etc/apt/sources.list.d/cassandra.sources.list

```

Install from repository,发现报错，官方仓库失效了。

```
sudo apt update
sudo apt install cassandra
```

添加源再安装：

```
echo "deb https://debian.cassandra.apache.org 311x main" | sudo tee -a /etc/apt/sources.list.d/cassandra.sources.list

curl https://downloads.apache.org/cassandra/KEYS | sudo apt-key add -

sudo apt-get update

sudo apt-get install cassandra
```

配置Cassandra：

```
cqlsh localhost 9042
cqlsh> UPDATE system.local SET cluster_name = 'thp' where key='local';
nodetool flush
```

修改Cassandra配置文件,cluster_name改为'thp'，将'xx.xx.xx.xx'替换为本机IP。

```
vi /etc/cassandra/cassandra.yaml

# content from /etc/cassandra/cassandra.yaml

cluster_name: 'thp'
listen_address: 'xx.xx.xx.xx' # address for nodes
rpc_address: 'xx.xx.xx.xx' # address for clients
seed_provider:
    - class_name: org.apache.cassandra.locator.SimpleSeedProvider
      parameters:
          # Ex: "<ip1>,<ip2>,<ip3>"
          - seeds: 'xx.xx.xx.xx' # self for the first node
data_file_directories:
  - '/var/lib/cassandra/data'
commitlog_directory: '/var/lib/cassandra/commitlog'
saved_caches_directory: '/var/lib/cassandra/saved_caches'
hints_directory: 
  - '/var/lib/cassandra/hints'
```

重启

```
systemctl daemon-reload
service cassandra restart
```

检查运行状态：

```
systemctl status cassandra
```

查看开放的端口

```
netstat -ltpnd

#有这两行说明运行正常。
tcp        0      0 192.168.76.145:7000     0.0.0.0:*               LISTEN      15268/java          
tcp        0      0 192.168.76.145:9042     0.0.0.0:*               LISTEN      15268/java        
```









## 安装the hive

```
#创建存储文件夹
mkdir -p /opt/thp/thehive/files

#添加apt-key
#国外用户使用
curl https://raw.githubusercontent.com/TheHive-Project/TheHive/master/PGP-PUBLIC-KEY | sudo apt-key add -

#国内用户使用，对raw的代理
curl https://raw.fastgit.org/TheHive-Project/TheHive/master/PGP-PUBLIC-KEY | sudo apt-key add -


#安装thehive
echo 'deb https://deb.thehive-project.org release main' | sudo tee -a /etc/apt/sources.list.d/thehive-project.list
sudo apt-get update
sudo apt-get install thehive4

#创建Indexing engine
chown -R thehive:thehive /opt/thp/thehive/files

mkdir /opt/thp/thehive/index
chown thehive:thehive -R /opt/thp/thehive/index
```



## 配置

为成功运行需要做好以下三个配置：

- Secret key configuration

```
查看文件内是否存在密钥。
/etc/thehive/secret.conf
```

- Database configuration

修改配置文件

```
vi /etc/thehive/application.conf
```

```
db {
  provider: janusgraph
  janusgraph {
    storage {
      backend: cql  #取消本行前面注释
      hostname: ["192.168.76.145"] # 改为本机IP
      #username: "<cassandra_username>"       # login to connect to database (if configured in Cassandra)
      #password: "<cassandra_passowrd"
      cql {
        cluster-name: thp       # cluster name
        keyspace: thehive           # name of the keyspace
        local-datacenter: datacenter1   # name of the datacenter where TheHive runs (relevant only on multi datacenter setup)
        # replication-factor: 2 # number of replica
        read-consistency-level: ONE
        write-consistency-level: ONE
      }
    }
  }
      ## Index configuration
    index.search {
      backend : lucene
      directory:  /opt/thp/thehive/index
    }
}
```

- File storage configuration


```
## Storage configuration
storage {
provider = localfs
localfs.location = /opt/thp/thehive/files
}
```

下面给出我的完整配置文件：

```
###
## Documentation is available at https://docs.thehive-project.org/thehive/
###

## Include Play secret key
# More information on secret key at https://www.playframework.com/documentation/2.8.x/ApplicationSecret
include "/etc/thehive/secret.conf"

## Database configuration
db.janusgraph {
  storage {
    ## Cassandra configuration
    # More information at https://docs.janusgraph.org/basics/configuration-reference/#storagecql
    backend: cql
    hostname: ["192.168.76.145"]
    # Cassandra authentication (if configured)
    // username: "thehive"
    // password: "password"
    cql {
      cluster-name: thp
      keyspace: thehive
    }
  }
  index.search {
    backend: lucene
    directory: /opt/thp/thehive/index
    # If TheHive is in cluster ElasticSearch must be used:
    // backend: elasticsearch
    // hostname: ["ip1", "ip2"]
    // index-name: thehive
  }

  ## For test only !
  # Comment the two lines below before enable Cassandra database
  storage.backend: berkeleyje
  storage.directory: /opt/thp/thehive/database
  // berkeleyje.freeDisk: 200 # disk usage threshold
}

## Attachment storage configuration
storage {
  ## Local filesystem
  provider: localfs
  localfs.location: /opt/thp/thehive/files

  ## Hadoop filesystem (HDFS)
  // provider: hdfs
  // hdfs {
  //   root: "hdfs://localhost:10000" # namenode server hostname
  //   location: "/thehive"           # location inside HDFS
  //   username: thehive              # file owner
  // }
}

## Authentication configuration
# More information at https://github.com/TheHive-Project/TheHiveDocs/TheHive4/Administration/Authentication.md
//auth {
//  providers: [
//    {name: session}               # required !
//    {name: basic, realm: thehive}
//    {name: local}
//    {name: key}
//  ]
# The format of logins must be valid email address format. If the provided login doesn't contain `@` the following
# domain is automatically appended
//  defaultUserDomain: "thehive.local"
//}

## CORTEX configuration
# More information at https://github.com/TheHive-Project/TheHiveDocs/TheHive4/Administration/Connectors.md
# Enable Cortex connector
// play.modules.enabled += org.thp.thehive.connector.cortex.CortexModule
// cortex {
//  servers: [
//    {
//      name: "local"                # Cortex name
//      url: "http://localhost:9001" # URL of Cortex instance
//      auth {
//        type: "bearer"
//        key: "***"                 # Cortex API key
//      }
//      wsConfig {}                  # HTTP client configuration (SSL and proxy)
//    }
//  ]
// }

## MISP configuration
# More information at https://github.com/TheHive-Project/TheHiveDocs/TheHive4/Administration/Connectors.md
# Enable MISP connector
// play.modules.enabled += org.thp.thehive.connector.misp.MispModule
// misp {
//  interval: 1 hour
//  servers: [
//    {
//      name = "local"            # MISP name
//      url = "http://localhost/" # URL or MISP
//      auth {
//        type = key
//        key = "***"             # MISP API key
//      }
//      wsConfig {}               # HTTP client configuration (SSL and proxy)
//    }
//  ]
//}

# Define maximum size of attachments (default 10MB)
//play.http.parser.maxDiskBuffer: 1GB
```



## 启动the hive

```
service thehive start
```

查看运行状态：

```
service thehive status
```

查看日志：

```
tail -f  /var/log/thehive/application.log
```



## 遇到的问题

我在执行`cqlsh localhost 9042`此命令的时候报错，排查了很久。

> Connection error: ('Unable to connect to any servers', {'127.0.0.1': error(10061, "Tried connecting to [('127.0.0.1', 9042)]. Last error: No connection could be made because the target machine actively refused it")})

可能是因为刚开始装了4.0版本，导致出了问题。

卸载重装。



## docker安装the hive
待更新。

