# 小米路由器 Mini（R1CM）刷机操作说明

小米路由器MINI（R1CM）是小米公司于2014年发布的一款双频智能路由器。它搭载联发科MT7620A处理器，适合作为轻量级网络实验或OpenWrt学习平台。

## 1. 硬件配置

- 产品型号： 小米路由器 Mini
- 处理器： MediaTek MT7620A，单核，主频 580MHz
- 运行内存： 128MB DDR2 RAM
- 闪存： 16MB SPI Flash
- 2.4GHz 无线： 集成于 MT7620A，支持 802.11b/g/n，2×2 MIMO，最高理论速率 300Mbps
- 5GHz 无线： MediaTek MT7612E，支持 802.11a/b/g/n/ac，2×2 MIMO，最高理论速率 867Mbps
- 无线规格： 双频 AC1200，2×2 MIMO
- 有线接口： 1 个 10/100M 自适应 WAN 口，2 个 10/100M 自适应 LAN 口
- USB 接口： 1 个 USB 2.0 接口
- 天线： 2 根外置双频全向天线，支持角度调节
- 固件支持： 原厂 MiWiFi ROM及部分 OpenWrt、Padavan 等第三方固件，具体取决于硬件版本和固件适配情况
- 电源规格： 12V/1A DC 电源适配器，路由器本体为 DC 电源接口

## 2. 获取 SSH 密码

1. 下载 [小米路由器客户端](https://www1.miwifi.com/miwifi_download.html)，登录小米账号，会自动提示您绑定路由器。
2. 登陆 [小米官网](http://d.miwifi.com/rom/ssh) 获取 SSH 密码。
3. 下载工具包，按使用方法引导操作。

![](../../assets/_images/router/xiaomi/ssh.png)

## 3. 刷入旧版固件

因为小米的新固件更换了密钥，直接按照官网教程刷 miwifi_ssh.bin 会出错导致路由器亮红灯，故需刷入旧版固件后再开启 SSH 。

1. 将`小米路由器Mini-R1CM_C3480_2.3.76（降级固件）.bin`文件复制到U盘（FAT/FAT32格式）的根目录下，保证文件名为`miwifi.bin`。
2. 断开小米路由器的电源，将U盘插入USB接口。
3. 按住reset键之后重新接入电源，指示灯变为黄色闪烁状态即可松开reset键。
4. 等待约5-10分钟，指示灯变为蓝色即表示刷机完成。

## 4. 开启 SSH

接下来我们就可以按照官网的教程来开启 SSH 了。以下步骤的 2、3、4 步与上面完全相同

1. 将 miwifi_ssh.bin 拷贝入 U 盘根目录，同时删除 miwifi.bin。
2. 将 U 盘插入路由器 USB 接口，拔掉路由器电源线。
3. 用尖锐物抵住路由器 reset 孔不松，同时接通路由器电源，直到路由器前置 LED 变为闪烁黄灯方可松手。
4. 等待一会，待路由器前置 LED 变为蓝色常亮即成功。

## 5. 备份分区

原厂固件下用 `dd` 备份，是救砖或者回到原厂系统的最后手段。（我没有备份）

1. 使用putty登陆路由器，主机名称192.168.31.1 端口号22 连接类型ssh，登陆时有提示就点击"是"
![](../../assets/_images/router/xiaomi/bakup_1.png)

2. 用户名是你在小米的SSH页面获取的root，密码是后面的数字
![](../../assets/_images/router/xiaomi/bakup_2.png)

3. 使用 `df –h` 命令查看插入 U 盘的盘符。（当前是sdb1）
![](../../assets/_images/router/xiaomi/bakup_3.png)

4. 使用 `cat /proc/mtd` 查看分区情况，如下图
![](../../assets/_images/router/xiaomi/bakup_4.png)

5. 开始备份，将 `sdb1` 替换成实际U盘的盘符，逐行执行

```bash
dd if=/dev/mtd0 of=/extdisks/sdb1/0-all.bin
dd if=/dev/mtd1 of=/extdisks/sdb1/1-bootloader.bin
dd if=/dev/mtd2 of=/extdisks/sdb1/2-config.bin
dd if=/dev/mtd3 of=/extdisks/sdb1/3-Factory.bin
dd if=/dev/mtd4 of=/extdisks/sdb1/4-OS1.bin
dd if=/dev/mtd5 of=/extdisks/sdb1/5-rootfs.bin
dd if=/dev/mtd6 of=/extdisks/sdb1/6-OS2.bin
dd if=/dev/mtd7 of=/extdisks/sdb1/7-overlay.bin
dd if=/dev/mtd8 of=/extdisks/sdb1/8-crash.bin
dd if=/dev/mtd9 of=/extdisks/sdb1/9-reserved.bin
dd if=/dev/mtd10 of=/extdisks/sdb1/10-Bdata.bin
```

![](../../assets/_images/router/xiaomi/bakup_5.png)

## 6. 刷入 breed

1. 使用 winscp 登陆路由器，将 `小米路由器Mini-R1CM-breed.bin` 改名为 `breed.bin` 并上传到路由器的/tmp目录下

2. 使用 putty 登陆路由器，执行 `mtd -r write /tmp/breed.bin Bootloader`, 机器会重启

![](../../assets/_images/router/xiaomi/breed_1.png)

3. 刷完breed，拔掉电源，按住reset键后再给路由器通电，等到路由器灯闪的时候（大概3秒即可），松开reset键，电脑上在浏览器中输入192.168.1.1，就进入breed控制台了

4. 刷入第三方固件，刷入前请先备份【EEPROM】和【编程器固件】
![](../../assets/_images/router/xiaomi/breed_2.png)

5. 点击【固件更新】，选择要刷机的固件即可。最后画面提示，需要手动检查设备状态。本次刷入的是Padavan，路由器地址：192.168.123.1，账密：admin/admin

![](../../assets/_images/router/xiaomi/breed_3.png)
![](../../assets/_images/router/xiaomi/breed_4.png)
![](../../assets/_images/router/xiaomi/breed_5.png)
![](../../assets/_images/router/xiaomi/breed_6.png)
