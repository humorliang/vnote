# 本教程参考使用 kubeasz 进行安装

> 项目地址： https://github.com/easzlab/kubeasz

## 环境准备

```
四台centos虚拟机,并在 master01 上安装好ansible 以及配置好免密登录其他节点

192.168.139.21 master01
192.168.139.22 master02
192.168.139.23 master03
192.168.139.24 node01
```

## 开始安装
**1. 下载工具脚本 ezdown ，以及下载相关镜像**
```sh

# 更新一下所有节点
ansible nodes -a "yum install update -y"

# 下载工具脚本ezdown，使用kubeasz版本3.0.1，推荐根据对应的集群选择版本
export release=3.0.1
# 如果没有 wget 命令 安装 wget 命令  yum install wget 
wget https://github.com/easzlab/kubeasz/releases/download/${release}/ezdown
# 修改执行权限
chmod +x ./ezdown
# 使用工具脚本下载 所有的依赖和安装包  
# 所有文件（kubeasz代码、二进制、离线镜像）均已整理好放入目录 /etc/kubeasz
./ezdown -D

# 查看下载内容
ll /etc/kubeasz

#查看目录  /etc/kubeasz
tree -L 1

# ├── ansible.cfg
# ├── bin
# ├── clusters # 集群示例目录
# ├── docs
# ├── down  # 相关镜像和安装包
# ├── example
# ├── ezctl  # 脚本命令
# ├── ezdown
# ├── manifests
# ├── pics
# ├── playbooks
# ├── README.md
# ├── roles  
# └── tools

```

**2. 使用ezctl 创建集群实例配置信息**

```sh
# 切换到工具目录
cd /etc/kubeasz

# 创建实例
./ezctl new k8s-01

# 修改相关配置信息，主要是 节点IP等，根据自己机器的环境去设置相关信息
vim /etc/kubeasz/clusters/k8s-01/hosts
vim /etc/kubeasz/clusters/k8s-01/config.yml
```

**3. 进行每一步安装或者直接全部安装**
这里使用步骤安装，详情请看 kubeasz 文档   
```sh
# 一键安装
./ezctl setup k8s-01 all

# 或者分步安装，具体使用 ezctl help setup 查看分步安装帮助信息

# available steps:
#     01  prepare            to prepare CA/certs & kubeconfig & other system settings 
#     02  etcd               to setup the etcd cluster
#     03  container-runtime  to setup the container runtime(docker or containerd)
#     04  kube-master        to setup the master nodes
#     05  kube-node          to setup the worker nodes
#     06  network            to setup the network plugin
#     07  cluster-addon      to setup other useful plugins
#     90  all                to run 01~07 all at once
#     10  ex-lb              to install external loadbalance for accessing k8s from outside
#     11  harbor             to install a new harbor server or to integrate with an existed one

#  分布安装 可以进行校验
# ./ezctl setup k8s-01 01
# ./ezctl setup k8s-01 02
# ./ezctl setup k8s-01 03
# ./ezctl setup k8s-01 04
```
**3.1 步骤解析**

[01-创建证书和安装准备](https://github.com/easzlab/kubeasz/blob/master/docs/setup/01-CA_and_prerequisite.md)

**创建证书**
根据认证对象可以将证书分成三类：

    服务器证书`server cert`
    客户端证书`client cert`
    对等证书`peer cert`(既是`server cert`又是`client cert`)

在`kubernetes` 集群中需要的证书种类如下：

- `etcd `节点需要标识自己服务的`server cert`，也需要`client cert`与`etcd`集群其他节点交互，当然可以分别指定2个证书，为方便这里使用一个对等证书.

- `master` 节点需要标识 `apiserver`服务的`server cert`，也需要`client cert`连接`etcd`集群，这里也使用一个对等证书.

- `kubectl` `calico` `kube-proxy` 只需要`client cert`，因此证书请求中 `hosts` 字段可以为空.

- `kubelet` 需要标识自己服务的`server cert`，也需要`client cert`请求`apiserver`，也使用一个对等证书.

**生成 kubeconfig 配置文件**

kubectl使用~/.kube/config 配置文件与kube-apiserver进行交互，且拥有管理 K8S集群的完全权限，
```sh
# RBAC 预定义的 ClusterRoleBinding 将 Group system:masters 与 ClusterRole cluster-admin 绑定，这就赋予了kubectl所有集群权限
kubectl describe clusterrolebinding cluster-admin

# 使用kubectl config 生成kubeconfig 自动保存到 ~/.kube/config，生成后 cat ~/.kube/config可以验证配置文件包含 kube-apiserver 地址、证书、用户名等信息

kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=127.0.0.1:8443
kubectl config set-credentials admin --client-certificate=admin.pem --embed-certs=true --client-key=admin-key.pem
kubectl config set-context kubernetes --cluster=kubernetes --user=admin
kubectl config use-context kubernetes
```

**生成 kube-proxy.kubeconfig 配置文件**
```sh
# CN 指定该证书的 User 为 system:kube-proxy 预定义的 ClusterRoleBinding system:node-proxier 
# 将User system:kube-proxy 与 Role system:node-proxier 绑定，
# 授予了调用 kube-apiserver Proxy 相关 API 的权限
kubectl describe clusterrolebinding system:node-proxier

# 使用kubectl config 生成kubeconfig 自动保存到 kube-proxy.kubeconfig

kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=127.0.0.1:8443 --kubeconfig=kube-proxy.kubeconfig
kubectl config set-credentials kube-proxy --client-certificate=kube-proxy.pem --embed-certs=true --client-key=kube-proxy-key.pem --kubeconfig=kube-proxy.kubeconfig
kubectl config set-context default --cluster=kubernetes --user=kube-proxy --kubeconfig=kube-proxy.kubeconfig
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```


[02-安装etcd集群](https://github.com/easzlab/kubeasz/blob/master/docs/setup/02-install_etcd.md)

1. [部署ansible文件](https://github.com/easzlab/kubeasz/blob/master/roles/etcd/tasks/main.yml)
2. 使用命令安装
```sh
./ezctl setup k8s-01 01
```
3. 检查etcd
```sh
# 验证etcd集群状态 查看服务状态
systemctl status etcd

# 查看运行日志
journalctl -u etcd

# 在任一 etcd 集群节点上执行如下命令
export NODE_IPS="192.168.139.21 192.168.139.22 192.168.139.23"
for ip in ${NODE_IPS}; do
  ETCDCTL_API=3 etcdctl \
  --endpoints=https://${ip}:2379  \
  --cacert=/etc/kubernetes/ssl/ca.pem \
  --cert=/etc/etcd/ssl/etcd.pem \
  --key=/etc/etcd/ssl/etcd-key.pem \
  endpoint health; done
```

[03-安装容器运行时](https://github.com/easzlab/kubeasz/blob/master/docs/setup/03-container_runtime.md)

目前k8s官方推荐使用 [containerd](https://github.com/easzlab/kubeasz/blob/master/docs/setup/containerd.md)

**kubeasz 集成安装 containerd**
```
安装前修改配置文件，在clusters/k8s-01/hosts 中修改全局变量 CONTAINER_RUNTIME="containerd"
其他正常执行安装即可：分步安装 ezctl setup k8s-01 03
```

**默认使用docker**
```sh
#  查看一下是否是docker
cat /etc/kubeasz/clusters/k8s-01/hosts| grep CONTAINER_RUNTIME
```

**清理 iptables**
因为`calico`网络、`kube-proxy`等将大量使用 `iptables`规则，安装前清空所有`iptables`策略规则；常见发行版`Ubuntu`的 `ufw` 和` CentOS`的` firewalld`等基于iptables的防火墙最好直接卸载，避免不必要的冲突。

`WARNNING`: 如果有自定义的`iptables`规则也会被一并清除，如果一定要使用自定义规则，可以集群安装完成后在应用规则

```sh
iptables -F && iptables -X \
        && iptables -F -t nat && iptables -X -t nat \
        && iptables -F -t raw && iptables -X -t raw \
        && iptables -F -t mangle && iptables -X -t mangle
```
`calico` 网络支持 `network-policy`，使用的`calico-kube-controllers` 会使用到`iptables` 所有的四个表 `filter` `nat` `raw` `mangle`，所以一并清理

**一键安装设置**
```sh
./ezctl setup k8s-01 03
```
**验证**
安装成功后验证如下：
```sh
systemctl status docker 	# 服务状态
journalctl -u docker 		# 运行日志
docker version
docker info
```
```sh
# 查看 iptables filter表 FORWARD链，最后要有一个 -A FORWARD -j ACCEPT 保底允许规则
iptables-save | grep FORWARD 
```
[04-安装master节点](https://github.com/easzlab/kubeasz/blob/master/docs/setup/04-install_kube_master.md)
部署`master`节点主要包含三个组件`apiserver` `scheduler` `controller-manager`，其中：

`apiserver`提供集群管理的REST API接口，包括认证授权、数据校验以及集群状态变更等
- 只有API Server才直接操作etcd
- 其他模块通过API Server查询或修改数据
- 提供其他模块之间的数据交互和通信的枢纽

`scheduler`负责分配调度Pod到集群内的`node`节点
- 监听`kube-apiserver`，查询还未分配`Node`的`Pod`
- 根据调度策略为这些`Pod`分配节点

`controller-manager`由一系列的控制器组成，它通过`apiserver`监控整个集群的状态，并确保集群处于预期的工作状态
**一键安装**
```sh
./ezctl setup k8s-01 04
```
**验证**
```sh
# 查看进程状态
systemctl status kube-apiserver
systemctl status kube-controller-manager
systemctl status kube-scheduler
# 查看进程运行日志
journalctl -u kube-apiserver
journalctl -u kube-controller-manager
journalctl -u kube-scheduler
```

执行` kubectl get componentstatus(cs)` 可以看到
```sh
# 这里不影响使用详情请见issues  https://github.com/easzlab/kubeasz/issues/1000

# 解决： https://github.com/easzlab/kubeasz/commit/421dd0b3b35ef2ec5ce80b0f564051d3dc708689
# 直接修改 roles/kube-master/templates/kube-scheduler-config.yaml.j2  地址为 0.0.0.0 
NAME                 STATUS    MESSAGE              ERROR
scheduler            Unhealthy  Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection refused
controller-manager   Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
etcd-2               Healthy   {"health": "true"}   
etcd-1               Healthy   {"health": "true"} 
```
[05-安装node节点](https://github.com/easzlab/kubeasz/blob/master/docs/setup/05-install_kube_node.md)
```
kube_node 是集群中运行工作负载的节点，前置条件需要先部署好kube_master节点，它需要部署如下组件：

kubelet： kube_node上最主要的组件
kube-proxy： 发布应用服务与负载均衡
haproxy：用于请求转发到多个 apiserver，详见HA-2x 架构
calico： 配置容器网络 (或者其他网络组件)
```
**一键安装**
```sh
./ezctl setup k8s-01 05

```
**验证**
```sh
# 验证 node 状态kube
systemctl status kubelet	# 查看状态
systemctl status kube-proxy
journalctl -u kubelet		# 查看日志
journalctl -u kube-proxy 
```
[06-安装集群网络](https://github.com/easzlab/kubeasz/blob/master/docs/setup/06-install_network_plugin.md)
首先回顾下K8S网络设计原则，在配置集群网络插件或者实践K8S 应用/服务部署请时刻想到这些原则：

- 1.每个`Pod`都拥有一个独立IP地址，`Pod`内所有容器共享一个网络命名空间
- 2.集群内所有`Pod`都在一个直接连通的扁平网络中，可通过`IP`直接访问
  - 所有容器之间无需`NAT`就可以直接互相访问
  - 所有`Node`和所有容器之间无需NAT就可以直接互相访问
  - 容器自己看到的IP跟其他容器看到的一样
- 3.`Service cluster IP`尽可在集群内部访问，外部请求需要通过`NodePort`、`LoadBalance`或者`Ingress`来访问

`Container Network Interface (CNI)`是目前CNCF主推的网络模型，它由两部分组成：

- `CNI Plugin`负责给容器配置网络，它包括两个基本的接口
  - 配置网络: `AddNetwork(net *NetworkConfig, rt *RuntimeConf) (types.Result, error)`
  - 清理网络: `DelNetwork(net *NetworkConfig, rt *RuntimeConf) error`
- `IPAM Plugin`负责给容器分配IP地址

`Kubernetes Pod`的网络是这样创建的：

- 0.每个Pod除了创建时指定的容器外，都有一个kubelet启动时指定的`基础容器`，比如：`easzlab/pause` `registry.access.redhat.com/rhel7/pod-infrastructure`
- 1.首先 kubelet创建`基础容器`生成network namespace
- 2.然后 kubelet调用网络CNI driver，由它根据配置调用具体的CNI 插件
- 3.然后 CNI 插件给`基础容器`配置网络
- 4.最后 Pod 中其他的容器共享使用`基础容器`的网络

**修改插件类型**
```sh
#  参考文档 https://github.com/easzlab/kubeasz/blob/master/docs/setup/network-plugin/calico.md
vim /etc/kubeasz/clusters/k8s-01/hosts

# CLUSTER_NETWORK="calico"

```

**一键安装**
```
./ezctl setup k8s-01 06
```
[07-安装集群插件](https://github.com/easzlab/kubeasz/blob/master/docs/setup/07-install_cluster_addon.md)
常用、必要的插件自动集成到安装脚本之中:

- [自动脚本](https://github.com/easzlab/kubeasz/blob/master/roles/cluster-addon/tasks/main.yml)

配置开关
- 参照[配置指南](https://github.com/easzlab/kubeasz/blob/master/docs/setup/config_guide.md)，生成后在 `roles/cluster-addon/defaults/main.yml` 配置

脚本介绍
- 1. 根据`hosts`文件中配置的`CLUSTER_DNS_SVC_IP` `CLUSTER_DNS_DOMAIN`等参数生成`kubedns.yaml`和`coredns.yaml`文件
- 2. 注册变量`pod_info`，`pod_info`用来判断现有集群是否已经运行各种插件
- 3. 根据`pod_info`和配置开关逐个进行/跳过插件安装

查看相关组件
```sh
#
kubectl get po -A

NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-5677ffd49-k5gtv   1/1     Running   0          7m56s
kube-system   calico-node-2x2lf                         1/1     Running   0          7m56s
kube-system   calico-node-575gh                         1/1     Running   0          7m56s
kube-system   calico-node-dc5gt                         1/1     Running   0          7m56s
kube-system   calico-node-v2ccc                         1/1     Running   0          7m56s
kube-system   coredns-5787695b7f-t88cz                  1/1     Running   0          56s
kube-system   metrics-server-8568cf894b-2xfvp           1/1     Running   0          49s
kube-system   node-local-dns-9v47m                      1/1     Running   0          54s
kube-system   node-local-dns-bbzcm                      1/1     Running   0          54s
kube-system   node-local-dns-l8sz2                      1/1     Running   0          54s
kube-system   node-local-dns-ls2qf                      1/1     Running   0          54s

```