# 搭建 the hive
方法1：官方5.X docker安装。

方法2：非官方docker安装
https://ls111.me/thehive-cortex-misp-installation-using-docker-compose/
https://github.com/ls111-cybersec/thehive-cortex-misp-docker-compose-lab11update
此博主也提供了一个开箱即用的ova镜像。

方法3：使用OVA镜像https://docs.strangebee.com/resources/demo/

## 使用docker安装
### 参考

https://chinyati.medium.com/the-hive-cortex-through-docker-installation-e50cbadb6cb0

https://wmvalente.medium.com/installing-misp-the-hive-and-cortex-part-5-d8a21c886fa8

https://github.com/TheHive-Project/Docker-Templates/tree/main/docker/thehive4-cortex31-shuffle

### compose.yml
注意，MISP配置中HOSTNAME的IP需要根据自己系统IP进行修改。
在TheHive中配置MISP服务器时，需要使用https的形式，并关闭安全检查
Do not check Certificate Autority
Disable hostname Verification

如果镜像无法顺利下载，可以用dockerhub的代替。并和compose.yml文件中的image保持一致。

```
docker pull strangebee/thehive:5.1
docker pull cassandra:4
docker pull elasticsearch:7.17.9
docker pull minio/minio
docker pull thehiveproject/cortex:3.1.7
docker pull coolacid/misp-docker:core-latest
docker pull mysql/mysql-server:5.7
docker pull coolacid/misp-docker:modules-latest
docker pull redis:5.0.6
```

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


docker-compose up -d 启动
查看运行状态：
```
oot@h4m5t:/home/h4m5t/Desktop/misp# docker ps
CONTAINER ID   IMAGE                                                  COMMAND                  CREATED        STATUS                  PORTS                                                       NAMES
19b4e9b3456f   strangebee/thehive:5.1                                 "/opt/thehive/entryp…"   40 hours ago   Up 40 hours             0.0.0.0:9000->9000/tcp                                      misp-thehive-1
03727c8b2035   coolacid/misp-docker:core-latest                       "/entrypoint.sh"         40 hours ago   Up 40 hours             0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp                    misp-misp-1
02eef218fc79   coolacid/misp-docker:modules-latest                    "/usr/local/bin/misp…"   40 hours ago   Up 40 hours                                                                         misp-misp-modules-1
a3595546433a   thehiveproject/cortex:3.1.7                            "/opt/cortex/entrypo…"   40 hours ago   Up 40 hours             0.0.0.0:9001->9001/tcp                                      misp-cortex-1
ddade05aae85   cassandra:4                                            "docker-entrypoint.s…"   40 hours ago   Up 40 hours             7000-7001/tcp, 7199/tcp, 9160/tcp, 0.0.0.0:9042->9042/tcp   misp-cassandra-1
a994b83800f8   docker.elastic.co/elasticsearch/elasticsearch:7.17.9   "/bin/tini -- /usr/l…"   40 hours ago   Up 40 hours             0.0.0.0:9200->9200/tcp, 9300/tcp                            misp-elasticsearch-1
faea8de5c62f   mysql/mysql-server:5.7                                 "/entrypoint.sh mysq…"   40 hours ago   Up 40 hours (healthy)   3306/tcp, 33060/tcp                                         misp-misp_mysql-1
6452b2ab987b   redis:5.0.6                                            "docker-entrypoint.s…"   40 hours ago   Up 40 hours             6379/tcp                                                    misp-redis-1
fe0f8f43b185   quay.io/minio/minio                                    "/usr/bin/docker-ent…"   40 hours ago   Up 40 hours             9000/tcp, 0.0.0.0:9002->9002/tcp                            misp-minio-1
```

查看端口开放情况
```
root@h4m5t:/home/h4m5t/Desktop/misp# netstat -ltpnd
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:9200            0.0.0.0:*               LISTEN      198646/docker-proxy 
tcp        0      0 0.0.0.0:9042            0.0.0.0:*               LISTEN      198659/docker-proxy 
tcp        0      0 0.0.0.0:9001            0.0.0.0:*               LISTEN      198980/docker-proxy 
tcp        0      0 0.0.0.0:9000            0.0.0.0:*               LISTEN      201100/docker-proxy 
tcp        0      0 0.0.0.0:9002            0.0.0.0:*               LISTEN      198628/docker-proxy 
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      199010/docker-proxy 
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      198996/docker-proxy 
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      693/systemd-resolve 
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN      402601/cupsd        
tcp6       0      0 ::1:631                 :::*                    LISTEN      402601/cupsd   
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





