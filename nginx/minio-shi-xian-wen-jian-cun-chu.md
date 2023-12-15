---
description: minio实现文件存储
---

# minio实现文件存储

## &#x20;背景

当前项目中，文件存储使用的是本地磁盘；整体状况如下

* 上传时：后端直接将文件写到本地磁盘
* 下载时：通过springboot 提供的静态目录映射完成

当前的设计在分布式场景下会有问题，加入文件服务有多个实例，在多个实例场景下将文件写到一个统一的本地磁盘是比较麻烦的

## 目标

实现文件的分布式存储

## 方案

目前主流的文件存储方案大致如下

| 方案   | 详情                 | 优点          | 缺点              |
| ---- | ------------------ | ----------- | --------------- |
| 三方存储 | 1、各种oss，比如aws，tfs等 | 用户量大，专业团队维护 | 付费              |
| 自建   | 1、fastdfs，minio等   | 免费          | 维护成本，问题快速解决能力不足 |

公司出于成本考虑，决定使用自建的方式，结合开源热度和维护团队情况，最终选择使用minio作为实现

这里使用docker作为演示

## 实施步骤

* 镜像拉取

```bash
 docker pull bitnami/minio:2023
```

* 新建本地磁盘映射

```bash
mkdir -p /opt/data/minio
chown 1001:1001 /opt/data/minio # minio默认用户1001
```

* 启动

注意，登录用户名最小长度5位，密码最小长度8位

```bash
 docker run -d --name minio --restart always -p 9000:9000 -p 9001:9001 -v /opt/data/minio:/bitnami/minio/data --env="MINIO_ROOT_USER=控制台登录用户名" --env="MINIO_ROOT_PASSWORD=控制台登录密码" --privileged=true bitnami/minio:2023
```

* 防火墙开放端口

```bash
firewall-cmd --zone=public --add-port=9000/tcp --permanent
firewall-cmd --zone=public --add-port=9001/tcp --permanent
firewall-cmd --list-all
```

* 初始化

浏览器窗口打开，地址http://ip:9000，登录使用刚才启动时设置的密码

<figure><img src="../.gitbook/assets/minio-login.png" alt=""><figcaption></figcaption></figure>

* 创建bucket

<figure><img src="../.gitbook/assets/create-bucket-1.png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/create-bucket-2.png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/create-bucket-3.png" alt=""><figcaption></figcaption></figure>

* 创建access keys

<figure><img src="../.gitbook/assets/create-access-key-1.png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/create-access-key-2.png" alt=""><figcaption></figcaption></figure>

## 历史数据导入

* mc客户端安装

```bash
curl https://dl.min.io/client/mc/release/linux-amd64/mc \
  --create-dirs \
  -o $HOME/minio-binaries/mc

chmod +x $HOME/minio-binaries/mc
export PATH=$PATH:$HOME/minio-binaries/

mc --help
```

* 添加别名

```bash
mc alias set <别名>  http://<ip>:9000 <access_key> <secret_key>
```

* 历史数据导入

```
mc cp -r <本地文件存储根目录>  <别名>/<bucket>
```
