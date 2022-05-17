## centos 修改界面登录方式
CentOS 7 中直接使用 systemd 指令修改启动目的状态即可。

使用，
```sh
# 查询
systemctl get-default

# 设置终端界面
systemctl set-default multi-user.target
```
可以查询到当前所设定的状态。`multi-user.target` 相当于以前的 level 3，也就是命令行终端；而 `graphical.target` 相当于以前的 level 5，也就是图形界面。

所以如果要设置默认启动到图形界面，则执行，
```sh
systemctl set-default graphical.target
```