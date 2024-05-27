---
title: 搭建k8s集群（二）
categories:
  - 云计算
tags:
  - Kubernetes
abbrlink: b7d5da89
date: 2023-12-20 21:30:55
---

# 搭建Kubernetes集群（二）

上次讲述了Kubernetes的基础概念，今天我们开始搭建Kubernetes集群。

## 一、前置环境

### 1、搭建k8s环境平台规划

> 本次搭建k8s集群为单master集群，配置两个node节点

#### 服务器硬件配置要求

> **环境测试配置要求**

| 节点   | CPU内核数 | 内存大小 | 硬盘大小 |
| ------ | --------- | -------- | -------- |
| master | 2核以上   | 4G以上   | 30G以上  |
| node   | 2核以上   | 4G以上   | 20G以上  |

### 2、安装docker

此次的搭建Kubernetes集群依托于docker进行，docker的基本概念这里不做过多解释，详细内容可以查看[Docker中文文档](https://www.docker.org.cn/index.html)

#### 1.centos下安装docker

> 其他系统参照如下文档

[https://docs.docker.com/engine/install/centos/](https://docs.docker.com/engine/install/centos/)

#### 2.移除以前docker相关包

```bash
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

#### 3.配置yum源

```bash
sudo yum install -y yum-utils
sudo yum-config-manager \
--add-repo \
http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

#### 4.安装docker

```bash
sudo yum install -y docker-ce docker-ce-cli containerd.io


#以下是在安装k8s的时候使用
yum install -y docker-ce-20.10.7 docker-ce-cli-20.10.7  containerd.io-1.4.6
```

#### 5.配置开机自启动

```bash
systemctl enable docker --now
```

#### 6.配置加速

> 这里额外添加了docker的生产环境核心配置cgroup

```bash

sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://82m9ar63.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 二、搭建kubernetes集群

### 1、安装kubeadm

- 一台兼容的 Linux 主机。Kubernetes 项目为基于 Debian 和 Red Hat 的 Linux 发行版以及一些不提供包管理器的发行版提供通用的指令
- 每台机器 2 GB 或更多的 RAM （如果少于这个数字将会影响你应用的运行内存)
- 2 CPU 核或更多
- 集群中的所有机器的网络彼此均能相互连接(公网和内网都可以)

- - **设置防火墙放行规则**

- 节点之中不可以有重复的主机名、MAC 地址或 product_uuid。请参见[这里](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#verify-mac-address)了解更多详细信息。

- - **设置不同hostname**

- 开启机器上的某些端口。请参见[这里](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports) 了解更多详细信息。

- - **内网互信**

- 禁用交换分区。为了保证 kubelet 正常工作，你 **必须** 禁用交换分区。
  - **永久关闭**

#### 1.基础环境

> 所有机器执行以下操作

```bash
#各个机器设置自己的域名
hostnamectl set-hostname xxxx


# 将 SELinux 设置为 permissive 模式（相当于将其禁用）
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

#关闭swap
swapoff -a  
sed -ri 's/.*swap.*/#&/' /etc/fstab

#允许 iptables 检查桥接流量
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system

```

#### 2.安装kubelet、kubeadm、kubectl

```bash

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
   http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF


sudo yum install -y kubelet-1.20.9 kubeadm-1.20.9 kubectl-1.20.9 --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```

> kubelet 现在每隔几秒就会重启，因为它陷入了一个等待 kubeadm 指令的死循环

### 2、使用kubeadm引导集群

#### 1.下载各个及其需要的镜像

````bash
sudo tee ./images.sh <<-'EOF'
#!/bin/bash
images=(
kube-apiserver:v1.20.9
kube-proxy:v1.20.9
kube-controller-manager:v1.20.9
kube-scheduler:v1.20.9
coredns:1.7.0
etcd:3.4.13-0
pause:3.2
)
for imageName in ${images[@]} ; do
docker pull registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/$imageName
done
EOF
   
chmod +x ./images.sh && ./images.sh
````

#### 2.初始化主节点

```bash
#所有机器添加master域名映射，以下需要修改为自己的
echo "192.168.43.130  cluster-endpoint" >> /etc/hosts

#主节点初始化
kubeadm init \
--apiserver-advertise-address=192.168.43.130 \
--control-plane-endpoint=cluster-endpoint \
--image-repository registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images \
--kubernetes-version v1.20.9 \
--service-cidr=10.96.0.0/16 \
--pod-network-cidr=192.168.0.0/16

#所有网络范围不重叠
```

##### 初始化控制面板成功

```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join cluster-endpoint:6443 --token opisoe.jmt4vl7nl8bmq15t \
    --discovery-token-ca-cert-hash sha256:e9489dac463642525ff6db90fda3fc8b2619a49e2feb2a0a2c16998bf8a90f40 \
    --control-plane 

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join cluster-endpoint:6443 --token opisoe.jmt4vl7nl8bmq15t \
    --discovery-token-ca-cert-hash sha256:e9489dac463642525ff6db90fda3fc8b2619a49e2feb2a0a2c16998bf8a90f40 
[root@k8s-master ~]# 
```

```bash
#查看集群所有节点
kubectl get nodes

#根据配置文件，给集群创建资源
kubectl apply -f xxxx.yaml

#查看集群部署了哪些应用？
docker ps   ===   kubectl get pods -A
# 运行中的应用在docker里面叫容器，在k8s里面叫Pod
kubectl get pods -A
```

#### 3.安装网络组件

[calico官网](https://docs.tigera.io/)

```bash
curl https://docs.projectcalico.org/v3.20/manifests/calico.yaml -O

kubectl apply -f calico.yaml
```

#### 4.加入node节点

```bash
kubeadm join cluster-endpoint:6443 --token x5g4uy.wpjjdbgra92s25pp \
	--discovery-token-ca-cert-hash sha256:6255797916eaee52bf9dda9429db616fcd828436708345a308f4b917d3457a22
```

> 新令牌
>
> kubeadm token create --print-join-command

> ***高可用部署方式，也是在这一步的时候，使用添加主节点的命令即可***

#### 5.验证集群

- 验证集群节点状态
  - kubectl get nodes

## 结束

这次文章，我们将kubernetes的主节点的初始化和网络设置已经完成，下次我们将配置kubernetes的node节点以及安装一个非常方便的可视化控制面板dashboard。以上就是我对dashboard集群搭建的一些见解，感兴趣的小伙伴可以在评论区里发表你的观点。
