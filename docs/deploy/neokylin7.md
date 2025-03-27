# NeoKylin Server 7.0

- 官方网站：https://www.kylinos.cn

## 1. 系统安装

1. 选择图形化安装

![](../../assets/_images/deploy/neokylin/1.png)

2. 选择语言

![](../../assets/_images/deploy/neokylin/2.png)

3. 设置系统分区

![](../../assets/_images/deploy/neokylin/3.png)

![](../../assets/_images/deploy/neokylin/3_1.png)

4. 设置语言支持

![](../../assets/_images/deploy/neokylin/4.png)

5. 设置软件选择

![](../../assets/_images/deploy/neokylin/5.png)

![](../../assets/_images/deploy/neokylin/5_1.png)

6. 设置网络和主机名

![](../../assets/_images/deploy/neokylin/6.png)

![](../../assets/_images/deploy/neokylin/6_1.png)

![](../../assets/_images/deploy/neokylin/6_2.png)

7. 设置时间和日期

![](../../assets/_images/deploy/neokylin/7.png)

![](../../assets/_images/deploy/neokylin/7_1.png)

8. KDUMP设置

![](../../assets/_images/deploy/neokylin/8.png)

![](../../assets/_images/deploy/neokylin/8_1.png)

9. 安全策略

![](../../assets/_images/deploy/neokylin/9.png)

10. 设置管理员密码

![](../../assets/_images/deploy/neokylin/10.png)

![](../../assets/_images/deploy/neokylin/10_1.png)

11. 系统重启

![](../../assets/_images/deploy/neokylin/11.png)


## 2. 初始化

### 2.1 关闭防火墙

```bash
systemctl stop firewalld.service
systemctl disable firewalld.service     # 关闭
systemctl enable firewalld.service      # 随系统启动
```


### 2.2 安装常用软件

```bash
# 查看版本
nkvers  

yum install -y zip unzip telnet lsof ntpdate openssh-server wget net-tools.x86_64
yum install -y gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel

# 同步时间
/usr/sbin/ntpdate ntp4.aliyun.com;/sbin/hwclock -w      

```

