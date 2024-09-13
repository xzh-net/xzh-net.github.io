# Windows 10 Enterprise LTSC 2019 (x64) 

## 1. 系统安装

1. 选择语言

![](../../assets/_images/deploy/win10/1.png)

2. 完全安装

![](../../assets/_images/deploy/win10/2.png)

3. 磁盘格式化

![](../../assets/_images/deploy/win10/3.png)

4. 等待安装

![](../../assets/_images/deploy/win10/4.png)

5. 区域设置

![](../../assets/_images/deploy/win10/5.png)

6. 设置键盘布局，然后单击跳过

![](../../assets/_images/deploy/win10/6.png)

![](../../assets/_images/deploy/win10/7.png)

7. 网络设置

![](../../assets/_images/deploy/win10/8.png)

8. 创建用户

![](../../assets/_images/deploy/win10/9.png)

9. 隐私设置

![](../../assets/_images/deploy/win10/10.png)

10. 进入桌面

![](../../assets/_images/deploy/win10/11.png)

![](../../assets/_images/deploy/win10/12.png)

11. 网络设置

![](../../assets/_images/deploy/win10/13.png)

12. 激活

下载地址：https://github.com/massgravel/Microsoft-Activation-Scripts


## 2. 工具

### 2.1 VirtualBox

重置ID

```bash
cd C:\Program Files\Oracle\VirtualBox
VBoxManage internalcommands sethduuid "D:\VirtualBox VMs\centos7\centos7.vdi"
```


