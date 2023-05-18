# kali-linux-2021.2

- 官方网址：https://www.kali.org
- 下载地址：http://old.kali.org/kali-images

## 1. 安装系统

1. 选择Graphical install(图形化安装)

![](../../assets/_images/deploy/kali/1.png)

2. 选择【Chinese(Simplified)-中文(简体)】，然后点击Continue

![](../../assets/_images/deploy/kali/2.png)

3. 选择中国然后继续

![](../../assets/_images/deploy/kali/3.png)

4. 选择语言然后继续

![](../../assets/_images/deploy/kali/4.png)

5. 稍等片刻

![](../../assets/_images/deploy/kali/5.png)

6. 【配置网络】这里输入一个主机名，然后继续

![](../../assets/_images/deploy/kali/6.png)

7. 输入域名

![](../../assets/_images/deploy/kali/7.png)

8. 【设置用户名和密码】这里输入一个普通用户账号的用户名，然后继续

![](../../assets/_images/deploy/kali/8.png)

9. 【设置用户名和密码】这里输入一个账号的用户名，并且记住该用户名，然后继续

![](../../assets/_images/deploy/kali/9.png)

10. 【设置用户名和密码】这里为刚刚的用户设置一个密码，然后继续

![](../../assets/_images/deploy/kali/10.png)

11. 【磁盘分区】这里选择【向导-使用整个磁盘】，然后继续

![](../../assets/_images/deploy/kali/11.png)

12. 【磁盘分区】这一步保持默认继续就行

![](../../assets/_images/deploy/kali/12.png)

13. 【对磁盘进行分区】这里分区方案选择推荐的第一个【将所有文件放在同一个分区中(推荐新手使用)】，然后继续

![](../../assets/_images/deploy/kali/13.png)

14. 【对磁盘进行分区】这里我们选择【结束分区设定并将修改写入磁盘】这个选项，然后点击继续

![](../../assets/_images/deploy/kali/14.png)

15. 【对磁盘进行分区】这里我们选`是`然后继续

![](../../assets/_images/deploy/kali/15.png)

16. 这里保持默认，直接点击继续

![](../../assets/_images/deploy/kali/16.png)

17. 耐心等待

![](../../assets/_images/deploy/kali/17.png)

18. 安装GRUB启动引导器】选`是`然后点击继续

![](../../assets/_images/deploy/kali/18.png)

19. 【安装GRUB启动引导器】这里一定得选择/dev/sda，然后继续

![](../../assets/_images/deploy/kali/19.png)

20. 安装完成，点击继续

![](../../assets/_images/deploy/kali/20.png)


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