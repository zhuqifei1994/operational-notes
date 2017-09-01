## kubernetes安装文档
### 服务环境

####主机列表

| 主机名   | 主机IP         | slave        |
| ----- | ------------ | ------------ |
| node1 | 192.168.1.10 | master/slave |
| node2 | 192.168.1.11 | slave        |

####服务列表

| 服务名        | 版本号    |
| ---------- | ------ |
| go         | 1.7.5  |
| kubernetes | 1.5.6  |
| etcd       | 2.3.1  |
| flanneld   | 0.5.5  |
| docker     | 1.12.5 |



###安装步骤

#### 服务器准备

```Bash
1. 节点已经安装了docker版本1.2+ 和 bridge-utils
2. 所有机器可以相互通信，主节点需要连接到internet才能下载所需文件，子节点不需要
3. 系统为ubuntu14.04LTS64位服务器
4. 所有远程服务器都可以通过密钥认证ssh登录，无需密码
5. 所有机器上的远程用户都使用/bin/bash作为其登录shell，并具有sudo权限
```

#### 安装go环境

使用gvm工具安装go环境

```Bash
$ bash < <(curl -s -S -L https://raw.githubusercontent.com/moovweb/gvm/master/binscripts/gvm-installer)  #如果使用zsh请将 bash 改为 zsh
$ sudo apt-get install curl git mercurial make binutils bison gcc build-essential                        #安装服务依赖
$ gvm install go1.4                                                                                      #需要先安装go1.4版本
$ gvm use go1.4                                                                                          #使用go1.4版本
$ gvm install go1.7.5                                                                                    #升级go版本到1.7.5
$ gvm use go1.7.5   
```

#### 配置主机免密登录

生成主机公钥，将所有服务的 id_rsa.pub 内容拷贝到 master 节点的 authorized_keys 内。

```Bash
$ ssh-keygen
$ ls ~/.ssh/
authorized_keys  id_rsa  id_rsa.pub  known_hosts
```

测试主机间登录，无需输入密码直接登录即可

```Bash
$ ssh root@192.168.10.11
```

#### kubernetes安装

**安装依赖包**

在各个节点安装 bridge-utils

```Bash
$ apt-get update
$ apt-get install bridge-utils
```

**下载kubernetes安装包**

```Bash
$ wget https://github.com/kubernetes/kubernetes/releases/download/v1.5.6/kubernetes.tar.gz
$ tar -zxvf kubernetes.tar.gz
```

```Bash
$ cd kubernetes/cluster/ubuntu/minion/init_conf
$ kubelet.conf
将 exec "$KUBELET" $KUBELET_OPTS 改为：
exec "$KUBELET" $KUBELET_OPTS --pod-infra-container-image="docker.io/kubernetes/pause"
```

**解压salt**

```Bash
$ tar zxf kubernetes/server/kubernetes-salt.tar.gz
$ cp -r kubernetes/saltbase kubernetes/cluster/saltbase
```

**修改默认的安装脚本**

