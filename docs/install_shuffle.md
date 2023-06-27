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

### 2.设置用户名和密码

### 3.根据提示创建后端数据库

这一步可能需要几分钟，有可能报错。

如果报错，请观察后端日志。适当调大服务器的内存。

### 4.导入APP和Workflows

https://shuffler.io/apps

官网提供了很多APP和workflows,可以直接导入，或者从官网下载后手动导入。
