# Etcd 3.3.10

Etcd是CoreOS基于Raft协议开发的分布式key-value存储，可用于服务发现、共享配置以及一致性保障（如数据库选主、分布式锁等）

- 官方网站：https://etcd.io
- 下载地址：https://github.com-/etcd-io/etcd/releases/download/v3.3.10/etcd-v3.3.10-linux-amd64.tar.gz

## 1. 安装

### 1.1 上传解压

```bash
cd /opt/software
tar -xzf etcd-v3.3.10-linux-amd64.tar.gz -C /opt/
mv /opt/etcd-v3.3.10-linux-amd64 /opt/etcd
mkdir -p /opt/etcd/{bin,cfg,ssl,tls}
mv /opt/etcd/{etcd,etcdctl} /opt/etcd/bin/
```


### 1.2 下载证书工具

```bash
cd /opt/etcd/tls
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64
cp cfssl_linux-amd64 /usr/local/bin/cfssl
cp cfssljson_linux-amd64 /usr/local/bin/cfssljson
cp cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
```

### 1.3 生成自签CA

```bash
mkdir -p /opt/etcd/tls/{etcd,k8s}
cd /opt/etcd/tls/etcd
```

```bash
vi ca-config.json
```

```conf
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "www": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
```


```bash
vi ca-csr.json
```

```conf
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}
```

```bash
# 生成证书
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
# 查看证书
ls ca*pem
```

### 1.4 使用自签CA签发HTTPS证书

```bash
vi server-csr.json
```

```conf
{
    "CN": "etcd",
    "hosts": [
        "192.168.2.201",
        "192.168.2.202",
        "192.168.2.203"
        ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing"
        }
    ]
}
```


```bash
# 生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server
# 查看证书
ls server*pem
```


### 1.5 修改配置文件

```bash
vi /opt/etcd/cfg/etcd.conf
```

```conf
#[Member]
ETCD_NAME="etcd-1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.2.201:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.2.201:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.2.201:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.2.201:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.2.201:2380,etcd-2=https://192.168.2.202:2380,etcd-3=https://192.168.2.203:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```

### 1.6 设置开启启动

```bash
vi /usr/lib/systemd/system/etcd.service
```

```conf
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/opt/etcd/cfg/etcd.conf
ExecStart=/opt/etcd/bin/etcd \
        --name=${ETCD_NAME} \
        --data-dir=${ETCD_DATA_DIR} \
        --listen-peer-urls=${ETCD_LISTEN_PEER_URLS} \
        --listen-client-urls=${ETCD_LISTEN_CLIENT_URLS},http://127.0.0.1:2379 \
        --advertise-client-urls=${ETCD_ADVERTISE_CLIENT_URLS} \
        --initial-advertise-peer-urls=${ETCD_INITIAL_ADVERTISE_PEER_URLS} \
        --initial-cluster=${ETCD_INITIAL_CLUSTER} \
        --initial-cluster-token=${ETCD_INITIAL_CLUSTER_TOKEN} \
        --initial-cluster-state=new \
        --cert-file=/opt/etcd/ssl/server.pem \
        --key-file=/opt/etcd/ssl/server-key.pem \
        --peer-cert-file=/opt/etcd/ssl/server.pem \
        --peer-key-file=/opt/etcd/ssl/server-key.pem \
        --trusted-ca-file=/opt/etcd/ssl/ca.pem \
        --peer-trusted-ca-file=/opt/etcd/ssl/ca.pem
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

拷贝证书

```bash
cp /opt/etcd/tls/etcd/ca*pem /opt/etcd/tls/etcd/server*pem /opt/etcd/ssl/
```

设置开机启动

```bash
systemctl daemon-reload
systemctl enable etcd
```

### 1.7 应用分发

```bash
scp -r /opt/etcd/ node02:/opt/
scp -r /opt/etcd/ node03:/opt/
scp /usr/lib/systemd/system/etcd.service node02:/usr/lib/systemd/system/
scp /usr/lib/systemd/system/etcd.service node03:/usr/lib/systemd/system/
```

node02修改etcd.conf

```bash
vi /opt/etcd/cfg/etcd.conf
```

```conf
#[Member]
ETCD_NAME="etcd-2"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.2.202:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.2.202:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.2.202:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.2.202:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.2.201:2380,etcd-2=https://192.168.2.202:2380,etcd-3=https://192.168.2.203:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```

node03修改etcd.conf

```bash
vi /opt/etcd/cfg/etcd.conf
```

```conf
#[Member]
ETCD_NAME="etcd-3"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.2.203:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.2.203:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.2.203:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.2.203:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.2.201:2380,etcd-2=https://192.168.2.202:2380,etcd-3=https://192.168.2.203:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```

分别设置开机启动

```bash
systemctl daemon-reload
systemctl enable etcd
```

### 1.8 启动集群

```bash
systemctl start etcd    # 三台机器
netstat -lntup |grep etcd
```

### 1.9 客户端测试

```bash
export ETCDCTL_API=3
/opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://192.168.2.201:2379,https://192.168.2.202:2379,https://192.168.2.203:2379" endpoint health   # 查看集群状态
/opt/etcd/bin/etcdctl member list --write-out table     # 查看成员
/opt/etcd/bin/etcdctl put /root/test/keyOne "xzh20221020"
/opt/etcd/bin/etcdctl get /root/test/keyOne             # 获取数据
/opt/etcd/bin/etcdctl del /root/test/keyOne             # 删除数据
```