## 1.安装 LDAP

```Bash
$ apt-get update
$ apt-get install slapd ldap-utils
```

安装过程会提示设置LDAP管理员账号密码：

![20170906150466346771942.png](http://cdn.zqifei.com/20170906150466346771942.png)

再次确认密码：

![20170906150466350381346.png](http://cdn.zqifei.com/20170906150466350381346.png)

## 2.配置 LDAP

打开 `/etc/ldap/ldap.conf` 文件。安装以下内容进行配置：

```bash
$ vim /etc/ldap/ldap.conf
#
# LDAP Defaults
#

# See ldap.conf(5) for details
# This file should be world readable but not world writeable.

BASE   dc=domain.com,dc=com
URL    ldap://:389

#SIZELIMIT      12
#TIMELIMIT      15
#DEREF          never

# TLS certificates (needed for GnuTLS)
TLS_CACERT      /etc/ssl/certs/ca-certificates.crt
```

执行以下命令进行配置：

```Bash
$ dpkg-reconfigure slapd
```

以下界面选择 "NO" 后按Enter继续:

![2017090615046636189543.png](http://cdn.zqifei.com/2017090615046636189543.png)

输入DNS domain名称：

![20170906150466894882341.png](http://cdn.zqifei.com/20170906150466894882341.png)

输入组织名称:

![20170906150466902698347.png](http://cdn.zqifei.com/20170906150466902698347.png)

输入LDAP管理员密码:

![20170906150466369835125.png](http://cdn.zqifei.com/20170906150466369835125.png)

再次确认LDAP管理员密码:

![20170906150466372510553.png](http://cdn.zqifei.com/20170906150466372510553.png)

选择HDB数据库:

![2017090615046637432197.png](http://cdn.zqifei.com/2017090615046637432197.png)

选择删除LDAP服务时自动删除数据库:

![20170906150466377023274.png](http://cdn.zqifei.com/20170906150466377023274.png)

选择删除之前的数据库:

![20170906150466379288300.png](http://cdn.zqifei.com/20170906150466379288300.png)

选择 "No" 后，LDAP服务配置完成并启动。

![2017090615046638157804.png](http://cdn.zqifei.com/2017090615046638157804.png)

### 3.测试LDAP服务

输入 `ldapsearch -x` 会看到以下输出:

![20170906150466917589335.png](http://cdn.zqifei.com/20170906150466917589335.png)

### 4.修改LDAP配置

拷贝DB_CONFIG 和 slapd.conf两个文件到 /var/lib/ldap 和 /etc/ldap 目录下

```bash
$ cp -rp /usr/share/slapd/DB_CONFIG /var/lib/slapd/
$ cp -rp /usr/share/slapd/slapd.conf /etc/ldap/
```

修改rsync.conf记录ldap日志

```bash
$ echo "local4.debug   /var/log/ldap.log" >> /etc/rsyslog.conf
$ /etc/init.d/rsyslog restart
```

修改配置文件/etc/ldap/slapd.conf

![20170808150216644151461.png](http://oe1qcatok.bkt.clouddn.com/20170808150216644151461.png)

![20170808150216650616321.png](http://oe1qcatok.bkt.clouddn.com/20170808150216650616321.png)

![20170906150466927049313.png](http://cdn.zqifei.com/20170906150466927049313.png)

```bash
注意密码是通过slappasswd生成的（例如使用的是123456作为密码）
root@ldap:/etc/ldap# slappasswd -h {md5}      
New password:
Re-enter new password:
{MD5}lk1phaN9bHwNex19prBWCA==
```

![20170808150216679075860.png](http://oe1qcatok.bkt.clouddn.com/20170808150216679075860.png)

![201709061504669315253.png](http://cdn.zqifei.com/201709061504669315253.png)

生成新的 slapd.d 目录

```bash
$ cd /etc/ldap/
$ rm -rf slapd.d
$ mkdir slapd.d
$ slaptest -f slapd.conf  -F slapd.d
$ chown -R openldap slapd.d （改变新生成的slapd.d属主）
$ service slapd restart
```