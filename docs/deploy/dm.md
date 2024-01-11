# DM 8

## 1. 安装

### 1.1 准备工作

环境CentOS7_x86_64架构，数据库版本dm8_rh7_64_ent_8.1.1.87

安装前必须创建dmdba用户，禁止使用root用户安装数据库。

#### 1.1.1 创建用户

```bash
groupadd dinstall
useradd -g dinstall -m -d /home/dmdba -s /bin/bash dmdba
passwd dmdba
```

#### 1.1.2 修改文件打开最大数

永久生效

```bash
vi /etc/security/limits.conf

dmdba hard nofile 65536
dmdba soft nofile 65536
dmdba hard stack 32768
dmdba soft stack 16384
```

切换到 dmdba 用户，查看是否生效，命令如下：

```bash
su - dmdba
ulimit -a
```

临时生效

```bash
ulimit -n 65536
```

#### 1.1.3 挂载镜像

切换到 root 用户，将 DM 数据库的 iso 安装包保存在任意位置，例如 /opt 目录下，执行如下命令挂载镜像：

```bash
cd /opt
unzip dm8_20230418_x86_rh6_64.zip
mkdir /mnt
mount -o loop /opt/dm8_20230418_x86_rh6_64.iso /mnt
```

#### 1.1.4 创建安装目录

使用 root 用户建立文件夹，待 dmdba 用户建立完成后需将文件所有者更改为 dmdba 用户，否则无法安装到该目录下

```bash
mkdir /dm8
```

#### 1.1.5 修改安装目录权限

将新建的安装路径目录权限的用户修改为 dmdba，用户组修改为 dinstall。命令如下：

```bash
chown dmdba:dinstall -R /dm8/
chmod -R 755 /dm8   # 给安装路径下的文件设置 755 权限
```

### 1.2 数据库安装

DM 数据库在 Linux 环境下支持命令行安装和图形化安装

#### 1.2.1 命令行安装

```bash
su - dmdba
cd /mnt/
./DMInstall.bin -i
```

按需求选择安装语言，默认为中文。本地安装选择【不输入 Key文件】，选择【默认时区 21】。

![](../../assets/_images/deploy/dm/choose-lang-time.png)

选择【1-典型安装】，按已规划的安装目录 /dm8 完成数据库软件安装，不建议使用默认安装目录。

![](../../assets/_images/deploy/dm/choose-type-path.png)

数据库安装大概 1~2 分钟，数据库安装完成后，显示如下界面。

![](../../assets/_images/deploy/dm/install-success.png)

数据库安装完成后，需要切换至 root 用户执行上图中的命令 /dm8/script/root/root_installer.sh 创建 DmAPService，否则会影响数据库备份

#### 1.2.2 配置环境变量

切换到 root 用户进入 dmdba 用户的根目录下，配置对应的环境变量。DM_HOME 变量和动态链接库文件的加载路径在程序安装成功后会自动导入。命令如下：

```bash
cd /home/dmdba/
vim .bash_profile

export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/dm8/bin"
export DM_HOME="/dm8"
export PATH=$PATH:$DM_HOME/bin:$DM_HOME/tool
```

![](../../assets/_images/deploy/dm/dm-home-path.png)

切换至 dmdba 用户下，执行以下命令，使环境变量生效。

```bash
su - dmdba
source .bash_profile
```

### 1.3 配置实例

使用 dmdba 用户配置实例，进入到 DM 数据库安装目录下的 bin 目录中，使用 dminit 命令初始化实例

```bash
cd /dm8/bin
./dminit help
```

![](../../assets/_images/deploy/dm/ml-licence-dminithelp.png)

需要注意的是页大小 (page_size)、簇大小 (extent_size)、大小写敏感 (case_sensitive)、字符集 (charset) 这四个参数，一旦确定无法修改，需谨慎设置
   - extent_size 指数据文件使用的簇大小，即每次分配新的段空间时连续的页数。只能是 16 页或 32 页或 64 页之一，缺省使用 16 页。
   - page_size 数据文件使用的页大小，可以为 4 KB、8 KB、16 KB 或 32 KB 之一，选择的页大小越大，则 DM 支持的元组长度也越大，但同时空间利用率可能下降，缺省使用 8 KB。
   - case_sensitive 标识符大小写敏感，默认值为 Y 。当大小写敏感时，小写的标识符应用双引号括起，否则被转换为大写；当大小写不敏感时，系统不自动转换标识符的大小写，在标识符比较时也不区分大小写，只能是 Y、y、N、n、1、0 之一。
   - charset 字符集选项。0 代表 GB18030；1 代表 UTF-8；2 代表韩文字符集 EUC-KR；取值 0、1 或 2 之一。默认值为 0。

可以使用默认参数初始化实例，需要附加实例存放路径。此处以初始化实例到 /dm8/data 目录下为例，初始化命令如下：

```bash
./dminit path=/dm8/data
```

### 1.4 注册服务

注册服务需使用 root 用户进行注册。使用 root 用户进入数据库安装目录的 /dm8/script/root 下，如下所示：

```bash
cd /dm8/script/root
./dm_service_installer.sh -t dmserver -dm_ini /dm8/data/DAMENG/dm.ini -p DMSERVER
```

用户可根据自己的环境更改 dm.ini 文件的路径以及服务名，如下所示：

```bash
./dm_service_installer.sh -h
```

### 1.5 启停服务

```bash
systemctl start DmServiceDMSERVER.service
systemctl stop DmServiceDMSERVER.service
systemctl restart DmServiceDMSERVER.service
systemctl status DmServiceDMSERVER.service
```

### 1.6 客户端测试

SYSDBA/SYSDBA

## 2. 库操作

### 2.1 用户管理

```bash
create user xzh identified by 123456789 limit password_life_time 60, failed_login_attemps 5, password_lock_time 5;
drop user xzh cascade;
create user xzh identified by "123456789" default tablespace tbs1 temporary tablespace temp_tbs1;
grant resource,public,soi,svi,vti to xzh;
```


### 2.2 表空间管理

```bash
create tablespace tbs1 datafile '/opt/dmdbms/data/prod/tbs1_01.dbf' size 128 autoextend on next 4 maxsize 2048; # 初始大小128m，每次自动扩充4m，最大尺寸2g
drop tablespace tbs1; 
alter tablespace tbs1 offline|online; # 表空间脱机
select tablespace_name from dba_tablespaces;
select tablespace_name,file_name from dba_data_files;
```    

### 2.3 备份恢复

### 2.4 数据迁移

## 3. 表操作

## 4. PLSQL

