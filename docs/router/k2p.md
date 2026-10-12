# K2P A2 刷机操作说明

斐讯 K2P 是一款采用联发科平台的双频千兆无线路由器，支持 IEEE 802.11a/b/g/n/ac 无线网络标准，属于 Wi-Fi 5（AC1200）级别。该设备采用 MediaTek MT7621AT 双核处理器，配备 128MB DDR3 运行内存和 16MB SPI Flash 闪存，搭载 MT7615DN 双频无线芯片，支持 2.4GHz 和 5GHz 双频 Wi-Fi。机身采用金属外壳设计，具备良好的散热条件，并提供 1 个千兆 WAN 口和 4 个千兆 LAN 口，适合家庭宽带接入、千兆局域网及轻量级网络服务应用。该型号也受到 OpenWrt 等第三方固件用户的关注，具有一定的固件扩展和网络功能定制能力。

## 1. 硬件配置

- 产品型号： PHICOMM K2P A2
- 处理器： MediaTek MT7621AT，双核，主频 880MHz
- 运行内存： 128MB DDR3 RAM
- 闪存： 16MB SPI NOR Flash
- 无线芯片： MediaTek MT7615DN
- 2.4GHz 无线速率： 最高理论速率约 400Mbps（需终端支持相应调制技术）
- 5GHz 无线速率： 最高理论速率 867Mbps
- 无线规格： 双频 AC1200，双频并发，2×2 MIMO
- 有线接口： 5 个千兆以太网口，包括 1 个 WAN 口和 4 个 LAN 口
- 天线： 4 根外置天线
- 机身材质： 金属外壳
- 固件支持： 原厂固件及部分 OpenWrt、Padavan 等第三方固件，具体取决于硬件版本和固件适配情况
- 电源规格： 12V DC，1.5A

## 2. 刷入 Breed 引导系统

### 2.1 V22.5.7.85 及更早

如果斐讯官方系统版本为 V22.5.7.85 版本，可以在 `功能设置-手动升级` 页面直接导入 `k2p_mtk_v14_breed.bin`，成功升级之后按照 Breed 说明刷入第三方固件。

![](../../assets/_images/router/k2p/boot.png)

### 2.2 V22.5.13.27 ~ V22.7.8.2

如果斐讯官方系统版本为 V22.5.13.27 ~ V22.7.8.2 版本，请从官方 BootLoade 刷入 `k2p_mtk_v14_breed.bin`，成功升级之后按照 Breed 说明刷入第三方固件。

1. 断开路由器电源，按住 Reset 键不放，然后插上电源，按住40秒后放开
2. 访问 192.168.2.1 进入官方 Bootloader 界面，选择 `k2p_mtk_v14_breed.bin`，然后点击 `Update firmware`
![](../../assets/_images/router/k2p/bootloade1.png)
![](../../assets/_images/router/k2p/bootloade2.png)

3. 等待重新启动后K2P灯恢复为黄色，访问 192.168.2.1 进入斐讯系统登录界面，表示成功。成功升级之后按照 Breed 说明刷入第三方固件。


### 2.3 V22.7.8.2 及之后

1. 如果当前固件是 V22.7.8.2 版本及之后，需要使用 `RoutAckProV1B2` 工具激活 Telnet

断开路由器电源，按住 Reset 键不放，然后插上电源，按住10秒后放开，此时已恢复出厂设置。等两分钟后指示灯变黄，登录 192.168.2.1，，账号密码：admin/admin，能进入说明ip没问题。

先点击 `打开Telent`，框内显示成功，再点击 `设置永久标志`

![](../../assets/_images/router/k2p/telnet.png)
