# RabbitMQ 3.9.12

RabbitMQ是一个开源的消息代理的队列服务器，用来通过普通协议在完全不同的应用之间共享数据。是使用Erlang语言来编写的，基于AMQP协议

- 官方网站：https://www.rabbitmq.com/
- rabbitmq下载地址: https://github.com/rabbitmq/rabbitmq-server/releases/tag/v3.9.12
- erlang下载地址: https://github.com/rabbitmq/erlang-rpm/releases/tag/v23.3.4.10

## 1. 安装

### 1.1 上传解压

```bash
mkdir -p /opt/software
cd /opt/software            # 上传文件
yum install -y gcc socat    # 安装依赖
rpm -ivh erlang-23.3.4.10-1.el7.x86_64.rpm
rpm -ivh rabbitmq-server-3.9.12-1.el7.noarch.rpm
```

### 1.2 修改配置

配置参数
- https://github.com/rabbitmq/rabbitmq-server/blob/master/deps/rabbit/docs/rabbitmq.conf.example
- https://github.com/rabbitmq/rabbitmq-server/blob/master/deps/rabbit/docs/advanced.config.example
- https://www.rabbitmq.com/configure.html#config-items

#### 1.2.1 新建配置文件

```bash
cd /etc/rabbitmq
vi rabbitmq.conf
```

```conf
# 数据管理端口（默认端口为5672）
listeners.tcp.default=5762
# 界面管理端口（默认端口为15672）
management.tcp.port=15672
management.tcp.ip=0.0.0.0
```

#### 1.2.2 加载配置文件

```bash
cd /usr/lib/rabbitmq/lib/rabbitmq_server-3.9.12/sbin/
vi rabbitmq-defaults
# 添加配置文件路径
CONFIG_FILE=/etc/rabbitmq/rabbitmq.conf
```

### 1.3 插件管理

#### 1.3.1 延时队列

下载安装

```bash
wget https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/releases/download/3.9.0/rabbitmq_delayed_message_exchange-3.9.0.ez
cp rabbitmq_delayed_message_exchange-3.9.0.ez /usr/lib/rabbitmq/lib/rabbitmq_server-3.9.12/plugins/
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```

常用插件端口号

```bash
rabbitmq-plugins enable rabbitmq_mqtt               # 1883
rabbitmq-plugins enable rabbitmq_web_mqtt           # 15675
rabbitmq-plugins enable rabbitmq_stomp              # 61613
rabbitmq-plugins enable rabbitmq_web_stomp          # 15674
```

### 1.4 用户权限设置

开启管理界面
```bash
rabbitmq-plugins enable rabbitmq_management
```

添加用户
```bash
rabbitmqctl add_user admin 123456
```

用户授权，administartor为管理员权限，四种权限【management、policymaker、monitoring、administrator】
```bash
rabbitmqctl set_user_tags admin administrator
```

开启用户远程访问
```bash
rabbitmqctl set_permissions -p "/" admin ".*" ".*" ".*"
```


### 1.5 启动服务

```bash
systemctl start rabbitmq-server
# 查看日志
cd /var/log/rabbitmq
```

访问地址：http://0.0.0.0:15672


## 2. 命令

```bash
rabbitmqctl environment           # 查看有效配置
rabbitmq-server                   # 当前窗口启动 rabbitmq
rabbitmq-server -detached         # 后台启动 rabbitmq
rabbitmqctl stop                  # 停止 rabbitmq
rabbitmqctl list_queues           # 查看所有队列
rabbitmqctl list_vhosts           # 查看所有虚拟主机
rabbitmqctl start_app             # 在Erlang VM运行的情况下启动RabbitMQ应用
rabbitmqctl stop_app
rabbitmqctl status                # 查看节点状态
rabbitmq-plugins list             # 查看所有可用的插件
rabbitmq-plugins enable <plugin-name>                 # 启用插件
rabbitmq-plugins disable <plugin-name>                # 停用插件
rabbitmqctl add_user username password                # 添加用户
rabbitmqctl list_users                                # 列出所有用户
rabbitmqctl delete_user username                      # 删除用户
rabbitmqctl clear_permissions -p vhostpath username   # 清除用户权限
rabbitmqctl list_user_permissions username            # 列出用户权限
rabbitmqctl change_password username newpassword      # 修改密码
rabbitmqctl set_permissions -p vhostpath username ".*" ".*" ".*"    # 设置用户权限
rabbitmqctl add_vhost vhostpath                       # 创建虚拟主机
rabbitmqctl list_permissions -p vhostpath             # 列出虚拟主机上的所有权限
rabbitmqctl delete_vhost vhost vhostpath              # 删除虚拟主机

rabbitmqctl cluster_status                            # 查看集群状态
rabbitmqctl set_cluster_name xzh_cluster              # 设置集群名称
rabbitmqctl forget_cluster_node rabbit@rabbit-node3   # 移除节点/下线  
```

## 3. 集群

### 3.1 单机多实例

#### 3.1.1 创建配置文件

```bash
cd /etc/rabbitmq
```

```bash
echo "listeners.tcp.default = 5671
management.listener.port = 15671
vm_memory_high_watermark.relative = 0.2
vm_memory_high_watermark_paging_ratio = 0.2
disk_free_limit.absolute = 1GB
cluster_partition_handling = autoheal
default_vhost = /">>rabbitmq1.conf
```

```bash
echo "listeners.tcp.default = 5672
management.listener.port = 15672
vm_memory_high_watermark.relative = 0.2
vm_memory_high_watermark_paging_ratio = 0.2
disk_free_limit.absolute = 1GB
cluster_partition_handling = autoheal
default_vhost = /">>rabbitmq2.conf
```

```bash
echo "listeners.tcp.default = 5673
management.listener.port = 15673
vm_memory_high_watermark.relative = 0.2
vm_memory_high_watermark_paging_ratio = 0.2
disk_free_limit.absolute = 1GB
cluster_partition_handling = autoheal
default_vhost = /">>rabbitmq3.conf
```

#### 3.1.2 启动服务

```bash
RABBITMQ_NODE_PORT=5671 RABBITMQ_NODENAME=rabbit1 RABBITMQ_SERVER_START_ARGS="-rabbitmq_management listener [{port,15671}]"  RABBITMQ_CONFIG_FILE=/etc/rabbitmq/rabbitmq1.conf rabbitmq-server -detached
RABBITMQ_NODE_PORT=5672 RABBITMQ_NODENAME=rabbit2 RABBITMQ_SERVER_START_ARGS="-rabbitmq_management listener [{port,15672}]"  RABBITMQ_CONFIG_FILE=/etc/rabbitmq/rabbitmq2.conf rabbitmq-server -detached
RABBITMQ_NODE_PORT=5673 RABBITMQ_NODENAME=rabbit3 RABBITMQ_SERVER_START_ARGS="-rabbitmq_management listener [{port,15673}]"  RABBITMQ_CONFIG_FILE=/etc/rabbitmq/rabbitmq3.conf rabbitmq-server -detached
```

#### 3.1.3 rabbit2和rabbit3加入集群

1号节点启动
```bash
rabbitmqctl -n rabbit1 stop_app 
rabbitmqctl -n rabbit1 reset
rabbitmqctl -n rabbit1 start_app
```

2号节点加入
```bash
rabbitmqctl -n rabbit2 stop_app
rabbitmqctl -n rabbit2 reset
rabbitmqctl -n rabbit2 join_cluster rabbit1@localhost
rabbitmqctl -n rabbit2 start_app
```

3号节点加入
```bash
rabbitmqctl -n rabbit3 stop_app
rabbitmqctl -n rabbit3 reset
rabbitmqctl -n rabbit3 join_cluster rabbit1@localhost
rabbitmqctl -n rabbit3 start_app
```

验证集群状态

```bash
rabbitmqctl cluster_status -n rabbit1
```

#### 3.1.4 添加用户授权

```bash
rabbitmq-plugins -n rabbit1 enable rabbitmq_management
rabbitmqctl -n rabbit1 add_user admin 123456
rabbitmqctl -n rabbit1 set_permissions -p "/" admin  ".*" ".*" ".*"
rabbitmqctl -n rabbit1 set_user_tags admin administrator
```

#### 3.1.5 开启镜像同步

```
rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all"}'
```

### 3.2 镜像队列集群

#### 3.2.1 初始化

1. 分别设置主机名称

```bash
hostnamectl set-hostname rabbit-node1
```

2. 修改每台机器的/etc/hosts 文件

```bash
cat >> /etc/hosts <<EOF
192.168.3.200 rabbit-node1
192.168.3.201 rabbit-node2
192.168.3.202 rabbit-node3
EOF
```

#### 3.2.2 配置Erlang Cookie

机器之间通信借助于erlang进行消息传输，所以要求集群中所有节点必须有相同的erlang.cookie，将主节点的文件同步到node2和node3

```bash
systemctl stop rabbitmq-server
scp /var/lib/rabbitmq/.erlang.cookie root@rabbit-node2:/var/lib/rabbitmq/
scp /var/lib/rabbitmq/.erlang.cookie root@rabbit-node3:/var/lib/rabbitmq/
chmod 600 /var/lib/rabbitmq/.erlang.cookie
```

#### 3.2.3 启动服务

停掉所有的MQ节点然后使用集群的方式启动，三台机器分别执行

```bash
systemctl start rabbitmq-server
```

#### 3.2.4 加入集群

在node2和node3分别执行以下命令，使rabbit-node2加入node1，rabbit-node3加入node1，--ram标识内存节点，集群必须保证有一个磁盘节点

```bash
rabbitmqctl stop_app        
rabbitmqctl reset           
rabbitmqctl join_cluster --ram rabbit@rabbit-node1
rabbitmqctl start_app    
```

#### 3.2.5 添加用户授权

```bash
rabbitmq-plugins enable rabbitmq_management
rabbitmqctl add_user admin 123456
rabbitmqctl set_user_tags admin administrator
rabbitmqctl set_permissions -p "/" admin ".*" ".*" ".*"
```

#### 3.2.6 开启镜像同步

将所有队列设置为镜像队列，在主节点执行

```bash
rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all"}'
```

## 4. 高可用

1. [HAProxy安装](deploy/haproxy)
2. [KeepAlived安装](deploy/keepalived)