# Ubuntu 22.04.2 LTS

- 官方网址：https://cn.ubuntu.com/
- 下载地址：https://releases.ubuntu.com

## 1. 安装系统

1. 选择Graphical install(图形化安装)

![](../../assets/_images/deploy/ubuntu/1.png)


## 2. 虚拟机设置

### 2.1 SSH配置

1. 修改root密码

```bash
sudo passwd root
```

2. 开启远程登录

```bash
vi /etc/ssh/sshd_config

# 配置文件
Port 22
ListenAddress 0.0.0.0
ListenAddress ::
PermitRootLogin yes # 允许远程登录
PasswordAuthentication yes  # 开启用户名和密码来验证

/etc/init.d/ssh restart # 启动服务
update-rc.d ssh enable  # 开机启动
```

### 2.2 网络配置

设置静态ip

```bash
vi /etc/network/interfaces
```

```bash
auto eth0
iface eth0 inet static
address 192.168.2.129
netmask 255.255.255.0
gateway 192.168.2.1
```

设置dns

```bash
vim /etc/resolv.conf
```

```bash
/etc/init.d/networking restart  # bug：需要重启
ifconfig eth0 up    # 启用网卡
dhclient eth0       # 分配IP
```

### 2.3 更换apt源

```bash
vim /etc/apt/sources.list
```

```bash
deb http://mirrors.aliyun.com/kali kali-rolling main non-free contrib
deb-src http://mirrors.aliyun.com/kali kali-rolling main non-free contrib
```

```bash
apt update
```