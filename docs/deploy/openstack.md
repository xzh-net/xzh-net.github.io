# OpenStack 2024.2

- 官网地址：https://docs.openstack.org/
- 镜像下载：http://download.cirros-cloud.net/


## 1. 部署

使用xxx一键部署


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

#### 2.2.5 镜像管理

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

#### 2.2.6 卷管理

```bash
# 创建卷
openstack volume create --size 1 --image cirros-0.6.3-x86_64 --availability-zone nova --description "Test volume" test-volume
# 查询
openstack volume list                                   # 查看卷列表
openstack volume show test-volume                       # 查看卷详情
openstack volume set --name test-volume2 test-volume    # 修改卷名
openstack volume set --size 20 test_volume              # 修改卷大小为20G
openstack volume delete test-volume                     # 删除卷
```

#### 2.2.7 实例类型管理

```bash
# 查询实例类型
openstack flavor list
# 创建实例类型
openstack flavor create --ram 2048 --disk 20 --vcpus 1 m1.small
# 创建实例类型
openstack flavor create --ram 4096 --disk 40 --vcpus 2 m1.medium
# 创建实例类型
openstack flavor create --ram 8192 --disk 80 --vcpus 4 m1.large
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

#### 2.2.11 安全组管理

#### 2.2.12 密钥对管理

#### 2.2.13 浮动IP管理


