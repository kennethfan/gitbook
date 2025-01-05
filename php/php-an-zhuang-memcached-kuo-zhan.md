# PHP安装memcached扩展

## 安装GCC

```bash
sudo yum -y install gcc
```

## 安装libevent

### 下载[libevent](http://libevent.org/)

```bash
wget https://github.com/libevent/libevent/releases/download/release-2.0.22-stable/libevent-2.0.22-stable.tar.gz
```

### 解压文件

```bash
tar -zvxf libevent-2.0.22-stable.tar.gz
```

### 进入目录

```bash
cd libevent-2.0.22-stable
```

### 配置

```base
./configure --prefix=/usr/local
```

### 编译

```baseh
make
```

### 安装

```bash
make test && make install
```

## 安装memcached

### 下载[memcached](http://memcached.org/downloads)

```bash
wget http://memcached.org/files/memcached-1.4.25.tar.gz
```

### 解压文件

```bash
tar -zvxf memcached-1.4.25.tar.gz
```

### 进入目录

```bash
cd memcached-1.4.25
```

### 配置

```bash
./configure --prefix=/usr/local --with-libevent=/usr/local
```

### 编译

```bash
make
```

### 安装

```bash
make test && make install
```

## 安装php memcache扩展

### 下载[源码](http://pecl.php.net/package/memcache)

```
wget http://pecl.php.net/get/memcache-2.2.7.tgz
```

### 解压

```bash
tar -zvxf memcache-2.2.7.tgz
```

### 进入目录

```bash
cd memcache-2.2.7
```

### 配置

```bash
/data/local/php/bin/phpize
./configure --enable-memcache=/usr/local/memcached --with-php-config=/usr/local/php/bin/php-config
```

### 编译

```bash
make
```

### 安装

```bash
make test && make install
```

### 修改php配置

添加一行代码

```bash
extension=memcache.so
```

### 查看配置

```bash
/usr/local/php/bin/php -m | grep memcache
```
