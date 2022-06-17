# Redhat 7

## 1. VMware 12

![](../../assets/_images/devops/linux/rhel7/Vm1.png)
![](../../assets/_images/devops/linux/rhel7/Vm2.png)
![](../../assets/_images/devops/linux/rhel7/Vm3.png)
![](../../assets/_images/devops/linux/rhel7/Vm4.png)
![](../../assets/_images/devops/linux/rhel7/Vm5.png)
![](../../assets/_images/devops/linux/rhel7/Vm6.png)
![](../../assets/_images/devops/linux/rhel7/Vm7.png)

设置虚拟网络

![](../../assets/_images/devops/linux/rhel7/Vm8.png)

?> 桥接模式，必须手动选择宿主机对应的网卡，同时确认VMware Bridge Protocol被勾选，DNE LightWeight Filter未被勾选

![](../../assets/_images/devops/linux/rhel7/Vm9.png)

![](../../assets/_images/devops/linux/rhel7/Vm9_win.png)

?>将默认的 VMnet8 （NET模式） 网卡设置为 192.168.109.0/255.255.255.0，这样宿主机 windows 系统会默认获得192.168.109.1的IP，导入虚拟机镜像后可以将虚拟机网卡选择为 VMnet8 ，可以实现虚拟机 linux与外网通讯

![](../../assets/_images/devops/linux/rhel7/Vm10.png)


?>将默认的 VMnet1 （仅主机模式） 网卡设置为 192.168.154.0/255.255.255.0，这样宿主机 windows 系统会默认获得192.168.154.1的IP，导入虚拟机镜像后可以将虚拟机网卡选择为 VMnet1 ，可以实现windows与虚拟机linux网络通信

![](../../assets/_images/devops/linux/rhel7/Vm11.png)


![](../../assets/_images/devops/linux/rhel7/Vm-net.png)

## 2. 安装

创建虚拟机

![](../../assets/_images/devops/linux/rhel7/1.png)
![](../../assets/_images/devops/linux/rhel7/2.png)
![](../../assets/_images/devops/linux/rhel7/3.png)
![](../../assets/_images/devops/linux/rhel7/4.png)
![](../../assets/_images/devops/linux/rhel7/5.png)
![](../../assets/_images/devops/linux/rhel7/6.png)
![](../../assets/_images/devops/linux/rhel7/7.png)
![](../../assets/_images/devops/linux/rhel7/8.png)
![](../../assets/_images/devops/linux/rhel7/9.png)

系统初始化

![](../../assets/_images/devops/linux/rhel7/10_start.png)

选择 Install Red Hat Enterprise Linux 7.9

![](../../assets/_images/devops/linux/rhel7/10.png)

语言选择界面，正式生产服务器建议安装英文版本，Continue继续

![](../../assets/_images/devops/linux/rhel7/11.png)

选择-系统SYSTEM-安装位置INSTALLTION DESTINATION，进入磁盘分区界面

![](../../assets/_images/devops/linux/rhel7/12.png)

选择-Other Storage Options-Partitoning-I will configure partitioning，点左上角的 Done ，进入下面的界面

![](../../assets/_images/devops/linux/rhel7/13.png)

```
新挂载点使用以下分区方案：标准Standard Partition
分区前先规划好，swap 交换分区，一般设置为内存的2倍，/ 剩余所有空间
备注：生产服务器建议单独再划分一个/data分区存放数据，把数据和系统分开
```

![](../../assets/_images/devops/linux/rhel7/14.png)
![](../../assets/_images/devops/linux/rhel7/15.png)
![](../../assets/_images/devops/linux/rhel7/16.png)
![](../../assets/_images/devops/linux/rhel7/17.png)

接受更改Accept Changes，进入下面的界面
    
![](../../assets/_images/devops/linux/rhel7/18.png)

选择-SOFTWARE-软件选择SOFTWARE SELECTION，我们使用的是基础设置服务器。

![](../../assets/_images/devops/linux/rhel7/19.png)

选择-LOCALIZATION-DATE & TIME 设置时区，中国范围内建议选择上海，并选择24小时制，单击完成按钮

![](../../assets/_images/devops/linux/rhel7/20.png)

选择-系统SYSTEM-SECURITY设置

![](../../assets/_images/devops/linux/rhel7/21.png)

选择default（默认的）策略就可以，通过select profile进行选择，单击完成即可

![](../../assets/_images/devops/linux/rhel7/22.png)

选择安装源

![](../../assets/_images/devops/linux/rhel7/23.png)

单击验证，验证光盘或镜像是否完整，防止安装过程出现软件包不完整，导致无法安装

![](../../assets/_images/devops/linux/rhel7/24.png)
![](../../assets/_images/devops/linux/rhel7/25.png)

选择额外软件仓库，可以在安装时检测是否有更新的软件包，进行更新安装，如果没有也可以手动添加新的网络仓库，然后单击完成按钮

![](../../assets/_images/devops/linux/rhel7/26.png)

KDUMP设置

![](../../assets/_images/devops/linux/rhel7/27.png)
![](../../assets/_images/devops/linux/rhel7/28.png)

网络配置，开启以太网连接，将会自动获取IP地址，如果要手动配置，单击配置，`请根据虚拟机所在网络调整`

![](../../assets/_images/devops/linux/rhel7/29.png)
![](../../assets/_images/devops/linux/rhel7/30.png)
![](../../assets/_images/devops/linux/rhel7/31.png)

全部配置完成，单击开始安装，设置管理员密码，等待系统重启

![](../../assets/_images/devops/linux/rhel7/32.png)
![](../../assets/_images/devops/linux/rhel7/33.png)


## 3. 虚拟机

### 3.1 yum更换

```bash
rpm -qa | grep yum
echo `rpm -qa | grep yum` > aa
# 删除yum程序
rpm -e yum-langpacks-0.4.2-7.el7.noarch --nodeps
rpm -e yum-3.4.3-168.el7.noarch --nodeps
rpm -e yum-metadata-parser-1.1.4-10.el7.x86_64 --nodeps
rpm -e yum-rhn-plugin-2.0.1-10.el7.noarch --nodeps
rpm -e yum-utils-1.1.31-54.el7_8.noarch --nodeps
# 下载yum安装包
wget http://mirrors.163.com/centos/7/os/x86_64/Packages/PackageKit-yum-1.1.10-2.el7.centos.x86_64.rpm
wget http://mirrors.163.com/centos/7/os/x86_64/Packages/yum-3.4.3-168.el7.centos.noarch.rpm
wget http://mirrors.163.com/centos/7/os/x86_64/Packages/yum-langpacks-0.4.2-7.el7.noarch.rpm
wget http://mirrors.163.com/centos/7/os/x86_64/Packages/yum-metadata-parser-1.1.4-10.el7.x86_64.rpm
wget http://mirrors.163.com/centos/7/os/x86_64/Packages/yum-rhn-plugin-2.0.1-10.el7.noarch.rpm
wget http://mirrors.163.com/centos/7/os/x86_64/Packages/yum-utils-1.1.31-54.el7_8.noarch.rpm
# 安装并设置yum源地址
rpm -ivh *.rpm --force --nodeps
vi /etc/yum.repos.d/CentOS-Base.repo
```

```conf
[base]
name=CentOS-$7 - Base - 163.com
#mirrorlist=http://mirrorlist.centos.org/?release=$7&arch=$basearch&repo=os
baseurl=http://mirrors.163.com/centos/7/os/$basearch/
gpgcheck=1
gpgkey=http://mirrors.163.com/centos/RPM-GPG-KEY-CentOS-7
#released updates

[updates]
name=CentOS-$7 - Updates - 163.com
#mirrorlist=http://mirrorlist.centos.org/?release=$7&arch=$basearch&repo=updates
baseurl=http://mirrors.163.com/centos/7/updates/$basearch/
gpgcheck=1
gpgkey=http://mirrors.163.com/centos/RPM-GPG-KEY-CentOS-7
#additional packages that may be useful

[extras]
name=CentOS-$7 - Extras - 163.com
#mirrorlist=http://mirrorlist.centos.org/?release=$7&arch=$basearch&repo=extras
baseurl=http://mirrors.163.com/centos/7/extras/$basearch/
gpgcheck=1
gpgkey=http://mirrors.163.com/centos/RPM-GPG-KEY-CentOS-7
#additional packages that extend functionality of existing packages

[centosplus]
name=CentOS-$7 - Plus - 163.com
baseurl=http://mirrors.163.com/centos/7/centosplus/$basearch/
gpgcheck=1
enabled=0
gpgkey=http://mirrors.163.com/centos/RPM-GPG-KEY-CentOS-7
```

```
yum clean all
```

### 3.1 网络设置

```bash
vi /etc/sysconfig/network-scripts/ifcfg-ens33   
systemctl restart NetworkManager    # 重启网络
ifdown ens33; ifup ens33
```