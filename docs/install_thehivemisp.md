# 安装Thehive+MISP

## docker-compose

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

  ## MISP
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
      - default

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
      - default

  redis:
    image: redis:5.0.6
    networks:
      - default

  misp-modules:
    image: coolacid/misp-docker:modules-latest
    environment:
      - "REDIS_BACKEND=redis"
    depends_on:
      - redis
      - misp_mysql
    networks:
      - default
volumes:
  mispsqldata:

networks:
  default: null
```

## .env

```
CORTEX_KEY=[]
COMPOSE_PROJECT_NAME=FreeSIEM
JOB_DIRECTORY=/opt/cortex/jobs
```



## vol

### 文件目录

文件路径如下：命令查看：`tree -L 3`

```
.
├── .env
├── docker-compose.yml
└── vol
    ├── cortex
    │   └── application.conf
    └── elasticsearch
        └── data
        └── logs
    └── thehive
        ├── application.conf
        └── data
        └── index
```

### 修改权限

```
chown -R 1000:1000 elasticsearch
chown -R 1000:1000 thehive
chown -R 1000:1000 cortex
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

```
CONTAINER ID   IMAGE                                 COMMAND                  CREATED          STATUS                    PORTS                                         NAMES
06d8e0c4246a   thehiveproject/thehive4:latest        "/opt/thehive/entryp…"   17 minutes ago   Up 17 minutes             0.0.0.0:9000->9000/tcp                        thehive4
8e89e636f930   coolacid/misp-docker:modules-latest   "/usr/local/bin/misp…"   17 minutes ago   Up 17 minutes                                                           freesiem-misp-modules-1
399f0e6e4d55   coolacid/misp-docker:core-latest      "/entrypoint.sh"         17 minutes ago   Up 17 minutes             0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp      freesiem-misp-1
4f3de964e586   thehiveproject/cortex:latest          "/opt/cortex/entrypo…"   17 minutes ago   Up 17 minutes             0.0.0.0:9001->9001/tcp                        cortex
cf9975c728f2   redis:5.0.6                           "docker-entrypoint.s…"   17 minutes ago   Up 17 minutes             6379/tcp                                      freesiem-redis-1
abbafb6a13d7   cassandra:3.11                        "docker-entrypoint.s…"   17 minutes ago   Up 17 minutes             7000-7001/tcp, 7199/tcp, 9042/tcp, 9160/tcp   cassandra
b22fa07b1444   mysql/mysql-server:5.7                "/entrypoint.sh mysq…"   17 minutes ago   Up 17 minutes (healthy)   3306/tcp, 33060/tcp                           freesiem-misp_mysql-1
5e6090a3b480   elasticsearch:7.11.1                  "/bin/tini -- /usr/l…"   17 minutes ago   Up 17 minutes             0.0.0.0:9200->9200/tcp, 9300/tcp              elasticsearch
```

