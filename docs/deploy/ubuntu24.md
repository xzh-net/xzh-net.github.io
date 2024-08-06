# Ubuntu 24.04 LTS

- 官方网址：https://cn.ubuntu.com
- 下载地址：https://releases.ubuntu.com

## 1. 命令

## 2. 虚拟机

### 2.1 手动配置IP地址

```bash
vi /etc/netplan/50-cloud-init.yaml
```

```yaml
# This file is generated from information provided by the datasource.  Changes
# to it will not persist across an instance reboot.  To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
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
            -   to: default
                via: 192.168.2.1
    version: 2
```

```bash
netplan apply
```

### 2.2 更换软件源

```bash
vim /etc/apt/sources.list.d/ubuntu.sources
```

```yaml
Types: deb
URIs: https://mirrors.tuna.tsinghua.edu.cn/ubuntu
Suites: noble noble-updates noble-backports
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

更新系统

```bash
sudo apt clean
sudo apt update && sudo apt upgrade -y
```