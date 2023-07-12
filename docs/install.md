# How to install
## this is a test.
数据流向：
1. 从wazuh直接到thehive
2. 从wazuh到shuffle再到thehive.
3. 从ELK到thehive
4. 从ELK到到shuffle再到thehive.
5. 从ELK到Wazuh再到thehive
6. 从ELK到Wazuh再到shuffle再到thehive
7. ELK到Shuffle，Wazuh到Shuffle同时存在。




### helllo
Thehive和Cortex都需要用admin账号登录，创建新的组织，在新组织下创建新的账号（Org-Admin权限），再进行后续操作。admin账户用来做管理。

Cortex上的analyzers可以添加MISP和VirusTotal等集成。MalwareBazaar. abuseipdb等。

在ElasticSearch中，创建连接器。webhook.
参考：https://github.com/archanchoudhury/SOC-OpenSource/blob/main/integration/integration.md
在规则rules（需要enable）触发告警Alert时，webhook会推送到thehive
测试内容如下：

```
{
"title" : "My Auto case",
"description" : "A VPN user has connected from a foreign country"
"tlp" : 3,
"tags" : ["automatic", "creation"]
}
```

Connectors可能需要License的激活。
解决⽅法
使⽤破解版，激活License
每次到期之后重新部署，试⽤期会刷新
使⽤Wazuh
ElastAlert


Wazuh和thehive集成
https://github.com/crow1011/wazuh2thehive
https://wazuh.com/blog/using-wazuh-and-thehive-for-threat-protection-and-incident-response/
https://github.com/TheHive-Project/TheHive4py

