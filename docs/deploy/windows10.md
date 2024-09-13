# Windows 10 Enterprise LTSC 2019 (x64) 

## 1. 系统安装

1. 选择语言

![](../../assets/_images/deploy/win10/1.png)

2. 完全安装

![](../../assets/_images/deploy/win10/2.png)

3. 磁盘格式化

![](../../assets/_images/deploy/win10/3.png)

4. 等待安装

![](../../assets/_images/deploy/win10/4.png)

5. 区域设置

![](../../assets/_images/deploy/win10/5.png)

6. 设置键盘布局，然后单击跳过

![](../../assets/_images/deploy/win10/6.png)

![](../../assets/_images/deploy/win10/7.png)

7. 网络设置

![](../../assets/_images/deploy/win10/8.png)

8. 创建用户

![](../../assets/_images/deploy/win10/9.png)

9. 隐私设置

![](../../assets/_images/deploy/win10/10.png)

10. 进入桌面

![](../../assets/_images/deploy/win10/11.png)

![](../../assets/_images/deploy/win10/12.png)

11. 网络设置

![](../../assets/_images/deploy/win10/13.png)

12. 激活

下载地址：https://github.com/massgravel/Microsoft-Activation-Scripts

## 2. 软件工具

### 2.1 VirtualBox

#### 2.1.1 修改UUID

```bash
cd C:\Program Files\Oracle\VirtualBox
VBoxManage internalcommands sethduuid "D:\VirtualBox VMs\centos7\centos7.vdi"
```

### 2.2 MySQL解压版安装

#### 2.2.1 下载

- 下载地址：https://dev.mysql.com/downloads/mysql/

#### 2.2.2 设置环境变量

新建MYSQL_HOME
```
MYSQL_HOME
D:\mysql-8.0.36-winx64
```

编辑PATH
```
PATH
%MYSQL_HOME%\bin
```

设置成功以后使用命令`mysql`进行验证，如果提示Can't connect to MySQL server on 'localhost'则证明添加成功。如果提示mysql不是内部或外部命令，也不是可运行的程序或批处理文件则表示添加添加失败，请重新检查步骤并重试。



#### 2.2.3 初始化配置

在bin的同级目录下新建一个my.ini文件

```ini
[client]
# 设置mysql客户端默认字符集
default-character-set=utf8

[mysql]
# 设置mysql客户端默认字符集
default-character-set=utf8mb4

[mysqld]
# 设置mysql客户端连接服务端时默认使用的端口3306
port = 3306

# 设置mysql的安装目录
basedir = D:\\mysql-8.0.36-winx64

# 设置mysql数据库的数据的存放目录
datadir = D:\\mysql-8.0.36-winx64\\data

# 允许最大连接数
max_connections=200

# 允许连接失败的次数。这是为了防止有人从该主机试图攻击数据库系统
max_connect_errors=10

# 服务端使用的字符集默认为8比特编码的latin1字符集【mysql8.0】
character_set_server = utf8mb4

# 创建新表时将使用的默认存储引擎
default-storage-engine=INNODB

# 默认使用“mysql_native_password”插件认证【mysql8.0】
# default_authentication_plugin=mysql_native_password​​
authentication_policy=*

# 跳过安全检查,如果跳过，可能不能执行修改用户密码sql语句
#skip-grant-tables

#开启查询缓存
#explicit_defaults_for_timestamp=true

# 创建模式 NO_AUTO_CREATE_USER再MYSQL8.0中已经被移除，不能再8.0以上版本配置【mysql8.0】
# sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES 
# sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
```

以管理员身份，运行命令行窗口：

```bash
mysqld --initialize-insecure
# 或者下面方式，生成随机密码
mysqld --initialize --user=mysql --console
```

如果没有出现报错信息，则证明data目录初始化没有问题，此时再查看MySQL目录下已经有data目录生成。

#### 2.2.4 注册服务并启动

```bash
mysqld -install     # 以管理员身份运行
net start mysql     # 启动mysql服务
net stop mysql      # 停止mysql服务
```

#### 2.2.5 登录MySQL

修改默认密码

```bash
mysqladmin -u root password 123456
```

登录

```bash
mysql -uroot -p123456
# mysql -u用户名 -p密码 -h要连接的mysql服务器的ip地址(默认127.0.0.1) -P端口号(默认3306)
```

#### 2.2.6 卸载MySQL

```bash
net stop mysql
mysqld -remove mysql
```