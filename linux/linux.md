## 主机可以ping 通，但是加上端口无法 curl 通
**现象:**
```
curl: (7) Failed to connect to 172.116.3.1 port 80: 没有到主机的路由
```
**解决:**
```sh
# 登录目标服务器 查看服务端口是否开启
netstat -ntpl|grep [$Port]

# 防火墙规则
iptables -nL

# 执行以下命令，查看防火墙状态。
systemctl status firewalld.service

# 执行以下命令，关闭防火墙。
systemctl stop firewalld.service

# 执行以下命令，设置开机不自启防火墙服务。
systemctl disable firewalld.service
```
