# Ubuntu 22.04 LTS

- 官方网站：https://cn.ubuntu.com
- 下载地址：https://releases.ubuntu.com

## 1. 系统安装

1. 等待初始化

![](../../assets/_images/deploy/ubuntu/1.png)

2. 选择语言

![](../../assets/_images/deploy/ubuntu/2.png)

3. 选择键盘布局

![](../../assets/_images/deploy/ubuntu/3.png)

4. 选择安装类型

![](../../assets/_images/deploy/ubuntu/4.png)

5. 网络配置

![](../../assets/_images/deploy/ubuntu/5.png)

6. 代理配置

![](../../assets/_images/deploy/ubuntu/6.png)

7. 镜像源配置

![](../../assets/_images/deploy/ubuntu/7.png)

8. 系统是否更新，选否

![](../../assets/_images/deploy/ubuntu/8.png)

9. 引导存储配置

![](../../assets/_images/deploy/ubuntu/9.png)

10. 存储配置

![](../../assets/_images/deploy/ubuntu/10.png)

11. 确定继续

![](../../assets/_images/deploy/ubuntu/11.png)

12. 配置账号信息

![](../../assets/_images/deploy/ubuntu/12.png)

13. 安装ssh

![](../../assets/_images/deploy/ubuntu/13.png)

14. 安装其他软件

![](../../assets/_images/deploy/ubuntu/14.png)

15. 安装更新

![](../../assets/_images/deploy/ubuntu/15.png)

16. 下载完毕，直接重启

具体根据自己网络状况，可以选择 Cancel update and reboot 取消更新并重启
![](../../assets/_images/deploy/ubuntu/16.png)

![](../../assets/_images/deploy/ubuntu/16_1.png)

17. 输入ENTER确定重启

![](../../assets/_images/deploy/ubuntu/17.png)

18. 共享文件夹

![](../../assets/_images/deploy/ubuntu/18.png)

手动挂载
```bash
mkdir /mnt/ubuntushare
sudo mount -t vboxsf share /mnt/ubuntushare
```

自动挂载
```bash
echo 'share /mnt/ubuntushare vboxsf rw,auto 0 0' >> /etc/fstab  # 自动挂载
cat /etc/fstab          # 查看写入分区信息
```

19. 挂载新的Linux磁盘

![](../../assets/_images/deploy/ubuntu/19.png)

```bash
fdisk -l
sudo mkfs.ext4 /dev/sdb                 # 格式化
sudo mkdir /mnt/sdb                     # 创建sdb文件夹
sudo mount -t ext4 /dev/sdb /mnt/sdb    # 将新磁盘分区挂载到/mnt/sdb目录下
df -lh                  # 查看磁盘使用情况和挂载情况
umount /dev/sdb         # 卸载分区
echo '/dev/sdb /mnt/sdb ext4 errors=remount-ro 0 1' >> /etc/fstab  # 自动挂载
cat /etc/fstab          # 查看写入分区信息
```

20. 开启KVM

![](../../assets/_images/deploy/ubuntu/20.png)

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo  # 如果返回的结果不是0就说明可以虚拟化
sudo apt install cpu-checker
kvm-ok
```

## 2. 初始化

### 2.1 开启root账号

```bash
sudo passwd root
```

### 2.2 配置静态IP地址

```bash
vi /etc/netplan/00-installer-config.yaml
```

```bash
network:
  ethernets:
    enp0s3:
      addresses:
      - 192.168.2.3/24
      nameservers:
        addresses:
        - 114.114.114.114
        search: []
      routes:
      - to: default
        via: 192.168.2.1
  version: 2
```

```bash
# 检查语法是否正确，如果没有错误，按回车确认
sudo netplan try
# 应用
netplan apply
# 验证DNS生效
nslookup www.baidu.com
```

> 配置网络报错：`Netplan configuration should NOT be accessible by others` 解决办法

```bash
chmod 600 /etc/netplan/00-installer-config.yaml
```

> OpenStack问题：在OpenStack节点重启后，其虚拟网络接口（网卡）在宿主机层面被重新创建，导致其内核设备名称发生变化

找出网卡的MAC地址，并修改配置文件中的`match`字段

```bash
network:
  version: 2
  renderer: networkd
  ethernets:
    my-static-interface:
      match:
        macaddress: fa:16:3e:ea:68:e4
      set-name: ens8
      addresses: [192.168.1.111/22]
      routes:
        - to: default
          via: 192.168.0.1  # 替换为你的网关IP
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]  # 替换为你的DNS服务器
```

### 2.3 ssh配置

```bash
vi /etc/ssh/sshd_config

# 配置文件
Port 22
ListenAddress 0.0.0.0
ListenAddress ::
PermitRootLogin yes         # 允许远程登录
PasswordAuthentication yes  # 开启用户名和密码来验证
```

```bash
/etc/init.d/ssh restart     # 启动服务
update-rc.d ssh enable      # 开机启动
```

### 2.4 关闭防火墙

启动、停止和检查状态
```bash
sudo apt-get install ufw    # 安装防火墙
sudo ufw enable             # 开启防火墙
sudo ufw disable            # 关闭防火墙
sudo ufw status             # 查看防火墙状态
```

增加和删除规则
```bash
sudo ufw allow 8080         # 允许HTTP流量通过端口8080
sudo ufw allow 22/tcp       # 允许SSH流量通过端口22，使用TCP协议
sudo ufw allow from 192.168.1.2 to any port 3306    # 允许192.168.1.2访问本机的3306端口
sudo ufw delete allow 8080  # 删除允许端口8080的规则
sudo ufw show added         # 查看添加的规则
```

设置默认策略
```bash
sudo ufw default allow      # 默认允许外部访问
sudo ufw default deny       # 默认禁止外部访问
```

其他
```bash
sudo ufw reset      # 重置防火墙规则
sudo ufw reload     # 重新加载防火墙规则
sudo ufw limit 22   # 限制22端口连接数
```


### 2.5 更换软件源

```bash
vim /etc/apt/sources.list
deb http://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse
```

更新软件包列表
```bash
apt update
apt upgrade
```

添加自定义软件源，并设置授信

```bash
echo -e "deb [trusted=yes] http://192.168.2.10 jammy main" >/etc/apt/sources.list && apt clean && apt update -y;
```

### 2.6 删除自带的工具

```bash
# 彻底删除 snap 以及其配置文件
sudo apt remove -y snapd --purge

# 删除云功能和自动升级功能
sudo apt remove -y unattended-upgrades cloud-init

# 清理相关依赖
sudo apt install -f && sudo apt autoremove
```

### 2.7 安装常用软件

```bash
apt-get install -y curl vim zip unzip xz-utils telnet lsof wget net-tools iputils-ping ntpdate

# 执行一次性同步
sudo ntpdate ntp4.aliyun.com

# deb安装包
dpkg -i jenkins_2.426.3_all.deb
dpkg --list | grep jenkins
dpkg -r jenkins
```

### 2.8 设置代理

```bash
cat >/etc/profile.d/proxy.sh<<EOF
export http_proxy="http://192.168.2.57:7890"
export https_proxy="http://192.168.2.57:7890"
EOF
```

```bash
source /etc/profile
```


### 2.9 安装Docker

1. 在线安装

```bash
# 卸载
sudo apt-get remove docker docker-engine docker.io containerd runc  
sudo apt-get update
# 安装apt依赖包
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common 
apt-get install ca-certificates curl gnupg lsb-release
# 安装秘钥
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# 添加软件源
sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# 安装
sudo apt-get install docker-ce
# 启动
systemctl start docker
systemctl enable docker
```

2. 安装docker-compose

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

3. 安装Portainer

```bash
docker volume create portainer_data
docker run -d -p 8000:8000 -p 9443:9443 --name portainer \
    --restart=always \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v portainer_data:/data \
    portainer/portainer-ce:2.18.2
```

### 2.10 添加用户

`useradd`更底层的命令，需要手动指定参数，默认情况下不创建主目录或设置密码，适合脚本自动化或需要精细控制用户参数的场景。

```text
-u 设置唯一标识符（UID）
-d 指定主目录，默认主目录在 /home/username
-m 创建主目录
-s 指定登录shell，系统默认 /bin/bash
-g 设置主用户组
-G 设置附加组
-r 创建系统用户，UID小于1000，默认不创建主目录
-p 设置密码
-M 不创建主目录
-c 指定用户的注释信息，对应 /etc/passwd 文件的第五个字段
-e 设置过期日期
-f 密码过期后多少天锁定账户。默认值为 -1，表示禁用此功能
```

`adduser`交互式命令，默认创建主目录，设置密码，适合手动操作或需要快速创建用户并设置密码的场景。

```bash
sudo adduser xzh
```

### 2.11 系统服务管理

#### 2.11.1 使用 systemd 启用 rc.local 服务

1. 创建系统 rc.local 文件

```bash
vi /etc/rc.local
```

```bash
#!/bin/bash
touch /var/log/rc.local
/bin/bash /usr/boot/root
su - xzh -c "/bin/bash /usr/boot/xzh"
```

创建 ROOT 用户启动文件

```bash
mkdir /usr/boot
vi /usr/boot/root
```

```sh
touch ~/root
/usr/local/keepalived/sbin/keepalived -f /etc/keepalived/keepalived.conf
```

创建非 ROOT 用户启动文件

```bash
sudo adduser xzh
vi /usr/boot/xzh
```

```sh
touch ~/xzh
/usr/local/redis/bin/redis-server /usr/local/redis/conf/redis.conf
```

授权

```bash
# 启动命令权限
chown xzh:xzh /usr/boot/xzh
# 程序执行权限
chown -R xzh:xzh /usr/local/redis
# 数据目录权限
chown -R xzh:xzh /data/redis
```


2. 给 rc.local 添加执行权限

```bash
sudo chmod +x /etc/rc.local
```

3. 创建 systemd 服务文件

```bash
vi /etc/systemd/system/rc-local.service
```

```conf
[Unit]
Description=/etc/rc.local Compatibility
ConditionPathExists=/etc/rc.local
After=network.target

[Service]
Type=forking
ExecStart=/etc/rc.local start
TimeoutSec=0
RemainAfterExit=yes
GuessMainPID=no

[Install]
WantedBy=multi-user.target
```

4. 启用并启动服务

```bash
sudo systemctl daemon-reload
sudo systemctl enable rc-local.service
sudo systemctl start rc-local.service
sudo systemctl status rc-local.service
sudo journalctl -u rc-local.service
```

#### 2.11.2 Systemd

Systemd是Linux系统里用来启动和管理服务的工具，是大多数现代Linux发行版（如 Ubuntu、CentOS、Debian、Fedora）默认的初始化系统，它是系统启动后，第一个跑起来的进程（PID 1）


1. 创建服务文件

```bash
vi /etc/systemd/system/mycustom.service
```

2. 添加配置

```conf
[Unit]
Description=My Custom Startup Script
ConditionPathExists=/path/to/your/script.sh
After=network.target

[Service]
Type=simple
ExecStart=/path/to/your/script.sh
TimeoutSec=0
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

3. 启动服务

```bash
sudo systemctl enable mycustom.service
sudo systemctl start mycustom.service
```

4. 验证服务状态

```bash
sudo systemctl status rc-local.service
# 或
sudo systemctl status mycustom.service
# 查看日志
sudo journalctl -u rc-local.service
```

5. 参考示例（jenkins）

```bash
vim /usr/lib/systemd/system/jenkins.service
```

```conf
#
# This file is managed by systemd(1). Do NOT edit this file manually!
# To override these settings, run:
#
#     systemctl edit jenkins
#
# For more information about drop-in files, see:
#
#     https://www.freedesktop.org/software/systemd/man/systemd.unit.html
#

[Unit]
Description=Jenkins Continuous Integration Server
Requires=network.target
After=network.target
StartLimitBurst=5
StartLimitIntervalSec=5m

[Service]
Type=notify
NotifyAccess=main
ExecStart=/usr/bin/jenkins
Restart=on-failure
SuccessExitStatus=143

# Configures the time to wait for start-up. If Jenkins does not signal start-up
# completion within the configured time, the service will be considered failed
# and will be shut down again. Takes a unit-less value in seconds, or a time span
# value such as "5min 20s". Pass "infinity" to disable the timeout logic.
#TimeoutStartSec=90

# Unix account that runs the Jenkins daemon
# Be careful when you change this, as you need to update the permissions of
# $JENKINS_HOME, $JENKINS_LOG, and (if you have already run Jenkins)
# $JENKINS_WEBROOT.
User=jenkins
Group=jenkins

# Directory where Jenkins stores its configuration and workspaces
Environment="JENKINS_HOME=/data/jenkins"
WorkingDirectory=/data/jenkins
```

### 2.12 时间同步服务（Chrony）

1. 服务端安装

```bash
# 更新包列表
sudo apt update

# 安装 Chrony 服务
sudo apt install chrony -y
```

2. 修改配置


```bash
# 备份默认配置文件
cp /etc/chrony/chrony.conf /etc/chrony/chrony.conf.bak
```


```bash
# 去掉注释行，生成文件
egrep -v "(^#|^$)" /etc/chrony/chrony.conf.bak > /etc/chrony/chrony.conf 
vi /etc/chrony/chrony.conf
```

```conf
# 监听所有IP的命令端口
bindcmdaddress 0.0.0.0
# 设置上游服务器
pool ntp4.aliyun.com iburst
# 允许同步的客户端网络
allow 172.17.17.0/24
# 设置本地时间层（当无法同步外部源时）
local stratum 10

confdir /etc/chrony/conf.d
sourcedir /run/chrony-dhcp
sourcedir /etc/chrony/sources.d
keyfile /etc/chrony/chrony.keys
driftfile /var/lib/chrony/chrony.drift
ntsdumpdir /var/lib/chrony
logdir /var/log/chrony
maxupdateskew 100.0
rtcsync
makestep 1 3
leapsectz right/UTC
```

3. 启动服务

```bash
sudo systemctl start chrony
sudo systemctl enable chrony
sudo systemctl status chrony
```

4. 常用命令

```bash
# 设置本地时区
timedatectl set-timezone Asia/Shanghai
# 查看同步状态
chronyc sources -v
# 查看服务器信息
chronyc tracking
# 查看已连接的客户端
chronyc clients
```


5. 客户端测试

```bash
# 安装 Chrony 服务
sudo apt install chrony -y
```

```bash
# 去掉注释行，生成文件
egrep -v "(^#|^$)" /etc/chrony/chrony.conf.bak > /etc/chrony/chrony.conf 
vi /etc/chrony/chrony.conf
```

```conf
server 172.17.17.160 minpoll 5 maxpoll 10 maxdelay .05

confdir /etc/chrony/conf.d
sourcedir /run/chrony-dhcp
sourcedir /etc/chrony/sources.d
keyfile /etc/chrony/chrony.keys
driftfile /var/lib/chrony/chrony.drift
ntsdumpdir /var/lib/chrony
logdir /var/log/chrony
maxupdateskew 100.0
rtcsync
makestep 1 3
leapsectz right/UTC
```

启动服务

```bash
sudo systemctl start chrony
sudo systemctl enable chrony
sudo systemctl status chrony
```


常用命令

```bash
# 设置本地时间
date -s "2025-12-10 15:20:30"
```