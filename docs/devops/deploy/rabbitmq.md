# RabbitMQ 3.9.12

## 1. 安装

### 1.1 RPM安装

#### 1.1.1 下载

- rabbitmq-server: https://github.com/rabbitmq/rabbitmq-server/releases/tag/v3.9.12
- erlang-rpm: https://github.com/rabbitmq/erlang-rpm/releases/tag/v23.3.4.10

#### 1.1.2 安装

```bash
yum install -y gcc socat  # 安装依赖
cd /opt
mkdir rabbitmq  # 上传文件
cd rabbitmq
rpm -ivh erlang-23.3.4.10-1.el7.x86_64.rpm
rpm -ivh rabbitmq-server-3.9.12-1.el7.noarch.rpm
```

#### 1.1.3 修改配置

- https://github.com/rabbitmq/rabbitmq-server/blob/master/deps/rabbit/docs/rabbitmq.conf.example
- https://github.com/rabbitmq/rabbitmq-server/blob/master/deps/rabbit/docs/advanced.config.example
- https://www.rabbitmq.com/configure.html#config-items

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

```bash
cd /usr/lib/rabbitmq/lib/rabbitmq_server-3.9.12/sbin/
vi rabbitmq-defaults                
# 添加配置文件路径
CONFIG_FILE=/etc/rabbitmq/rabbitmq.conf
```

#### 1.1.4 启动服务

```bash
systemctl start rabbitmq-server
rabbitmq-plugins enable rabbitmq_management       # 启用管理插件

rabbitmqctl add_user root 123456                  # 添加用户
rabbitmqctl set_user_tags root administrator      # 用户授权,administartor为管理员权限，四种权限【management、policymaker、monitoring、administrator】
cd /var/log/rabbitmq                              # 查看日志
```

### 1.2 编译安装

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
rabbitmqctl set_cluster_name my_rabbitmq_cluster      # 设置集群名称
rabbitmqctl forget_cluster_node rabbit@rabbitName     # 移除节点/下线
```

## 3. 集群

### 3.1 单机多实例


#### 3.1.1 启动实例

```bash
sudo RABBITMQ_NODE_PORT=5672 RABBITMQ_NODENAME=rabbit-1 rabbitmq-server start
sudo RABBITMQ_NODE_PORT=5673 RABBITMQ_SERVER_START_ARGS="-rabbitmq_management listener [{port,15673}]" RABBITMQ_NODENAME=rabbit-2 rabbitmq-server start &
```

#### 3.1.2 验证

```bash
ps aux|grep rabbitmq
```

#### 3.1.3 rabbit-1作为主节点

```bash
sudo rabbitmqctl -n rabbit-1 stop_app
sudo rabbitmqctl -n rabbit-1 reset
sudo rabbitmqctl -n rabbit-1 start_app
```

#### 3.1.4 rabbit-2作为从节点

```bash
sudo rabbitmqctl -n rabbit-2 stop_app
sudo rabbitmqctl -n rabbit-2 reset
sudo rabbitmqctl -n rabbit-2 join_cluster rabbit-1@localhost
sudo rabbitmqctl -n rabbit-2 start_app
```

#### 3.1.5 验证集群状态

```bash
sudo rabbitmqctl cluster_status -n rabbit-1
```

#### 3.1.6 集群添加账号

```bash
rabbitmqctl -n rabbit-1 add_user test 123456
rabbitmqctl -n rabbit-1 set_user_tags test administrator 
```

#### 3.1.7 开启镜像同步

```
rabbitmqctl -n rabbit-1 set_policy ha-all "^" '{"ha-mode":"all"}'
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

#### 3.2.4 启动服务

停掉所有的MQ节点然后使用集群的方式启动，三台机器分别执行

```bash
systemctl start rabbitmq-server
```

#### 3.2.5 加入集群

在node2和node3分别执行以下命令，使rabbit-node2加入node1, rabbit-node3加入node1 --ram标识内存节点，集群必须保证有一个磁盘节点

```bash
rabbitmqctl stop_app        
rabbitmqctl reset           
rabbitmqctl join_cluster --ram rabbit@rabbit-node1
rabbitmqctl start_app    

rabbitmqctl set_cluster_name rabbitmq_cd_itcast     # 修改集群的名字
rabbitmqctl forget_cluster_node rabbit@rabbit-node3 # 移除节点
rabbitmqctl cluster_status                          # 查看集群状态
```

#### 3.2.6 添加用户、授权

```bash
rabbitmqctl add_user root 123456                  
rabbitmqctl set_user_tags root administrator
```

#### 3.2.7 设置镜像队列策略

将所有队列设置为镜像队列，在主节点执行

```bash
rabbitmqctl -n rabbit-1 set_policy ha-all "^" '{"ha-mode":"all"}'
```

### 3.3 高可用

1. [HAProxy安装](devops/deploy/haproxy)
2. [KeepAlived安装](devops/deploy/keepalived)