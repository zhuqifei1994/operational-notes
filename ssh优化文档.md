## ssh 优化文档

ssh服务的配置文件为 **/etc/ssh/sshd_config**。

### 禁用密码登录

```bash
# 启用密钥验证
RSAAuthentication yes
PubkeyAuthentication yes

# 指定公钥数据库文件
AuthorizedKeysFile  .ssh/authorized_keys

# 禁用密码验证
PasswordAuthentication no
```

### 修改端口号
```bash
# 将默认的 22 端口改为 其他端口号
Port 51111 
```

### 禁止空密码登录
```bash
PermitEmptyPasswords yes
```

### 优化ssh连接过慢，DNS导致的问题
```bash
UseDNS no
```

### GSSAPIAuthentication，使ssh连接的更快
```bash
GSSAPIAuthentication no
```