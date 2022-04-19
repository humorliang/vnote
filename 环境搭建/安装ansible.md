# 安装ansible
## 基础环境
```
4 台centos7系统的虚拟机

192.168.139.21
192.168.139.22
192.168.139.23
192.168.139.24
```

## 准备安装

> vm 安装完毕Centos 系统，设置系统的IP 地址，以及DNS解析后，在其中 192.168.139.21 这台机器上安装 ansible 工作去操作其他机器。

**1. 环境更新**
```sh
# 更新epel 仓库
sudo yum -y install epel-release
# 更新系统
sudo yum -y update
```

**2. 安装ansible**
```sh
sudo yum -y install ansible

# 查看版本
ansible --version 
```

**3. 每台机器都安装配置好ssh 服务**
```sh
# 安装 ssh server
sudo yum install openssh-server -y
# 查看ssh 状态
systemctl status sshd
# 启动ssh
systemctl start sshd
# 查看IP
ip addr
```

**4. 配置ansible 的host**
```sh
sudo vim /etc/ansible/hosts

#  加入的内容 将机器进行分组
# [all]
# 192.168.139.21
# 192.168.139.22
# 192.168.139.23
# 192.168.139.24

# [k8s-master]
# 192.168.139.21
# 192.168.139.22
# 192.168.139.23

# [k8s-node]
# 192.168.139.24
```

**5. 生成ssh 密钥**
```sh
# 生成ssh key
ssh-keygen

# 将机器的key 复制到其他节点的ssh auth keys 列表里，方便后期ansible 免密操作
ssh-copy-id root@192.168.139.21
ssh-copy-id root@192.168.139.22
ssh-copy-id root@192.168.139.23
ssh-copy-id root@192.168.139.24

# 都安装完毕后我们可以 免密去登录每台节点了
# ssh root@192.168.139.22
```

**6. 测试一下ansible 访问各节点的连通性**
```sh
# 访问所有节点 均成功
ansible -m ping all
```

**7. ansible 学习系列教程**   
[学习教程](http://getansible.com/reference/resources)