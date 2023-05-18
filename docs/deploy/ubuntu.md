# Ubuntu 22.04.2 LTS

- 官方网址：https://cn.ubuntu.com/
- 下载地址：https://releases.ubuntu.com

## 1. 安装系统

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


## 2. 虚拟机设置

### 2.1 初始化

```bash
sudo passwd root    # 修改密码
apt-get install -y zip unzip telnet lsof wget net-tools
```

### 2.2 ssh配置

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

### 2.3 网络配置

```bash
vi /etc/netplan/00-network-manager-all.yaml
```

```bash
network:
  ethernets:
    enp0s3:
      addresses:
      - 172.17.17.161/24
      nameservers:
        addresses:
        - 114.114.114.114
        search: []
      routes:
      - to: default
        via: 172.17.17.2
  version: 2
```

```bash
netplan apply
```


### 2.4 更换apt源

```bash
vim /etc/apt/sources.list
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
apt update
```

### 2.5 docker

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

```bash
$ docker volume create portainer_data
$ docker run --name portainer
 -d -p 8000:8000 -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock 
-v portainer_data:/data portainer/portainer
```



