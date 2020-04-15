+++
title="Kubernetes (K8s) 集群搭建"
date=2020-04-15T11:07:38+08:00
categories=["Harbor","Kubernetes"]
toc=false
+++

### 一：简单介绍
#### 1. 先决条件
**节点是充当Kubernetes集群中的辅助计算机的VM或物理计算机**。每个条件都有一个Kubelet，它是用于管理节点并与Kubernetes主节点通信的代理。该节点还应该具有用于处理容器操作的工具，例如Docker或rkt。处理生产流量的Kubernetes集群**至少应该具有三个节点**。

**硬件**：
|资源|最低要求|
|:--:|:--:|
|CPU|2|
|内存|2GB|

#### 2. 集群图
![](https://pic.downk.cc/item/5e93d2c4c2a9a83be5aca1d4.png)
+ **主机负责管理集群**：主服务器协调集群中的所有活动，例如调度应用程序，维护应用程序的所需状态，扩展应用程序以及推出新的更新。

#### 3. 安装软件
1. [Minikube](https://github.com/kubernetes/minikube/releases)
Minikube是一种可以轻松在本地运行Kubernetes的工具。MiniKube支持运行一个单节点Kubernetes集群，以供希望试用Kubernetes或每天进行开发的用户使用，不建议运用于生产环境。
2. kubeadm
kubeadm支持单个或多个节点，可以运用于生产环境。

### 二：利用kubeadm快速部署k8s
#### 1. 基础配置和先决条件
```shell script
## 关闭selinux防火墙（临时，重启后失效）
$ setenforce 0
## 修改配置文件（重启后依然有效）
$ sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
## 查看selinux状态
$ getenforce
## 关闭Linux防火墙
$ systemctl stop firewalld.service
$ systemctl disable firewalld.service
## 查看防火墙状态
$ systemctl status firewalld.service

## 禁用swap设备
### 关闭当前已启用的所有swap设备
swapoff -a && sysctl -w vm.swappiness=0
### 编辑fstab配置文件，注释掉标为swap设备的所有行
vi /etc/fstab

## 加载br_netfilter模块
$ modprobe br_netfilter
## 设置允许路由转发
$ cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
$ sysctl --system
## 查看是否生成相关文件
ls /proc/sys/net/bridge

## 资源配置文件
# /etc/security/limits.conf是Linux资源使用配置文件，用来限制用户对系统资源的使用
```
#### 2. 配置k8syum仓库并安装kubeadm，kubelet和kubectl
```shell script
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
## 由于官网未开发同步方式，可能会有索引gpg检查失败的情况，这时使用以下命令进行安装
$ yum install -y --nogpgcheck kubelet kubeadm kubectl
## 设置开机自启并启动kubelet
$ systemctl enable kubelet && systemctl start kubelet
```

#### 3.安装Docker-ce
```shell script
## 卸载以前安装过的docker
$ sudo yum remove docker \
          docker-client \
          docker-client-latest \
          docker-common \
          docker-latest \
          docker-latest-logrotate \
          docker-logrotate \
          docker-engine
$ sudo yum install -y yum-utils device-mapper-persistent-data lvm2
## 使用阿里云镜像下载
$ sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
$ sudo yum makecache fast
$ sudo yum install -y docker-ce docker-ce-cli containerd.io
## 设置开机自启并启动
$ systemctl enable docker && systemctl start docker
## 配置镜像加速器
$ vi /etc/docker/daemon.json
{
    "registry-mirrors": ["http://hub-mirror.c.163.com"]
}
## 重启容器
$ systemctl restart docker
```

#### 4. 为kube-apiserver创建负载均衡器
1. 创建一个名为kube-apiserver的负载均衡器解析DNS
    + 在云环境中，应该将控制平面节点放置在TCP后面转发负载均衡
    + 不建议在云环境中直接使用IP地址
    + 负载均衡器必须能够在apiserver端口上与所有控制平面节点通信
    + [HA代理](https://github.com/haproxy/haproxy)可以被用来做一个负载均衡器
    + 确保负载均衡器的地址始终匹配`kubeadm`的`ControlPlaneEndPoint`地址
2. 修改hosts文件（局域网），避免直接使用IP
    ```shell script
    ## 修改hostname
    # 172.16.1.148
    $ hostnamectl set-hostname k8s-master-01
    # 172.16.1.30
    $ hostnamectl set-hostname k8s-master-02
    # 172.16.1.134
    $ hostnamectl set-hostname k8s-master-03
    ## 修改hosts
    $ vim /etc/hosts
    172.16.1.148   master01.k8s.io    k8s-master-01
    172.16.1.30    master02.k8s.io    k8s-master-02
    172.16.1.134   master03.k8s.io    k8s-master-03
    ```
3. 安装HA代理
    ```shell script
    $ yum install -y haproxy
    ## haproxy配置文件
    /etc/haproxy/haproxy.cfg
    ## 配置文件说明
    # global：参数是进程级的，通常是和操作系统相关。这些参数一般只设置一次，如果配置无误，就不需要再次进行修改
    # defaults：配置默认参数，这些参数可以被用到frontend，backend，linsten组件
    # frontend：接收请求的前端虚拟节点，Frontend可以更加规则直接指定具体使用后端的backend
    # backend：后端服务集群的配置，是真实服务器，一个backend对应一个或多个实体服务器
    # Listen：是Frontend和backend的组合体

    ## 将第一个控制平面节点添加到负载均衡器并测试连接：
    $ nc -v 172.16.1.148 6443
    ## 将其余控制平面节点添加到负载均衡器目标组

    ## 由于apiserver尚未运行，因此出现连接拒绝错误。
    ## 但是，超时意味着负载平衡器无法与控制平面节点通信。
    ## 如果发生超时，请重新配置负载均衡器以与控制平面节点通信
    ```
4. 初始化kubernetes集群
    ```shell script
    ## 修改镜像仓库为阿里云镜像仓库
    $ kubeadm init --kubernetes-version=1.18.1 \
        --apiserver-advertise-address=172.16.1.148 \
        --image-repository registry.aliyuncs.com/google_containers \
        --service-cidr=10.10.0.0/16  --pod-network-cidr=10.122.0.0/16 \
        --control-plane-endpoint "172.16.1.148:6443" \ 
        --upload-certs
    ## 记住生成的最后部分内容，此内容需要在其它节点加入Kubernetes集群时执行
    $ kubeadm join 172.16.1.148:6443 --token ov91fv.cg8g9gt84hk7ry2l \
    --discovery-token-ca-cert-hash sha256:6994aa3e5fc83155095c2cce0fd29679194dd67ba790f818182b527e2a72f48d
    ```
5. 配置kubectl环境变量
    ```shell script
    $ mkdir -p $HOME/.kube
    $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    $ sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ## 使得kubectl可以自动补充
    $ source <(kubectl completion bash)

    ## 查看pod状态
    $ kubectl get pods --namespace=kube-system
    ```
    可以看到coredns没有启动，这是因为没有配置网络插件。
6. 安装calico网络
    ```shell script
    $ kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
    ## 查看pod和node
    $ kubectl get pod --all-namespaces
    $ kubectl get node
    ```
    此时集群状态正常，可能由于网络原因需要等待几秒才会显示正常。
7. 安装kubernetes-dashboard
    ```shell script
    ## 下载安装kubernetes-dashboard
    $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta4/aio/deploy/recommended.yaml
    # 生成样本用户
    ## 默认情况下已经生成admin-user用户
    ## 直接绑定即可
    $ vi dashboard-adminuser.yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
        name: admin-user
    roleRef:
        apiGroup: rbac.authorization.k8s.io
        kind: ClusterRole
        name: cluster-admin
    subjects:
    - kind: ServiceAccount
      name: admin-user
      namespace: kubernetes-dashboard
    $ kubectl apply -f dashboard-adminuser.yaml
    # 获取token
    ## 检查是否有admin-user，没有的话需要创建
    $ kubectl -n kubernetes-dashboard get secret | grep admin-user
    ## 创建admin-user
    $ vi admin-user.yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
        name: admin-user
        namespace: kubernetes-dashboard
    ## 获取token
    ## Bash
    $ kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
    ## Powershell
    $ kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | sls admin-user | ForEach-Object { $_ -Split '\s+' } | Select -First 1)
    ## 生成证书（p12）并上传到浏览器
    $ grep 'client-certificate-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.crt
    $ grep 'client-key-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.key
    $ openssl pkcs12 -export -clcerts -inkey kubecfg.key -in kubecfg.crt -out kubecfg.p12 -name "kubernetes-client"
    ## 访问dashboard Web页面
    $ https://172.16.1.148:6443/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
    ## 输入token
    ```
8. 加入集群
    ```shell script
    # 添加其他控制平面节点（master节点）
    ## 复制文件到master02
    $ ssh root@master02.k8s.io mkdir -p /etc/kubernetes/pki/etcd
    $ scp /etc/kubernetes/pki/{ca.*,sa.*,front-proxy-ca.*} root@master02.k8s.io:/etc/kubernetes/pki
    $ scp /etc/kubernetes/pki/etcd/ca.* root@master02.k8s.io:/etc/kubernetes/pki/etcd
    ## master节点加入集群
    $ kubeadm join 172.16.1.148:6443 --token ov91fv.cg8g9gt84hk7ry2l \
    --discovery-token-ca-cert-hash sha256:6994aa3e5fc83155095c2cce0fd29679194dd67ba790f818182b527e2a72f48d \
    --control-plane

    # node节点加入集群
    $ kubeadm join 172.16.1.148:6443 --token ov91fv.cg8g9gt84hk7ry2l \
    --discovery-token-ca-cert-hash sha256:6994aa3e5fc83155095c2cce0fd29679194dd67ba790f818182b527e2a72f48d
    ```
### 参考文档
+ https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
+ https://www.cnblogs.com/rdchenxi/p/10848030.html
+ https://developer.aliyun.com/mirror/kubernetes?spm=a2c6h.13651102.0.0.3e221b11VGVruI
+ https://docs.docker.com/engine/install/centos/
+ https://www.cnblogs.com/sanduzxcvbnm/p/11781082.html
+ https://www.cnblogs.com/xiuyuanpingjie/p/10897937.html
+ https://www.cnblogs.com/happydreamzjl/p/11046941.html
+ https://www.kubernetes.org.cn/7189.html
+ https://github.com/kubernetes/dashboard/tree/master/docs
+ https://www.cnblogs.com/RainingNight/p/deploying-k8s-dashboard-ui.html
