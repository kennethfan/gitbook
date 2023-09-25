# k8s集群搭建

## 软件安装

### docker

#### 安装

```bash
sudo curl -fsSL https://get.docker.com/ | sudo sh 
```

#### 启动

```bash
sudo systemctl start docker.service && sudo systemctl enable docker
```

### containerd

#### 安装

```bash
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y containerd.io

```

#### 初始化&启动

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml # 生成默认配置
sudo systemctl start containerd && sudo systemctl enable containerd
```

### kube\*

#### 镜像源修改

```bash
sudo vim /etc/yum.repos.d/kubernetes.repo
```

内容如下

```
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
```

#### 安装

```bash
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

#### 自动补全

```bash
sudo yum -y install bash-completion
source /usr/share/bash-completion/bash_completion
sudo kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null
sudo chmod a+r /etc/bash_completion.d/kubectl
```

#### 启动

```bash
sudo systemctl start kubelet && sudo systemctl enable kubelet
```
