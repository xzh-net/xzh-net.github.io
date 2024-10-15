# Ubuntu 22.04.2 LTS

- 官方网址：https://cn.ubuntu.com
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

### 2.2 手动配置IP地址

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
netplan apply
```

> 配置网络报错：`Netplan configuration should NOT be accessible by others` 解决办法

```bash
chmod 600 /etc/netplan/00-installer-config.yaml
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
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
apt update      # 更新apt
apt upgrade     # 更新系统
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
apt-get install -y curl vim zip unzip xz-utils telnet lsof wget net-tools iputils-ping
```

### 2.8 安装Docker

#### 2.8.1 在线安装

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

#### 2.8.2 安装docker-compose

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

#### 2.8.3 安装Portainer

```bash
docker volume create portainer_data
docker run -d -p 8000:8000 -p 9443:9443 --name portainer \
    --restart=always \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v portainer_data:/data \
    portainer/portainer-ce:2.18.2
```