# SQL Server 2017

## 1. 安装

### 1.1 单机

#### 1.1.1 更新yum

```bash
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo 
wget -O /etc/yum.repos.d/mssql-server.repo https://packages.microsoft.com/config/rhel/7/mssql-server-2017.repo
yum makecache 
```

#### 1.1.2 安装mssql-server

```bash
yum install -y mssql-server
# 离线安装
wget https://packages.microsoft.com/rhel/7/mssql-server-2017/mssql-server-14.0.3445.2-4.x86_64.rpm
yum localinstall mssql-server-14.0.3445.2-4.x86_64.rpm
```

#### 1.1.3 初始化

```bash
sudo /opt/mssql/bin/mssql-conf setup    # 选择Developer版本，密码：1234Qwer
# 配置文件路径（可默认）
vi /opt/mssql/bin/mssql-conf
```

#### 1.1.4 启动服务

```bash
systemctl status mssql-server # 安装默认开机启动
systemctl stop mssql-server
systemctl start mssql-server
```

#### 1.1.5 客户端测试


1. 安装客户端

```bash
wget -O  /etc/yum.repos.d/msprod.repo https://packages.microsoft.com/config/rhel/7/prod.repo
yum install -y mssql-tools unixODBC-devel # 输入两次yes进行确认
```

2. 配置环境变量

```bash
echo "export PATH=$PATH:/opt/mssql-tools/bin" >> /etc/profile
source /etc/profile
```

3. 测试

```bash
sqlcmd -S localhost -U SA -p 1234Qwer
```

```sql
create database datax
go
create table pms_product (
    id int,
    name varchar(100),   
    brand_name varchar(100),  
    create_time datetime
)
go
insert into pms_product values (1,'nnn','xxx',getdate())
go
select * from pms_product
go
```

#### 1.1.6 卸载

```bash
yum remove mssql-server
rm -rf /var/opt/mssql/
```
