MISP: 创建组织，创建用户。add auth_key



docker-compose down

替换docker-compose.yml中的key

```
      #- "--no-config-cortex"
      - "--cortex-port"
      - "9001"
      - "--cortex-keys"
      - "vuaNtCWlpewWXzz9JI1L/CD/HsnuWLPC" #remember to change this to your API key
```

创建配置文件目录

mkdir -p thehive/conf

cd thehive/conf

vim application.conf

```
play.modules.enabled += org.thp.thehive.connector.misp.MispModule
misp {
  interval: 1 hour
  servers: [
    {
      name = "MISP"
      url = "https://misp"
      auth {
        type = key
        key = "Yl1M7biMolOcb1qk7HCgNih3OjpUyUJqesDXtazB" #your API Key here
      }
      tags = ["tag1", "tag2", "tag3"]
      caseTemplate = "misp"
      includedTheHiveOrganisations = ["Morgan Maxwell"]
    }
  ]
}
```

```
    volumes:
      - ./thehive/conf/application.conf:/etc/thehive/application.conf 
```



取消MISP证书验证。
