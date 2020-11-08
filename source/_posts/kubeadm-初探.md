---
title: kubeadm 初探
date: 2020-01-06 19:18:16
tags:
	- k8s
	- kubeadm
---

使用 kubeadm 安装 k8s 集群步骤整理。

## 示例服务器准备

Azure VM 上创建三台 centos 7.5 服务器分别作为 k8s master 和 k8s worker：

```
master: 207.46.156.242
worker1: 207.46.151.202
worker2: 168.63.140.91
```

## 操作步骤
### 安装 Docker

#### 安装相关依赖

```
sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
```
#### 添加 Docker 仓库

```
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

#### 安装 Docker 

```
sudo yum install docker-ce docker-ce-cli containerd.io
```

#### 启动 Docker

```
sudo systemctl start docker

sudo systemctl enable docker
```

### 安装 kubeadm

#### 安装 kubeadm , kubelet , kubectl

```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

# Set SELinux in permissive mode (effectively disabling it)
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

systemctl enable --now kubelet
```

#### 修改 Docker cgroups 启动

创建 `/etc/docker/daemon.json` 文件，加入以下内容

```
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```
重启 Docker

```
systemctl restart docker
```

查看修改后状态

```
docker info | grep Cgroup

Cgroup Driver: systemd
```

#### 配置 bridge 

```
echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
echo 1 > /proc/sys/net/bridge/bridge-nf-call-ip6tables
```

### 使用 kubeadm 部署 k8s 集群

#### master 节点

##### 执行初始化命令

```
kubeadm init
```

##### 记录 join 参数命令

```
kubeadm join 10.0.1.4:6443 --token mv7ql6.zvcdldux0io1kspb \
    --discovery-token-ca-cert-hash sha256:45746646f0f96bff66d66bd28a16312df3c6864f04fab58a7442609e3c207ddc
```

##### 执行集群配置命令

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

##### 部署网络插件

```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

#### worker 节点

分别执行 join 命令

```
kubeadm join 10.0.1.4:6443 --token mv7ql6.zvcdldux0io1kspb \
    --discovery-token-ca-cert-hash sha256:45746646f0f96bff66d66bd28a16312df3c6864f04fab58a7442609e3c207ddc
```

在 master 节点上查看 node 状态

```
kubectl get nodes

NAME       STATUS   ROLES    AGE     VERSION
master     Ready    master   8m45s   v1.17.0
worker01   Ready    <none>   2m31s   v1.17.0
worker02   Ready    <none>   2m23s   v1.17.0
```






