## kubernetes安装文档
### 服务环境

#### 主机列表

| 主机名   | 主机IP         | slave        | 系统版本        |
| ----- | ------------ | ------------ | ----------- |
| node1 | 192.168.1.10 | master/slave | Ubuntu14.04 |
| node2 | 192.168.1.11 | slave        | Ubuntu14.04 |

#### 服务列表

| 服务名        | 版本号    |
| ---------- | ------ |
| go         | 1.7.5  |
| kubernetes | 1.5.6  |
| etcd       | 2.3.1  |
| flanneld   | 0.5.5  |
| docker     | 1.12.5 |



### 安装步骤

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

修改 kubernetes/cluster/ubuntu/config-default.sh 脚本

```shell
# 修改以下配置
export nodes=$(nodes:-"root@192.168.1.10, root@192.168.1.11")
roles=${roles:-"ai i"}
export NUM_NODES=${NUM_NODES:-2}
export SERVICE_CLUSTER_IP_RANGE=${SERVICE_CLUSTER_IP_RANGE:--192.168.3.0/24}
export FLANNEL_NET=%{FLANNEL_NET:-"172.16.0.0/16"}
export ADMISSION_CONTROL=NamespaceLifecycle,LimitRanger,DefaultStorageClass,ResourceQuota
SERVICE_NODE_PORT_RANGE=${SERVICE_NODE_PORT_RANGE:-"1-65535"}
SSH_OPTS-oPort=1022 -oStricHostKeyChecking=no -oUserKnownHostsFile=/dev/null -oLogLevel=ERROR
```

修改 kubernetes/cluster/ubuntu/download-release.sh 脚本

```Bash

注释掉清理动作,添加个echo，否则会报错
function cleanup {
  # cleanup work
  # rm -rf flannel* kubernetes* etcd* binaries out
  echo "cleanup work"
}
# trap cleanup SIGHUP SIGINT SIGTERM
 
注释掉下载etcd
# curl -L https://github.com/coreos/etcd/releases/download/v${ETCD_VERSION}/${ETCD}.tar.gz -o etcd.tar.gz
 
注释掉获取k8s版本号,并制定version版本
# KUBE_VERSION=$(get_latest_version_number | sed 's/^v//')
KUBE_VERSION=1.5.6
 
注释掉删除动作
# rm -rf flannel* kubernetes* etcd* out
```

将本地下载好的安装包 copy 到指定位置

```Bash
$ cp intall_tar/* /root/kubernetes/kubernetes/cluster/ubuntu/
```

修改 kubernetes/cluster/ubuntu/util.sh 脚本

```Bash
修改common.sh路径，将 source "${KUBE_ROOT}/cluster/common.sh" 改为
# source "${KUBE_ROOT}/cluster/common.sh"
source "/root/kubernetes/kubernetes/cluster/common.sh"
source "/root/kubernetes/kubernetes/cluster/ubuntu/config-default.sh"
 
修改create-etcd-opts函数
function create-etcd-opts() {
  cat <<EOF > ~/kube/default/etcd
 ETCD_OPTS="\
 -name infra\
 -listen-client-urls http://127.0.0.1:4001 \
 -advertise-client-urls http://127.0.0.1:4001"
EOF
}
```

修改 root/kubernetes/kubernetes/cluster/ubuntu/master/init_scripts/etcd 脚本

```Bash
# 添加以下注释
ETCD_START="start-stop-daemon \
--start \
--background \
--quiet \
--exec $ETCD \
--make-pidfile \
--pidfile $ETCD_PIDFILE \
-- $ETCD_OPTS # \
# >> $ETCD_LOGFILE 2>&1"
```

#### 启动kubernetes服务

**清空防火墙规则**

```Bash
$ iptables -F
```

**拷贝kubernetes目录**

```Bash
$ scp -P 1022 /root/kubernetes root@192.168.1.11:/root
```

**执行启动脚本**

```Bash
$ cd /root/kubernetes/kuberentes/cluster
$ KUBERNETES_PROVIDER=ubuntu ./kube-up.sh #这里也可以把KUBERNETES_PROVIDER=ubuntu 变量配置到/etc/profile文件下
 
Cluster validation succeeded
Done, listing cluster services:
 
Kubernetes master is running at http://192.168.1.10:8080
```

**修改kubectl命令位置**

部署完成后 kubectl 命令会存放在 kubernetes/cluster/ubuntu/binaties 目录中，可以将其拷贝到系统 /usr/local/bin 目录下。

```
$ cd kubernetes/cluster/ubuntu/binaties
$ cp kubectl /usr/local/bin
$ cd
$ kubectl get nodes
NAME             STATUS    AGE
192.168.1.10     Ready     20m
192.168.1.11     Ready     9m
```

**修改dns启动脚本**

```yaml

$ cd kubernetes/cluster/addons/dns
$ vim skydns-rc.yaml.in
containers:
- name: kubedns
  args:
  - --domain={{ pillar['dns_domain'] }}.
  - --dns-port=10053
  - --config-map=kube-dns
  - --kube_master_url=http://192.168.1.10:8080 # 添加该部分
```

**修改dashboard启动脚本**

```yaml

```

**启动dns和dashboard服务**

```Bash
$ cd kubernetes/cluster/ubuntu
$ KUBERNETES_PROVIDER=ubuntu ./deployAddons.sh
Creating kube-system namespace...
The namespace 'kube-system' is already there. Skipping.
 
Deploying DNS on Kubernetes
No resources found.
deployment "kube-dns" created
service "kube-dns" created
Kube-dns controller and service are successfully deployed.
 
No resources found.
Creating Kubernetes Dashboard controller
deployment "kubernetes-dashboard" created
Creating Kubernetes Dashboard service
service "kubernetes-dashboard" created
```

**查看服务启动情况**

```Bash

$ kubectl get pods --namespace=kube-system
NAME                                   READY     STATUS             RESTARTS   AGE
kube-dns-3389952575-dsp6s              4/4       Running            0          29m
kubernetes-dashboard-561285145-ngg0f   1/1       Running            0          28m
```