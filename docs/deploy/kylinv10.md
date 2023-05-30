# Kylin-Server-V10-SP3

- 官方网址：https://www.kylinos.cn

## 1. 安装系统

1. 选择图形化安装

![](../../assets/_images/deploy/kylin/1.png)

2. 选择语言

![](../../assets/_images/deploy/kylin/2.png)

3. 设置系统分区

![](../../assets/_images/deploy/kylin/3.png)

![](../../assets/_images/deploy/kylin/3_1.png)

4. 设置语言支持

![](../../assets/_images/deploy/kylin/4.png)

5. 设置软件选择

![](../../assets/_images/deploy/kylin/5.png)

![](../../assets/_images/deploy/kylin/5_1.png)

6. 设置网络和主机名

![](../../assets/_images/deploy/kylin/6.png)

![](../../assets/_images/deploy/kylin/6_1.png)

![](../../assets/_images/deploy/kylin/6_2.png)

7. 设置时间和日期

![](../../assets/_images/deploy/kylin/7.png)

![](../../assets/_images/deploy/kylin/7_1.png)

8. 设置root用户和密码

![](../../assets/_images/deploy/kylin/8.png)

![](../../assets/_images/deploy/kylin/8_1.png)

9. 点击开始安装

![](../../assets/_images/deploy/kylin/9.png)

10. 查看实时安装进度

![](../../assets/_images/deploy/kylin/10.png)

11. 安装完成，重启系统

![](../../assets/_images/deploy/kylin/11.png)

12. 同意许可协议，安装结束

![](../../assets/_images/deploy/kylin/12.png)

## 2. 虚拟机设置

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

![](../../assets/_images/deploy/kylin/14.png)

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

