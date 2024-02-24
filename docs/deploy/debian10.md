# Debian 10.13

- 官方网址：https://www.debian.org
- 下载地址：http://cdimage.debian.org/cdimage/archive


## 1. 安装

1. 选择图形化安装

![](../../assets/_images/deploy/debian/1.png)

2. 选择语言，时区

![](../../assets/_images/deploy/debian/2.png)

![](../../assets/_images/deploy/debian/2_1.png)

3. 选择键盘布局

![](../../assets/_images/deploy/debian/3.png)

4. 配置主机名

![](../../assets/_images/deploy/debian/4.png)

5. 配置域名

![](../../assets/_images/deploy/debian/5.png)

6. 设置系统管理员root用户密码

![](../../assets/_images/deploy/debian/6.png)

7. 设置普通用户账号和密码

![](../../assets/_images/deploy/debian/7.png)

![](../../assets/_images/deploy/debian/7_1.png)

8. 磁盘分区

分区方法：使用整个磁盘

![](../../assets/_images/deploy/debian/8.png)

选择磁盘：SCSI3(0,0,0) - sda

![](../../assets/_images/deploy/debian/8_1.png)

方案：将所有文件放在一个分区中

![](../../assets/_images/deploy/debian/8_2.png)

将改动写入磁盘：是。`此时可以断开网络连接了，否则后面的安装非常慢`

![](../../assets/_images/deploy/debian/8_3.png)

![](../../assets/_images/deploy/debian/8_4.png)

9. 配置软件包管理器

扫描额外的安装介质：否

![](../../assets/_images/deploy/debian/9.png)

使用网络镜像：否

![](../../assets/_images/deploy/debian/9_1.png)

10. 正在设定 popularity-contest : 否

![](../../assets/_images/deploy/debian/10.png)

11. 软件选择

![](../../assets/_images/deploy/debian/11.png)

12. 安装 GRUB 启动引导器

![](../../assets/_images/deploy/debian/12.png)

![](../../assets/_images/deploy/debian/12_1.png)

13. 结束安装进程

![](../../assets/_images/deploy/debian/13.png)

## 2. 虚拟机

### 2.1 初始化

```bash
sudo passwd root    # 修改root密码
```

### 2.2 ssh配置

```bash
apt-get install ssh     # 安装
echo -e "ListenAddress 0.0.0.0\nPasswordAuthentication yes\nPermitRootLogin yes" >> /etc/ssh/sshd_config # 开启密码验证和root账号登录 
service ssh start       # 启动ssh服务
service ssh status      # 查看ssh服务状态
update-rc.d ssh enable  # 添加开机自启动
```

### 2.3 网络配置

1. 设置静态ip

```bash
ip addr
vi /etc/network/interfaces
```

```bash
auto enp0s3
iface enp0s3 inet static
address 192.168.2.3
netmask 255.255.255.0
gateway 192.168.2.1
```

```bash
systemctl restart networking
```

2. 设置dns

```bash
vim /etc/resolv.conf
```

### 2.4 更换apt源

1. 网络源

```bash
nano  /etc/apt/sources.list
```

```bash
# deb [trusted=yes] file:/media/cdrom bullseye contrib main
# deb http://security.debian.org/debian-security bullseye-security main contrib
# deb-src http://security.debian.org/debian-security bullseye-security main contrib
deb http://mirrors.163.com/debian/ bullseye main non-free contrib
deb http://mirrors.163.com/debian/ bullseye-updates main non-free contrib
deb http://mirrors.163.com/debian/ bullseye-backports main non-free contrib
deb-src http://mirrors.163.com/debian/ bullseye main non-free contrib
deb-src http://mirrors.163.com/debian/ bullseye-updates main non-free contrib
deb-src http://mirrors.163.com/debian/ bullseye-backports main non-free contrib
deb http://mirrors.163.com/debian-security/ bullseye/updates main non-free contrib
deb-src http://mirrors.163.com/debian-security/ bullseye/updates main non-free contrib
```

```bash
apt update
```

2. 光盘镜像源

![](../../assets/_images/deploy/debian/14.png)

```bash
cat  /proc/sys/dev/cdrom/info   # 查看光驱设备
mount /dev/sr0 /media/cdrom     # 手动挂载
```

添加源

```bash
nano /etc/apt/sources.list
deb [trusted=yes] file:/media/cdrom bullseye contrib main
apt update
```

### 2.5 安装vim

```bash
apt-get install vim
```

