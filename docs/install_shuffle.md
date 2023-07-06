# 安装shuffle

## shuffle介绍

[Shuffle](https://shuffler.io/) is an automation platform for and by the community, focusing on accessibility for anyone to automate. Security operations is complex, but it doesn't have to be.

国外比较火的SOAR（安全编排自动化与响应）工作流自动化工具，可以连接很多其他系统。



## 安装shuffle

### 1.下载docker和docker-compose

安装docker,使用官方脚本安装。

```
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```



安装docker-compose

```
#下载二进制包，可以使用国内仓库替换，如果提示证书不安全，则加上--insecure参数。
sudo curl -L "https://github.com/docker/compose/releases/download/v2.2.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose --insecure

#将可执行权限应用于二进制文件：
sudo chmod +x /usr/local/bin/docker-compose

#创建软链
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

#测试是否安装成功：
docker-compose version
```

或手动安装docker-compose
```
从github下载https://github.com/docker/compose/releases
docker-compose-linux-x86_64
使用sftp上传到服务器的tmp目录下。
修改文件名
mv docker-compose-linux-x86_64 docker-compose
移动到目录下/usr/local/bin/
mv docker-compose /usr/local/bin/
sudo chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
docker-compose version
```

docker换源
`vim /etc/docker/daemon.json`

```
{
    "registry-mirrors" : [
    "https://registry.docker-cn.com",
    "https://kfwkfulq.mirror.aliyuncs.com",
    "https://2lqq34jg.mirror.aliyuncs.com",
    "https://pee6w651.mirror.aliyuncs.com",
    "http://hub-mirror.c.163.com",
    "https://docker.mirrors.ustc.edu.cn",
    "https://cr.console.aliyun.com",
    "https://mirror.ccs.tencentyun.com"
  ]
}
```

建议使用阿里云docker镜像加速服务：https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors
```
{
  "registry-mirrors": ["https://2aibpl9o.mirror.aliyuncs.com"]
}
```

重启使配置生效

```
systemctl daemon-reload
systemctl restart docker.service
```

查看换源是否成功：

```
docker info
```

### 2.下载shuffle

```
git clone https://github.com/Shuffle/Shuffle
cd Shuffle
```

### 3.创建数据库

```
mkdir shuffle-database
sudo chown -R 1000:1000 shuffle-database
```

### 4.启动

```
docker-compose up -d
```

这一步可能会报错：

```
error pulling image configuration: download failed after attempts=6: x509: certificate signed by unknown authority
```

解决方法：

```
将ghcr.io替换为ghcr.nju.edu.cn
参考：https://zhuanlan.zhihu.com/p/602161992
```

### 5.修改参数

我没改，也不影响正常使用。

Recommended for Opensearch to work well

```
sudo sysctl -w vm.max_map_count=262144             # https://www.elastic.co/guide/en/elasticsearch/reference/current/vm-max-map-count.html
```



## 初始化配置

参考：

```
https://github.com/Shuffle/Shuffle/blob/main/.github/install-guide.md#after-installation
```

### 1.打开web页面

[http://localhost:3001](http://localhost:3001/)

尽量使用https访问，否则后面Copy WebHook URL会报错。
[https://localhost:3443](http://localhost:3443/)

### 2.设置用户名和密码

### 3.根据提示创建后端数据库

这一步可能需要几分钟，有可能报错。

如果报错，请观察后端日志。适当调大服务器的内存。

### 4.导入APP和Workflows

https://shuffler.io/apps

官网提供了很多APP和workflows,可以直接导入，或者从官网下载后手动导入。需要在官网创建和登录账号。
有些内置的APP，比如Shuffle Tools,只能用github pull，无法在官网下载后导入。
对于一些第三方APP，比如Wazuh,只能从官网下载OpenAPI（yaml文件）,后再导入。

推荐安装的几个APP：
* FortiGate Firewall
* Wazuh
* Jira
* Shuffle
* github
* Crowdsec
* virustotal_v3
* thehive_5


## 遇到的问题

https://github.com/Shuffle/Shuffle/issues/1097

安装之后web界面正常，但是工作流一直显示执行中。

查看docker日志：

```
2023/07/05 08:42:43 [ERROR] Container create error: Error response from daemon: No such image: ghcr.io/shuffle/shuffle-worker:latest
2023/07/05 08:42:43 [WARNING] Execution ID ca66efea-a673-442d-b40d-e17a9210fe24 failed to deploy: Error response from daemon: No such image: ghcr.io/shuffle/shuffle-worker:latest
2023/07/05 08:42:43 [ERROR] Container create error: Error response from daemon: No such image: ghcr.io/shuffle/shuffle-worker:latest
2023/07/05 08:42:43 [WARNING] Execution ID 56f9a8ea-3838-4f92-a26d-c47289e9b5c2 failed to deploy: Error response from daemon: No such image: ghcr.io/shuffle/shuffle-worker:latest
2023/07/05 08:42:43 [ERROR] Container create error: Error response from daemon: No such image: ghcr.io/shuffle/shuffle-worker:latest
2023/07/05 08:42:43 [WARNING] Execution ID e0e152c1-6cbd-4f1d-a38b-bc5cac7e0a37 failed to deploy: Error response from daemon: No such image: ghcr.io/shuffle/shuffle-worker:latest
```

```
2023/07/05 08:31:06 [error] 14#14: *11 connect() failed (111: Connection refused) while connecting to upstream, client: 172.29.132.144, server: localhost, request: "POST /api/v1/hooks/webhook_d7e6ae11-319e-48f8-92ba-9732547a5d19 HTTP/1.1", upstream: "http://172.19.0.4:5001/api/v1/hooks/webhook_d7e6ae11-319e-48f8-92ba-9732547a5d19", host: "172.29.132.142:3443"
172.29.132.144 - - [05/Jul/2023:08:31:06 +0000] "POST /api/v1/hooks/webhook_d7e6ae11-319e-48f8-92ba-9732547a5d19 HTTP/1.1" 502 157 "-" "python-requests/2.25.1"
2023/07/05 08:31:06 [error] 14#14: *13 connect() failed (111: Connection refused) while connecting to upstream, client: 172.29.132.144, server: localhost, request: "POST /api/v1/hooks/webhook_d7e6ae11-319e-48f8-92ba-9732547a5d19 HTTP/1.1", upstream: "http://172.19.0.4:5001/api/v1/hooks/webhook_d7e6ae11-319e-48f8-92ba-9732547a5d19", host: "172.29.132.142:3443"
172.29.132.144 - - [05/Jul/2023:08:31:06 +0000] "POST /api/v1/hooks/webhook_d7e6ae11-319e-48f8-92ba-9732547a5d19 HTTP/1.1" 502 157 "-" "python-requests/2.25.1"
2023/07/05 08:31:07 [error] 14#14: *15 connect() failed (111: Connection refused) while connecting to upstream, client: 172.29.132.144, server: localhost, request: "POST /api/v1/hooks/webhook_d7e6ae11-319e-48f8-92ba-9732547a5d19 HTTP/1.1", upstream: "http://172.19.0.4:5001/api/v1/hooks/webhook_d7e6ae11-319e-48f8-92ba-9732547a5d19", host: "172.29.132.142:3443"
172.29.132.144 - - [05/Jul/2023:08:31:07 +0000] "POST /api/v1/hooks/webhook_d7e6ae11-319e-48f8-92ba-9732547a5d19 HTTP/1.1" 502 157 "-" "python-requests/2.25.1"
```

发现缺少镜像。

下载镜像：

```
docker pull frikky/shuffle:shuffle-tools_1.2.0

docker pull ghcr.io/shuffle/shuffle-worker:latest
或者
docker pull frikky/shuffle:shuffle-worker
```

如果用代理服务器下载，可能需要修改镜像名(tag命令)，使得与要求的一直。

```
docker tag example.example.com:8088/frikky/shuffle:shuffle-tools_1.2.0 frikky/shuffle:shuffle-tools_1.2.0
```



如果运行报错或一直显示执行中，就查看后端日志：

主要是shuffle-orborus日志和shuffle-backend日志。

查看时会提示查看Worker Container日志。查看此日志，排查报错原因。

一般是镜像问题或内部通信问题。

