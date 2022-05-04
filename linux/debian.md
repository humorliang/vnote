# debian 操作系统常见操作

**1 卸载debian 预装应用和游戏**
```sh
# 卸载游戏
sudo apt purge iagno lightsoff four-in-a-row gnome-robots pegsolitaire gnome-2048 hitori gnome-klotski gnome-mines 
gnome-mahjongg gnome-sudoku quadrapassel swell-foop gnome-tetravex gnome-taquin aisleriot && sudo apt autoremove

#  卸载liboffice
sudo apt-get remove --purge libreoffice*  && sudo apt-get clean && sudo apt-get autoremove
```

**2 deb 软件包安装**
```sh
sudo dpkg -i 软件包名.deb
```

**3 安装常见的软件包**
```sh
sudo apt install vim wget curl htop git proxychains4 screenfetch tmux bash-completion  zsh fonts-powerline fzf net-tools openssh-server firewalld bat
```

**4 镜像源**
```sh
vim /etc/apt/sources.list

#  debian 镜像源
deb http://mirrors.ustc.edu.cn/debian/ bullseye main contrib non-free
deb-src http://mirrors.ustc.edu.cn/debian/ bullseye main contrib non-free

deb http://mirrors.ustc.edu.cn/debian/ bullseye-updates main contrib non-free
deb-src http://mirrors.ustc.edu.cn/debian/ bullseye-updates main contrib non-free

deb http://mirrors.ustc.edu.cn/debian/ bullseye-backports main contrib non-free
deb-src http://mirrors.ustc.edu.cn/debian/ bullseye-backports main contrib non-free

deb http://mirrors.ustc.edu.cn/debian-security/ bullseye-security main contrib non-free
deb-src http://mirrors.ustc.edu.cn/debian-security/ bullseye-security main contrib non-free

```
**错误解锁**
```sh
# 用sudo apt-get update时出现“ E: 无法获得锁 /var/lib/apt/lists/lock”错误。

# 方法一：
# 执行一下 
      
sudo dpkg --configure -a

# 方法二：

sudo rm /var/lib/apt/lists/lock

# 方法三：

# 1、ps-aux 查出apt-get进程的PID,
# 2、用sudo kill PID代码 杀死进程(我都是找出带apt字样的进程格杀勿论)
```