# ClickHouse 21.9.4.35

ClickHouse 是俄罗斯的 Yandex 于 2016 年开源的列式存储数据库（DBMS），使用 C++语言编写，主要用于在线分析处理查询（OLAP），能够使用 SQL 查询实时生成分析数据报告。

官网地址：https://clickhouse.com/

下载地址：https://repo.yandex.ru/clickhouse/tgz/stable/

## 1. 安装

### 1.1 单机

#### 1.1.1 上传解压

```bash
mkdir -p /opt/{clickhouse,software}
cd /opt/software
tar -zxvf clickhouse-common-static-21.9.4.35.tgz -C /opt/clickhouse
tar -zxvf clickhouse-common-static-dbg-21.9.4.35.tgz -C /opt/clickhouse
tar -zxvf clickhouse-server-21.9.4.35.tgz -C /opt/clickhouse
tar -zxvf clickhouse-client-21.9.4.35.tgz -C /opt/clickhouse
```

#### 1.1.2 安装依赖

```bash
yum install -y libtool
yum install -y *unixODBC*
```

#### 1.1.3 参数设置

1. 禁用SELINUX

```bash
sed -i 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

2. 修改文件资源限制

```bash
vi /etc/security/limits.conf
# 末尾添加
* soft nofile 65536
* hard nofile 65536
* soft nproc 131072
* hard nproc 131072
```

3. 修改用户线程数

```bash
vi /etc/security/limits.d/20-nproc.conf
# 末尾添加
* soft nofile 65536
* hard nofile 65536
* soft nproc 131072
* hard nproc 131072
```

#### 1.1.4 安装脚本

```bash
/opt/clickhouse/clickhouse-common-static-21.9.4.35/install/doinst.sh
/opt/clickhouse/clickhouse-common-static-dbg-21.9.4.35/install/doinst.sh
/opt/clickhouse/clickhouse-server-21.9.4.35/install/doinst.sh
/opt/clickhouse/clickhouse-client-21.9.4.35/install/doinst.sh
```

#### 1.1.5 启动

```bash
# 查看命令
clickhouse --help 
# 启动
clickhouse start 
```

#### 1.1.6 客户端测试

```bash
clickhouse-client -m  # 支持多行语句
clickhouse-client --password 123456
show databases;
create table t_tinylog ( id String, name String) engine=TinyLog;
```

#### 1.1.7 配置目录

```bash
# 命令目录
cd /usr/bin
ll | grep clickhouse
# 配置文件目录
cd /etc/clickhouse-server/
# 日志目录
cd /var/log/clickhouse-server/
# 数据文件目录
cd /var/lib/clickhouse/
# 允许远程访问
cd /etc/clickhouse-server/
vim config.xml   # 找到<listen_host>::</listen_host> 打开注释
# 重启服务
clickhouse restart
```

验证地址：http://192.168.2.201:8123



### 1.2 集群
