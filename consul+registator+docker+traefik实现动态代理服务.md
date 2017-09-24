# 编者的话
* Consul 是一个支持多数据中心分布式高可用的服务发现和配置共享的服务软件，由 HashiCorp 公司用 Go 语言开发。本文将介绍如何使用 Consul 将多个 Docker 容器组合起来，提供一套高可用可扩展的 web 服务。
* 之前的话我尝试使用 Docker、consul、registrator、consul-temlate 和 haproxy 来管理我底层的 Docker 集群，并实现自动配置服务代理。但在一段使用过程以后，发现只要底层的 Docker 服务有增删的操作，都会重启 haproxy 代理服务，如果操作频繁的话有时会出现代理服务重启异常，导致所有的服务无法正常访问的现象。经过调研和尝试，发现 traefik 可以很好的避免这类现象的发生。

# 快速搭建
## 1. 服务器环境
| 主机名   | IP地址        | 系统版本        | docker版本 |
| ----- | ----------- | ----------- | -------- |
| node1 | 192.168.0.2 | Ubuntu14.04 | 1.12.3   |
| node2 | 192.168.0.3 | Ubuntu14.04 | 1.12.3   |
| node3 | 192.168.0.4 | Ubuntu14.04 | 1.12.3   |

## 2. consul的搭建
consul 是一个拥有 HTTP API 和 DNS 的服务，它还包括很多其他的功能如：服务健康检查、跨主机集群构建和"key-value"存储。
### consul服务的部署
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
### consul DNS配置
#### 配置DNS服务
最好在每台主机都配置上 DNS 服务，IP 指向本机 IP 地址。
```bash
$ apt-get install dnsmasq
```
```bash
$ vim /etc/dnsmasq.d/consul
# IP需要改为自己的本地IP
server=/consul/192.168.0.2#8600
```
#### 修改Docker内部DNS
修改 Docker 配置文件，将 DNS 指向新配置的 DNS 服务IP。
```bash
$ vim /etc/default/docker
DOCKER_OPTS=" --storage-driver=aufs --dns=192.168.0.2"
```
#### 测试DNS服务
在集群任意一台服务器上起一个 redis 容器，启动时需要指定环境变量 "SERVER_NAME=redis" (可以自定义名)。
```bash
$ ping redis.service.dc1.consul
PING redis.service.dc1.consul (192.168.0.2) 56(84) bytes of data.
64 bytes from node1 (192.168.0.2): icmp_seq=1 ttl=64 time=0.021 ms
64 bytes from node1 (192.168.0.2): icmp_seq=1 ttl=64 time=0.021 ms
```
可以正确找到 redis 服务所在的服务器 IP 地址即可。


## 3. registrator搭建
使用 registartor 检测本地 docker 上所起的服务以及所暴露的端口，并注册到指定的 consul 机器上。

node1 启动脚本:

```Bash
#!/bin/bash
docker run -d --net=host --restart=always -v /var/run/docker.sock:/tmp/docker.sock --name registrator
  gliderlabs/registrator:v7 \
  -ip 192.168.0.2 \
  consul://192.168.0.2:8500
```
node2 启动脚本:
```Bash
#!/bin/bash
docker run -d --net=host --restart=always -v /var/run/docker.sock:/tmp/docker.sock --name registrator
  gliderlabs/registrator:v7 \
  -ip 192.168.0.3 \
  consul://192.168.0.3:8500
```

node2 启动脚本:
```Bash
#!/bin/bash
docker run -d --net=host --restart=always -v /var/run/docker.sock:/tmp/docker.sock --name registrator
  gliderlabs/registrator:v7 \
  -ip 192.168.0.3 \
  consul://192/168.0.4:8500
```

## 4. traefik搭建
### 配置traefik服务
配置文件 traefik.toml，这里我选择放在 node1 服务器上。
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
### traefik界面访问
通过浏览器访问 https://traefik.zqifei.com (记得将域名解析到代理服务器上)会出现账号密码验证，输入在配置文件设置的密码即可登录进去。更多 traefik 配置请参考 [traefik官方文档](https://docs.traefik.io/)。
![20170924150626425397349.png](http://cdn.zqifei.com/20170924150626425397349.png)