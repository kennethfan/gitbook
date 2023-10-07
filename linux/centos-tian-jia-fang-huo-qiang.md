# centos添加防火墙

## 背景

项目是和其他公司合作的，代码部署分布在两个平台，其中一部分部署在公司自建网络，一部分服务部署在阿里云(合作伙伴服务器都在阿里云)。

出于安全考虑，阿里云对外端口需要配置防火墙，但是有个问题：正常情况直接在阿里云安全组配置即可，但是因为公司自建网络，服务器出口ip不固定，如果使用阿里云安全组配置，需要频繁变更配置，每次都需要合作伙伴扫码登录阿里云，比较麻烦且时效性不可控

因此采用如下方案：在阿里云配置一个基本的网段(172.16.0.0/16，这里是个例子)，然后具体的服务器上使用liunx自带防火墙，配置具体ip白名单，这样就需要频繁登录阿里云

## 配置方案

其中$ip表示具体的ip，$port表示具体的端口

### 不限制ip，比如80,443端口

```bash
firewall-cmd --permanent --add-rich-rule="rule family="ipv4" port protocol="tcp" port="$port" accept"
```

### 限制ip

```bash
firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="$ip" port protocol="tcp" port="$port" accept"
```

### 重启防火墙

```bash
systemctl restart firewalld.service
```

### 查看防火墙配置

```bash
firewall-cmd --list-all
```
