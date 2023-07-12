# 安装thehive4
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
sudo apt-get install tree
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



## docker安装the hive4
### 文件目录
文件路径如下：命令查看：`tree -L 3`
```
.
├── docker-compose.yml
└── vol
    ├── cortex
    │   └── application.conf
    └── thehive
        ├── application.conf
```

### .env
和docker-compose在同级目录下：
```
CORTEX_KEY=[WWOgwpjYZvfkD6vgDh0VnWFfApM7bOF5]
COMPOSE_PROJECT_NAME=FreeSIEM
JOB_DIRECTORY=/opt/cortex/jobs
```

### docker-compose
docker-compose.yml

```
version: '3.8'
services:
  ## Cortex
  elasticsearch:
    image: 'elasticsearch:7.11.1'
    container_name: elasticsearch
    restart: unless-stopped
    ports:
      - '0.0.0.0:9200:9200'
    environment:
      - http.host=0.0.0.0
      - discovery.type=single-node
      - cluster.name=hive
      - script.allowed_types= inline
      - thread_pool.search.queue_size=100000
      - thread_pool.write.queue_size=10000
      - gateway.recover_after_nodes=1
      - xpack.security.enabled=false
      - bootstrap.memory_lock=true
      - ES_JAVA_OPTS=-Xms256m -Xmx256m
    ulimits:
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - './vol/elasticsearch/data:/usr/share/elasticsearch/data'
      - './vol/elasticsearch/logs:/usr/share/elasticsearch/logs'
  cortex:
    image: 'thehiveproject/cortex:latest'
    container_name: cortex
    restart: unless-stopped
    command:
      --job-directory ${JOB_DIRECTORY}
    environment:
      - 'JOB_DIRECTORY=${JOB_DIRECTORY}'
    volumes:
      - './vol/cortex/application.conf:/etc/cortex/application.conf'
      - '${JOB_DIRECTORY}:${JOB_DIRECTORY}'
      - '/var/run/docker.sock:/var/run/docker.sock'
    depends_on:
      - elasticsearch
    ports:
      - '0.0.0.0:9001:9001'

  ## TheHive
  thehive:
    image: 'thehiveproject/thehive4:latest'
    container_name: 'thehive4'
    depends_on:
      - cassandra
    ports:
      - '0.0.0.0:9000:9000'
    volumes:
      - ./vol/thehive/application.conf:/etc/thehive/application.conf
      - ./vol/thehive/data:/opt/thp/thehive/data
      - ./vol/thehive/index:/opt/thp/thehive/index
    networks:
      - default
    command: '--no-config --no-config-secret'

  cassandra:
    image: 'cassandra:3.11'
    container_name: cassandra
    hostname: cassandra
    environment:
      - MAX_HEAP_SIZE=1G
      - HEAP_NEWSIZE=1G
      - CASSANDRA_CLUSTER_NAME=thp
    volumes:
      - './vol/cassandra/data:/var/lib/cassandra/data'

networks:
  default: null
```


### Cortex配置文件application.conf

```
## SECRET KEY
#
# The secret key is used to secure cryptographic functions.
#
# IMPORTANT: If you deploy your application to several  instances,  make
# sure to use the same key.
play.http.secret.key="CortexTestPassword"

## ElasticSearch
search {
  index = cortex
  uri = "http://elasticsearch:9200"
}

## Cache
cache.job = 10 minutes

job {
  runner = [docker, process]
}

## ANALYZERS
analyzer {
  urls = [
    "https://download.thehive-project.org/analyzers.json"
    #"/absolute/path/of/analyzers"
  ]
}

# RESPONDERS
responder {
  urls = [
    "https://download.thehive-project.org/responders.json"
    #"/absolute/path/of/responders"
  ]
}
```

### Thehive配置文件application.conf
```
# Secret Key
# The secret key is used to secure cryptographic functions.
# WARNING: If you deploy your application on several servers, make sure to use the same key.
play.http.secret.key="dgngu325mbnbc39cxas4l5kb24503836y2vsvsg465989fbsvop9d09ds6df6"

# JanusGraph
db {
  provider: janusgraph
  janusgraph {
    storage {
      backend: cql
      hostname: ["cassandra"]

      cql {
        cluster-name: thp       # cluster name
        keyspace: thehive           # name of the keyspace
        read-consistency-level: ONE
        write-consistency-level: ONE
      }
    }
    
    ## Index configuration
    index {
      search {
        backend: lucene
        directory: /opt/thp/thehive/index
      }
    }
  }
}

storage {
   provider: localfs
   localfs.location: /opt/thp/thehive/data
}

play.http.parser.maxDiskBuffer: 50MB
play.modules.enabled += org.thp.thehive.connector.cortex.CortexModule
cortex {
  servers = [
    {
      name = local
      url = "http://cortex:9001"
      auth {
        type = "bearer"
        key = "4iDg0yQtTnfRQm9hms2ZrMkZnRlQPrtU"
      }
    }
  ]
  # Check job update time intervalcortex
  refreshDelay = 5 seconds
  # Maximum number of successive errors before give up
  maxRetryOnError = 3
  # Check remote Cortex status time interval
  statusCheckInterval = 30 seconds
}
```

### 启动：docker-compose up -d

运行后的目录结构：
```
.
├── docker-compose.yml
└── vol
    ├── cassandra
    │   └── data
    ├── cortex
    │   └── application.conf
    ├── elasticsearch
    │   ├── data
    │   └── logs
    └── thehive
        ├── application.conf
        ├── data
        └── index
```

docker ps查看服务运行状态：
```
root@TheHive:/home/thehive4# docker ps
CONTAINER ID   IMAGE                            COMMAND                  CREATED      STATUS      PORTS                                         NAMES
da52fee5183d   thehiveproject/cortex:latest     "/opt/cortex/entrypo…"   4 days ago   Up 4 days   0.0.0.0:9001->9001/tcp                        cortex
4b5691796e2b   thehiveproject/thehive4:latest   "/opt/thehive/entryp…"   4 days ago   Up 4 days   0.0.0.0:9000->9000/tcp                        thehive4
f015f074e82d   cassandra:3.11                   "docker-entrypoint.s…"   4 days ago   Up 4 days   7000-7001/tcp, 7199/tcp, 9042/tcp, 9160/tcp   cassandra
32c2682ddd0a   elasticsearch:7.11.1             "/bin/tini -- /usr/l…"   4 days ago   Up 4 days   0.0.0.0:9200->9200/tcp, 9300/tcp              elasticsearch
```

### 遇到的问题
thehive4的报错

```
[error] a.a.OneForOneStrategy [|] Unable to provision, see the following errors:

1) Error in custom provider, Configuration error: Configuration error[
The application secret has not been set, and we are in prod mode. Your application is not secure.
To set the application secret, please read http://playframework.com/documentation/latest/ApplicationSecret
```

发现是application.conf权限问题，增加所有用户的可读权限。

```
chmod 644 application.conf
```



elasticsearch启动失败，解决方法：https://github.com/TheHive-Project/Docker-Templates/issues/24

chown 1000:1000  /vol/elasticsearch/data

chown 1000:1000  /vol/elasticsearch/logs



继续报错：

```
[error] o.t.s.u.Retry [|] An error occurs
java.lang.IllegalArgumentException: Could not instantiate implementation: org.janusgraph.diskstorage.lucene.LuceneIndex
```

参考：https://github.com/TheHive-Project/TheHive/issues/1863



需要修改vol/thehive下的文件夹及文件权限。

```
chown -R 1000:1000 data
chown -R 1000:1000 index
```

查看实时日志：

```
docker logs 4b5691796e2b --follow
```

修改之后不再报错。

### 联动Cortex
在Cortex中创建新组织和新组织下的用户，复制API-Key，复制到thehive的配置文件application.conf中，重启服务即可。
