# 安装shuffle

## shuffle介绍

[Shuffle](https://shuffler.io/) is an automation platform for and by the community, focusing on accessibility for anyone to automate. Security operations is complex, but it doesn't have to be.

国外比较火的SOAR（安全编排自动化与响应）工作流自动化工具，可以连接很多其他系统。



## 安装shuffle

### 1.下载docker和docker-compose

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
