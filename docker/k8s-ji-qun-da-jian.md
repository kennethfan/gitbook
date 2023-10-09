# k8s集群搭建

## 机器准备

3台机器，分别是FL01，FL02，FL03

每台机器上都需要设置/etc/hosts

hostname也需要设置

```bash
sudo hostnamectl set-hostname FL01 # 不同机器请替换hostname
```

## 安装前准备

### 防火墙关闭

<pre class="language-bash"><code class="lang-bash">sudo systemctl stop firewalld 
sudo systemctl disable firewalld
<strong>sudo setenforce 0
</strong><strong>sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
</strong></code></pre>

### 内存交换区关闭

```bash
sudo swapoff -a
sudo sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab
```

### iptables设置

```bash
sudo iptables -F 
sudo iptables -X 
sudo iptables -F -t nat 
sudo iptables -X -t nat 
sudo iptables -P FORWARD ACCEPT
```

### 系统参数设置

```
vim /etc/sysctl.d/k8s.conf
```

内容如下

```
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```

```bash
sysctl --system
```

## 安装前准备

```bash
sudo yum -y update 
sudo yum install -y conntrack ipvsadm ipset jq sysstat curl iptables libseccomp
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```

## 软件安装

### docker

#### 镜像源配置

```bash
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

#### 安装

```bash
yum install -y docker-ce-18.09.0 docker-ce-cli-18.09.0 containerd.io
```

#### 参数设置

```bash
sudo vim /etc/docker/daemon.json
```

内容如下

```json
{
	"registry-mirrors": ["https://orptaaqe.mirror.aliyuncs.com"],
	"exec-opts": ["native.cgroupdriver=systemd"]
}
```

#### 启动

```bash
sudo systemctl start docker.service && sudo systemctl enable docker
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
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
```

#### 安装

```bash
sudo yum -y install kubectl-1.23.5-0 kubelet-1.23.5-0 kubeadm-1.23.5-0
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

### 主节点

#### 初始化

ip请自行替换

```bash
sudo kubeadm init --kubernetes-version=1.23.5 --apiserver-advertise-address=192.168.0.57 --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers
```

执行完之后，会有一个文字，显示加入的命令

```bash
kubeadm join 192.168.0.57:6443 --token zq0ulp.m5uhmwb9ku84lpv2  --discovery-token-ca-cert-hash sha256:1fc709d35dc3019f16779d25d2a9920feb0729e9b7f2796ce1e2cd9be98f6660
```

#### 配置copy

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### 网络配置

```bash
kubectl apply -f https://docs.projectcalico.org/v3.9/manifests/calico.yaml
```

### 从节点

执行刚才主节点输出的命令加入网络

## 参考

[https://baijiahao.baidu.com/s?id=1749026775713590928\&wfr=spider\&for=pc](https://baijiahao.baidu.com/s?id=1749026775713590928\&wfr=spider\&for=pc)
