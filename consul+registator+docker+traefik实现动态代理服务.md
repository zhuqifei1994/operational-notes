# 基本原理
使用 registartor 检测本地 docker 上所起的服务并注册到指定的 consul 机器上。使用 consul 来做服务发现。使用 traefik 做代理服务。
# 快速搭建
## 1. 服务器环境
| IP地址       |  系统版本    |
| ----------- | ----------- |
| 192.168.0.2 | Ubuntu14.04 |
| 192.168.0.3 | Ubuntu14.04 |
| 192.168.0.4 | Ubuntu14.04 |
## 2. consul的搭建
启动脚本：
```bash
#!/bin/bash
docker run -d --net=host --name consul consul:v0.6.4 \
  agent -server \
  -node=node1 \
  -client=192.168.0.2 \
  -bind=192.168.0.2 \
  -retry-join=192.168.0.3 \
  -retry-join=192.168.0.4
```

## 3. registrator搭建

启动脚本

```Bash
#!/bin/bash
docker run -d --net=host -v /var/run/docker.sock:/tmp/docker.sock --name registrator gliderlabs/registrator:v7 \
  -ip 192.168.0.2 \
  consul://192.168.0.2:8500
```

## 4. traefik 搭建
配置文件 traefik.toml
```toml
#####################################
# Global congifuration
#####################################

graceTimeOut = 10
logLevel = "INFO"
insecureSkipVerify = true
defaultEntryPoints = ["http", "https"]
traefikLogsFile = "/var/log/traefik.log"
accessLogsFile = "/var/log/access.log"
errorLogsFile = "/var/log/error.log"

[entryPoints]
 [entryPoints.http]
 address = ":80"
 [entryPoints.https]
 address = ":443"
   [entryPoints.https.tls]
	[[entryPoints.https.tls.certificates]]
	CertFile = "/etc/ssl/live/www.zqifei.com/fullchain.pem"
	KeyFile = "/etc/ssl/live/www.zqifei.com/privkey.pem"

[file]
[backends]
 [backends.backend1]
  [backends.backend1.circuitbreaker]
     expression = "NetworkErrorRatio() > 0.5"
  [backends.backend1.maxconn]
     amount = 10
     extractorfunc = "request.host"
  [backends.backend1.LoadBalancer]
     method = "drr"
  [backends.backend1.servers.server1]
     url = "http://192.168.0.2:8080/"

[frontends]
    [frontends.frontend1]
    backend = "backend1"
    entrypoints = ["http", "https"]
       [frontends.frontend1.routes.zqifei]
       rule = "Host:www.zqifei.com"

[web]
address = ":8080"
ReadOnly = false
 [web.statistics]
    RecentErrors = 10
    [web.auth.basic]
       users = ["zhuqifei:$1$4K.YXlm1$IkFqpMv6Vh7Yro4Qj5HUB1"]

[consul]
  endpoint = "192.168.0.2:8500"
  domain = "zqifei.com"
  watch = true
  prefix = "traefik"

[consulCatalog]
  endpoint = "192.168.0.2:8500"
  domain = "zqifei.com"
  prefix = "traefik"
```