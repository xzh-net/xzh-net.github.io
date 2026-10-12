# Newifi D2 V1.2 刷机操作说明

Newifi D2 又称新路由3（Newifi3），是一款采用联发科平台的双频千兆无线路由器，支持 IEEE 802.11a/b/g/n/ac 无线网络标准，属于 Wi-Fi 5（AC1200）级别。该设备采用 MT7621AT 双核处理器，配备 512MB DDR3 运行内存和 32MB 闪存，具有较好的多任务处理能力，并支持 OpenWrt 等第三方固件，适合家庭网络、轻量级软路由应用及网络功能扩展。

## 1. 硬件配置

- 产品型号： Newifi D2（新路由3）
- 处理器： MediaTek MT7621AT，双核，主频 880MHz
- 运行内存： 512MB DDR3 RAM
- 闪存： 32MB SPI NOR Flash
- 2.4GHz 无线芯片： MediaTek MT7603EN，支持 802.11b/g/n，最高理论速率 300Mbps
- 5GHz 无线芯片： MediaTek MT7612EN，支持 802.11a/n/ac，最高理论速率 867Mbps
- 无线规格： 双频 AC1200，2×2 MIMO
- 有线接口： 5 个千兆以太网口，包括 1 个 WAN 口和 4 个 LAN 口
- USB 接口： 1 个 USB 3.0 接口
- 天线： 4 根外置天线
- 固件支持： 原厂固件及部分 OpenWrt、Padavan 等第三方固件，具体取决于硬件版本和固件适配情况
- 电源规格： 12V DC，2A

## 2. 重置密码

长按reset重置路由器，设置无线密码，无线密码等于管理员密码

## 3. 开启 SSH

在原厂系统下，浏览器访问 `http://192.168.99.1/newifi/ifiwen_hss.html` ，看到 "success" 即表示 SSH 已开启

## 4. 刷入 Breed 引导系统

1. 使用 winscp 登陆路由器，将 `newifi-d2-jail-break.ko` 上传到路由器的/tmp目录下

2. 使用 putty 登陆路由器，执行 `insmod newifi-d2-jail-break.ko`, 两分钟左右路由器会自动重启

3. 断开路由器电源，按住 Reset 键不放，然后插上电源。等待约 5-10 秒，观察到电源指示灯闪烁后松开 Reset 键。电脑访问 192.168.1.1 即可进入 Breed Web 界面

4. 刷入第三方固件，刷入前请先备份【EEPROM】和【编程器固件】
![](../../assets/_images/router/xiaomi/breed_2.png)

5. 点击【固件更新】，选择要刷机的固件即可。最后画面提示，需要手动检查设备状态。本次刷入的是Padavan，路由器地址：192.168.123.1，账密：admin/admin

![](../../assets/_images/router/xiaomi/breed_3.png)
![](../../assets/_images/router/xiaomi/breed_4.png)
![](../../assets/_images/router/xiaomi/breed_5.png)
![](../../assets/_images/router/xiaomi/breed_6.png)

如果刷入的固件不满意，只需再次通过"按住 Reset 键上电"的方法进入 Breed 控制台，重新刷写其他固件即可。

## 5. 潘多拉固件解锁SSR

固件开发者在编译时，虽然集成了 SSR Plus 功能，但为了防止固件因包含特定插件而被"和谐"或导致下载链接失效，默认将这个插件的设置页面隐藏了起来。

使用 putty 登陆路由器，执行以下命令

```bash
echo  0xDEADBEEF > /etc/config/google_fu_mode
```

执行完毕后，刷新路由器管理后台的网页。在"服务"菜单下出现"ShadowSocksR Plus+"选项。