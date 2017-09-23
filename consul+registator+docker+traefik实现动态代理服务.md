# 基本原理
* 使用 registartor 检测本地 docker 上所起的服务以及所暴露的端口，并注册到指定的 consul 机器上。
* 使用 consul 来做服务发现。
* 使用 traefik 做代理服务，后端有新的docker服务创建以后会自动检测到该服务，并为该服务自动生成域名。
      
# 快速搭建
## 1. 服务器环境
| IP地址       |  系统版本    |
| ----------- | ----------- |
| 192.168.0.2 | Ubuntu14.04 |
| 192.168.0.3 | Ubuntu14.04 |
| 192.168.0.4 | Ubuntu14.04 |
      
## 2. consul的搭建
node1 启动脚本：
```bash
#!/bin/bash
docker run -d --net=host --restart=always --name consul consul:v0.6.4 \
  agent -server \
  -node=node1 \
  -client=192.168.0.2 \
  -bind=192.168.0.2 \
  -retry-join=192.168.0.3 \
  -retry-join=192.168.0.4
```

node2 启动脚本:
```bash
#!/bin/bash
docker run -d --net=host --restart=always --name consul consul:v0.6.4 \
  agent -server \
  -node=node2 \
  -client=192.168.0.3 \
  -bind=192.168.0.3 \
  -retry-join=192.168.0.2 \
  -retry-join=192.168.0.4
```

node3 启动脚本:
```bash
#!/bin/bash
docker run -d --net=host --restart=always --name consul consul:v0.6.4 \
  agent -server \
  -node=node3 \
  -client=192.168.0.4 \
  -bind=192.168.0.4 \
  -retry-join=192.168.0.2 \
  -retry-join=192.168.0.3
```


## 3. registrator搭建
启动脚本:
```Bash
#!/bin/bash
docker run -d --net=host --restart=always -v /var/run/docker.sock:/tmp/docker.sock --name registrator gliderlabs/registrator:v7 \
  -ip 192.168.0.2 \
  consul://192.168.0.2:8500
```

## 4. traefik 搭建
配置文件 traefik.toml
```bash
#####################################
# Global congifuration
#####################################

# 基础设置
graceTimeOut = 10
logLevel = "INFO"
insecureSkipVerify = true
defaultEntryPoints = ["http", "https"]
traefikLogsFile = "/var/log/traefik.log"
accessLogsFile = "/var/log/access.log"
errorLogsFile = "/var/log/error.log"

# 开启http和https，这里的https证书使用的是 certbot(https://github.com/certbot/certbot) 生成的。
[entryPoints]
 [entryPoints.http]
 address = ":80"
 [entryPoints.https]
 address = ":443"
   [entryPoints.https.tls]
	[[entryPoints.https.tls.certificates]]
	CertFile = "/etc/ssl/live/www.zqifei.com/fullchain.pem"
	KeyFile = "/etc/ssl/live/www.zqifei.com/privkey.pem"

# 这里我为traefik服务添加了域名：traefik.zqifei.com
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
       rule = "Host:traefik.zqifei.com"

# web端访问服务的端口号
[web]
address = ":8080"
ReadOnly = false
 [web.statistics]
    RecentErrors = 10
    [web.auth.basic]
       users = ["zhuqifei:$1$4K.YXlm1$IkFqpMv6Vh7Yro4Qj5HUB1"]

# consul节点及域名，这里需要将 *.zqifei.com 域名指定到部署traefik这台主机上。
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

启动脚本：
```bash
#!/bin/bash

docker_run() {
  docker run --restart=always -d -p 8080:8080 -p 80:80 -p 443:443 --name traefik \
  -v $PWD/traefik.toml:/etc/traefik/traefik.toml \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /data/volumes/traefik:/var/log \
  -v /etc/letsencrypt:/etc/ssl \
  traefik:v1.2.3
}

NAME=`docker ps -a | grep traefik | awk '{print $NF}'`

if [ "$NAME" = "traefik" ]; then
	docker rm -f traefik;
	echo "traefik have delete!";
  docker_run;
  echo "traefik start running!";
else
	echo "traefik not running!";
  docker_run;
  echo "traefik start running!";
fi
```
访问服务：
通过浏览器访问 https://traefik.zqifei.com 会出现账号密码验证，输入在配置文件设置的密码即可登录进去。
更多 traefik 配置请参考 [traefik官方文档](https://docs.traefik.io/)。
