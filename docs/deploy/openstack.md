# OpenStack 2024.2

- 官网地址：https://docs.openstack.org/
- 镜像制作：https://docs.openstack.org/image-guide/obtain-images.html
- cirros镜像：http://download.cirros-cloud.net/
- ubuntu镜像：https://cloud-images.ubuntu.com/
- centos镜像：https://cloud.centos.org/centos/7/images/



## 1. 部署

身份认证服务Keystone
计算服务Nova
网络服务Neutron
镜像服务Glance
对象存储服务Swift
块存储服务Cinder
仪表盘Horizon

```bash
vi /etc/cinder/cinder.conf
```

```conf
[DEFAULT]
transport_url = rabbit://openstack:000000@controller
auth_strategy = keystone
my_ip = 10.1.1.45
#enabled_backends = lvm
glance_api_servers = http://controller:9292
enabled_backends = ceph
backup_driver = cinder.backup.drivers.ceph.CephBackupDriver
backup_ceph_conf=/etc/ceph/ceph.conf
backup_ceph_user = cinder-backup
backup_ceph_chunk_size = 4194304
backup_ceph_pool = backups
backup_ceph_stripe_unit = 0
backup_ceph_stripe_count = 0
restore_discard_excess_bytes = true
[database]
connection = mysql+pymysql://cinder:cinderang@controller/cinder
[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = cinder
[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
target_protocol = iscsi
target_helper = tgtadm
volume_backend_name = lvm
[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
[ceph]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_pool = volumes
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = -1
glance_api_version = 2
rbd_user = cinder
rbd_secret_uuid = 1d5f2df8-c8dd-4bc8-85e0-6737284a3f79
volume_backend_name = ceph
```


## 2. 客户端

### 2.1 安装

1. 安装客户端

```bash
# 安装
sudo apt install -y python3-openstackclient
```

2. 配置

```bash
vi /usr/bin/rc
```

```bash
# 添加配置
export OS_AUTH_URL=http://172.17.19.49:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=000000
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
```

3. 验证

```bash
# 加载配置
. /usr/bin/rc
openstack token issue
```

### 2.2 常用命令

#### 2.2.1 域管理

```bash
# 创建域
openstack domain create --description "Demo Domain" default2
# 查看域
openstack domain list               # 查看域列表
openstack domain show default2      # 查看域详情
```

#### 2.2.2 用户管理

```bash
# 创建用户
openstack user create --domain default --password 123456 xuzhihao
# 添加角色
openstack role add --project demo --user xuzhihao admin     
# 查看用户
openstack user list
openstack user show xuzhihao                    # 查看用户详情
openstack user set --name xuzhihao2 xuzhihao    # 修改用户名
openstack user set --password-prompt xuzhihao   # 修改密码
openstack user set --disable xuzhihao           # 禁用用户
openstack user set --enable xuzhihao            # 启用用户
openstack user delete xuzhihao                  # 删除用户
```

#### 2.2.3 角色管理

```bash
# 创建角色
openstack role create admin
# 查看角色
openstack role add --project demo --user xuzhihao admin     # 添加角色
openstack role remove --project demo --user xuzhihao admin  # 移除角色
openstack role assignment list --project demo               # 查看项目角色
```

#### 2.2.4 项目管理
   
```bash
# 创建项目
openstack project create --domain default --description "Project for demo" demo
# 查询项目
openstack project list
openstack project show demo                 # 查看demo详情
openstack project set --name demo2 demo     # 修改项目名
openstack project set --disable demo        # 禁用项目
openstack project set --enable demo         # 启用项目
openstack project delete demo               # 删除项目
```

#### 2.2.5 卷管理

```bash
# 新建卷（size的单位为GB）
openstack volume create --size 2 extvolume2
# 将卷连接到instance
openstack server add volume xzh.test.1 extvolume2
# 查询
openstack volume list                                   # 查看卷列表
openstack volume show test-volume                       # 查看卷详情
openstack volume set --name test-volume3 extvolume2     # 修改卷名
openstack volume set --size 20 extvolume3               # 修改卷大小为20G
openstack volume foece delete extvolume3                # 删除卷
# 使用镜像创建卷
openstack volume create --size 1 --image cirros-0.6.3-x86_64 --availability-zone nova --description "m1.small.volume" m1.small.volume
```

> 如果你在执行这个命令时遇到了关于默认卷类型 ceph 找不到的问题 `The request cannot be fulfilled as the default volume type ceph cannot be found`，你可以通过以下命令来查看可用的卷类型，然后选择一个可用的卷类型来创建卷：

```bash
# 查看卷类型
openstack volume type list
# 创建卷类型
openstack volume type create ceph
# 设置卷类型属性
openstack volume type set --property "volume_backend_name=ceph" ceph
# 删除卷类型
openstack volume type delete ceph
```

```bash
# 新建指定类型的卷
openstack volume create --size 2 --type ceph xzh.volume.1
```


#### 2.2.6 镜像管理

```bash
# 创建镜像
openstack image create --disk-format qcow2 --container-format bare --public --file /opt/cirros-0.6.3-x86_64-disk.img cirros-0.6.3-x86_64
# 删除镜像
openstack image delete cirros-0.6.3-x86_64
# 查看镜像
openstack image list
# 查看镜像详情
openstack image show cirros-0.6.3-x86_64
# 修改镜像属性
openstack image set --property key=value cirros-0.6.3-x86_64
# 删除镜像属性
openstack image unset --property key cirros-0.6.3-x86_64
```


#### 2.2.7 主机类型管理

```bash
# 查询实例类型
openstack flavor list
# 创建实例类型
openstack flavor create --ram 2048 --disk 20 --vcpus 1 m1.small
openstack flavor create --ram 4096 --disk 40 --vcpus 2 m1.medium
openstack flavor create --ram 8192 --disk 80 --vcpus 4 m1.large
# 测试专用实例
openstack flavor create --ram 256 --disk 1 --vcpus 1 m1.small
```


#### 2.2.8 网络管理

```bash
# 可用的网络类型
openstack network agent list
# 查询网络
openstack network list  
# 查询子网
openstack subnet list

# 创建共享wan网
openstack network create  --share --external \
  --provider-physical-network physnet1 \
  --provider-network-type flat wan

# 创建共享wan网子网，子网必须和openstack在同一网段，并且可以连接到网关
openstack subnet create --network wan \
  --allocation-pool start=172.17.19.100,end=172.17.19.150 \
  --dns-nameserver 114.114.114.114  --gateway 172.17.17.254 \
  --subnet-range 172.17.16.0/22 wan-subnet 

# 创建私有lan网
openstack network create --project admin lan
# 创建私有lan网子网
openstack subnet create --network lan \
  --allocation-pool start=192.168.2.160,end=192.168.2.200 \
  --dns-nameserver 114.114.114.114  --gateway 192.168.0.254 \
  --subnet-range 192.168.0.0/16 lan-subnet

openstack network set --name new-network-name old-network-name   # 修改网络名称
openstack network set --shared wan      # 设置网络为共享
openstack network unset --share wan     # 取消网络共享

```

#### 2.2.9 路由管理

```bash
# 创建路由
openstack router create router
# 添加路由到外网
openstack router set --external-gateway wan router
# 添加路由到子网（在控制台设置子网接口的ip地址是子网的网关地址）
openstack router add subnet router lan-subnet
```

#### 2.2.10 实例管理

```bash
 openstack server create
     (--image <镜像> | --volume <卷>)
     --flavor <实例类型>
     [--security-group <安全组>]
     [--key-name <密钥对>]
     [--property <服务器属性>]
     [--file <目的文件名=源文件名>]
     [--user-data <实例注入文件信息>]
     [--availability-zone <域名>]
     [--block-device-mapping <块设备映射>]
     [--nic <net-id=网络ID,v4-fixed-ip=IP地址,v6-fixed-ip=IPv6地址,port-id=端口UUID,auto,none>]
     [--network <网络>]
     [--port <端口>]
     [--hint <键=值>]
     [--config-drive <配置驱动器卷>|True]
     [--min <创建实例最小数量>]
     [--max <创建实例最大数量>]
     [--wait]
     <实例名>
```

```bash
# 镜像创建实例
openstack server create --flavor m1.small --image cirros-0.6.3-x86_64 --security-group mygroup --nic net-id=lan --key-name mykey --wait xzh.test.1
# 卷创建实例，使用 -wait参数用于指示命令执行后，等待相关的异步操作完成，然后再返回结果
openstack server create --flavor m1.small --volume m1.small.volume --security-group mygroup --nic net-id=wan --key-name mykey --wait xzh.test.2

openstack server list                       # 查看实例列表
openstack server show xzh.test.1            # 查看实例详情
openstack console log show xzh.test.1       # 查看控制台日志
# 暂停恢复
openstack server pause <server_id>
openstack server unpause <server_id>
# 挂起恢复
openstack server suspend <server_id>
openstack server resume <server_id>
# 废弃恢复
openstack server shelve <server_id>
openstack server unshelve <server_id>
# 开关机
openstack server start <server_id>
openstack server stop <server_id>
openstack server reboot <server_id>
# 修改删除
openstack server set --user-data <数据文件Base64> <server_id>
openstack server delete <server_id>
# 调整大小
openstack server resize <server_id> <flavor_name>
```

```bash
# 创建实例并注入用户数据
openstack server create --user-data <datafile_name> --flavor m1.small --image cirros-0.6.3-x86_64 --security-group mygroup --nic net-id=lan --key-name mykey --wait xzh.test.3
```

#### 2.2.11 安全组管理

```bash
# 创建安全组
openstack security group create --description "test security group" mygroup
# 查看安全组列表
openstack security group list
# 删除安全组
openstack security group delete mygroup
# 查看安全组详情
openstack security group show mygroup
```

添加规则

```bash
# 允许所有入站的ping流量  
openstack security group rule create --ingress --proto icmp mygroup
# 允许192.168.1网段入站的SSH流量  
openstack security group rule create --ingress --proto tcp --remote-ip 192.168.1.0/24 --dst-port 22 mygroup
# 查看规则列表
openstack security group rule list mygroup
# 删除规则
openstack security group rule delete <rule_id>
```


#### 2.2.12 密钥对管理

```bash
# 创建密钥对
openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey
# 查询密钥
openstack keypair list
# 查看密钥详情
openstack keypair show mykey
# 删除密钥
openstack keypair delete mykey
# 查看公钥
openstack keypair show --public-key mykey
```


#### 2.2.13 浮动IP管理

```bash
# 在外部网络浮动IP池分配ip
openstack floating ip create wan
# 查看浮动ip
openstack floating ip list
# 查看浮动ip详情
openstack floating ip show 172.17.19.181
# 删除浮动ip
openstack floating ip delete 172.17.19.190
# 绑定浮动ip到虚拟机
openstack server add floating ip xzh.test.1 172.17.19.181
# 解绑浮动ip
openstack server remove floating ip xzh.test.1 172.17.19.181
```

#### 2.2.14 快照管理

```bash 

```
