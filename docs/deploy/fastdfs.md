# FastDFS 5.11

FastDFS 是一款轻量级的开源分布式文件系统，功能包括：文件存储、文件同步、文件上传、文件下载等，解决了文件大容量存储和高性能访问问题。特别适合以文件为载体的在线服务，如图片、视频、文档服务等等

## 1. 单机

### 1.1 安装依赖

```bash
yum -y install gcc-c++ libevent
```

### 1.2 安装libfastcommon

```bash
mkdir -p /opt/software
cd /opt/software
wget https://github.com/happyfish100/libfastcommon/archive/V1.0.38.tar.gz 
tar -zxvf libfastcommon-1.0.38.tar.gz
cd libfastcommon-1.0.38
./make.sh
./make.sh install
```

FastDFS主程序设置的lib目录是/usr/local/lib，所以此处需要重新设置软链接

```bash
ln -s /usr/lib64/libfastcommon.so /usr/local/lib/libfastcommon.so
ln -s /usr/lib64/libfastcommon.so /usr/lib/libfastcommon.so
ln -s /usr/lib64/libfdfsclient.so /usr/local/lib/libfdfsclient.so
ln -s /usr/lib64/libfdfsclient.so /usr/lib/libfdfsclient.so
```

安装完成后 /usr/lib64、/usr/lib 文件夹内会有 libfastcommon.so 和 libfdfsclient.so

```bash
ll /usr/lib | grep libfastcommon.so 
ll /usr/lib | grep libfdfsclient.so 
```

### 1.3 安装FastDFS

```bash
cd /opt/software
wget https://github.com/happyfish100/fastdfs/archive/V5.11.tar.gz
tar -zxvf fastdfs-5.11.tar.gz
cd fastdfs-5.11
./make.sh
./make.sh install
```

安装完成后/etc/fdfs文件夹内会有client.conf.sample、storage.conf.sample、storage_ids.conf.sample、tracker.conf.sample文件

```bash
#将示例配置文件拷贝一份
cd /etc/fdfs/
cp client.conf.sample client.conf
cp storage.conf.sample storage.conf
cp tracker.conf.sample tracker.conf
```

### 1.4 启动FastDFS

#### 1.4.1 Tracker配置

1. 创建存储日志和数据的根目录

```bash
mkdir -p /data/fastdfs/tracker
```

2. 修改配置

```bash
vi /etc/fdfs/tracker.conf
```

```conf
# false是生效，true是屏蔽
disabled=false
bind_addr=
port=22122
connect_timeout=30
# tracker在通过网络发送接收数据的超时时间
network_timeout=60
# 数据和日志的存放地点
base_path=/data/fastdfs/tracker
# 服务所支持的最大链接数
max_connections=1024
accept_threads=1
# 工作线程数一般为cpu个数
work_threads=20
min_buff_size = 8KB
max_buff_size = 128KB
# 在存储文件时选择group的策略，0:轮训策略 1:指定某一个组 2:负载均衡，选择空闲空间最大的group
store_lookup=2
# 如果上面的store_lookup选择了1，则这里需要指定一个group                    
store_group=group1
# 在group中的哪台storage做主storage，当一个文件上传后由这台机器同步，0：轮训策略 1：根据ip地址排序，第一个 2:根据优先级排序，第一个
store_server=0
# 选择文件上传到storage中的哪个(目录/挂载点),storage可以有多个存放文件的base path 0:轮训策略 2:负载均衡，选择空闲空间最大的             
store_path=0
# 选择那个storage作为主下载服务器，0:轮训策略 1:主上传storage作为主下载服务器       
download_server=0
# 系统预留空间，当一个group中的任何storage的剩余空间小于定义的值，整个group就不能上传文件了         
reserved_storage_space = 10%
log_level=info
run_by_group=
run_by_user=
# 允许那些机器连接tracker默认是所有机器
allow_hosts=*
# 设置日志信息刷新到disk的频率，默认10s
sync_log_buff_interval = 10
# 检测storage服务器的间隔时间，storage定期主动向tracker发送心跳，如果在指定的时间没收到信号，tracker人为storage故障，默认120s          
check_active_interval = 120
# 线程栈的大小，最小64K
thread_stack_size = 64KB
# storage的ip改变后服务端是否自动调整，storage进程重启时才自动调整              
storage_ip_changed_auto_adjust = true
# storage之间同步文件的最大延迟，默认1天
storage_sync_file_max_delay = 86400
# 同步一个文件所花费的最大时间 
storage_sync_file_max_time = 300
# 是否用一个trunk文件存储多个小文件     
use_trunk_file = false
# 最小的solt大小，应该小于4KB，默认256bytes
slot_min_size = 256
# 最大的solt大小，如果上传的文件小于默认值，则上传文件被放入trunk文件中
slot_max_size = 16MB
# trunk文件的默认大小，应该大于4M 
trunk_file_size = 64MB
trunk_create_file_advance = false
trunk_create_file_time_base = 02:00
trunk_create_file_interval = 86400
trunk_create_file_space_threshold = 20G
trunk_init_check_occupying = false
trunk_init_reload_from_binlog = false
trunk_compress_binlog_min_interval = 0
use_storage_id = false
storage_ids_filename = storage_ids.conf
id_type_in_filename = ip
store_slave_file_use_link = false
rotate_error_log = false
error_log_rotate_time=00:00
rotate_error_log_size = 0
log_file_keep_days = 0
use_connection_pool = true
connection_pool_max_idle_time = 3600
# http服务端口
http.server_port=8080
# 检测storage上http服务的时间间隔，<=0表示不检测
http.check_alive_interval=30
# 检测storage上http服务时所用请求的类型，tcp只检测是否可以连接，http必须返回200
http.check_alive_type=tcp
# 通过url检测storage http服务状态
http.check_alive_uri=/status.html
```

3. 启动Tracker

```bash
/usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf
tail -f /data/fastdfs/tracker/logs/trackerd.log
```

4. 设置开机启动

```bash
chkconfig fdfs_trackerd on
systemctl enable fdfs_trackerd
```

#### 1.4.2 Storage配置

1. 创建存储日志和数据的根目录

```bash
mkdir -p /data/fastdfs/storage/base
mkdir -p /data/fastdfs/storage/store

```

2. 修改配置

```bash
vi /etc/fdfs/storage.conf
```

```conf
disabled=false
# 这个storage服务器属于那个group
group_name=group1
bind_addr=
client_bind=true
port=23000
connect_timeout=30
network_timeout=60
# 主动向tracker发送心跳检测的时间间隔
heart_beat_interval=30
# 主动向tracker发送磁盘使用率的时间间隔
stat_report_interval=60
# 数据日志的存放地点
base_path=/data/fastdfs/storage/base
# 服务所支持的最大链接数 
max_connections=1024
buff_size = 256KB
accept_threads=1
# 工作线程数一般为cpu个数
work_threads=20
# 磁盘IO是否读写分离
disk_rw_separated = true
# 混合读写时的读写线程数
disk_reader_threads = 5
# 混合读写时的读写线程数
disk_writer_threads = 5
# 同步文件时如果binlog没有要同步的文件，则延迟多少毫秒后重新读取，0表示不延迟
sync_wait_msec=50
# 同步完一个文件后间隔多少毫秒同步下一个文件，0表示不休息直接同步
sync_interval=0
# 表示这段时间内同步文件
sync_start_time=00:00
# 表示这段时间内同步文件
sync_end_time=23:59
# 同步完多少文件后写mark标记
write_mark_file_freq=500
# storage在存储文件时支持多路径，默认只设置一个
store_path_count=1
# 数据存储目录
store_path0=/data/fastdfs/storage/store
# subdir_count * subdir_count个目录会在store_path下创建，采用两级存储
subdir_count_per_path=256
# 设置tracker_server
tracker_server=192.168.2.201:22122
log_level=info
run_by_group=
run_by_user=
allow_hosts=*
# 文件在数据目录下的存放策略，0:轮训 1:随机
file_distribute_path_mode=0
# 当问及是轮训存放时，一个目录下可存放的文件数目
file_distribute_rotate_count=100
# 写入多少字节后就开始同步，0表示不同步
fsync_after_written_bytes=0
# 刷新日志信息到disk的间隔
sync_log_buff_interval=10
sync_binlog_buff_interval=10
# 同步storage的状态信息到disk的间隔
sync_stat_file_interval=300
# 线程栈大小
thread_stack_size=512KB
# 设置文件上传服务器的优先级，值越小越高
upload_priority=10
# 是否检测文件重复存在，1:检测 0:不检测
if_alias_prefix=
# 当check_file_duplicate设置为1时，次值必须设置
check_file_duplicate=0
file_signature_method=hash
key_namespace=FastDFS
# 与FastDHT建立连接的方式 0:短连接 1:长连接
keep_alive=0
use_access_log = false
rotate_access_log = false
access_log_rotate_time=00:00
rotate_error_log = false
error_log_rotate_time=00:00
rotate_access_log_size = 0
rotate_error_log_size = 0
log_file_keep_days = 0
file_sync_skip_invalid_record=false
use_connection_pool = true
connection_pool_max_idle_time = 3600
http.domain_name=
http.server_port=8888
```

3. 启动Storage

```bash
/usr/bin/fdfs_storaged /etc/fdfs/storage.conf
tail -f /data/fastdfs/storage/base/logs/storaged.log
```

4. 设置开机启动

```bash
chkconfig fdfs_storaged on
systemctl enable fdfs_storaged
```

#### 1.4.3 Client配置

1. 修改配置

```bash
vim /etc/fdfs/client.conf
```

```conf
connect_timeout=30
network_timeout=60
base_path=/data/fastdfs/tracker
tracker_server=192.168.2.201:22122
log_level=info
use_connection_pool = false
connection_pool_max_idle_time = 3600
load_fdfs_parameters_from_tracker=false
use_storage_id = false
storage_ids_filename = storage_ids.conf
http.tracker_server_port=8080
```

2. 上传

```bash
fdfs_upload_file /etc/fdfs/client.conf /opt/M001.jpg
```

3. 删除

```bash
fdfs_delete_file /etc/fdfs/client.conf group1/M00/00/00/rBERoWWJQ0GAUTbuAAB79bkxfWY442.jpg
```

4. 查看集群状态

```bash
fdfs_monitor /etc/fdfs/client.conf
```

### 1.6 配置插件

1. 下载解压

```bash
cd /opt/software
wget  https://github.com/happyfish100/fastdfs-nginx-module/archive/V1.20.tar.gz
tar -zxvf fastdfs-nginx-module-1.20.tar.gz
```

2. 配置config

```bash
cd fastdfs-nginx-module-1.20/src
vi config
```

修改以下两个参数

```conf
ngx_module_incs="/usr/include/fastdfs /usr/include/fastcommon/"
CORE_INCS="$CORE_INCS /usr/include/fastdfs /usr/include/fastcommon/"
```

3. 配置mod_fastdfs.conf

```bash
cp mod_fastdfs.conf /etc/fdfs/
vi /etc/fdfs/mod_fastdfs.conf
```

```conf
connect_timeout=30
network_timeout=30
base_path=/data/fastdfs
load_fdfs_parameters_from_tracker=true
storage_sync_file_max_delay = 86400
use_storage_id = false
storage_ids_filename = storage_ids.conf
tracker_server=192.168.2.201:22122
storage_server_port=23000
group_name=group1
url_have_group_name = true
store_path_count=1
store_path0=/data/fastdfs/storage/store
log_level=info
log_filename=
response_mode=proxy
if_alias_prefix=
flv_support = true
flv_extension = flv
group_count = 0
#include http.conf
```

4. 复制映射文件

```bash
cp /opt/software/fastdfs-5.11/conf/mime.types /etc/fdfs/
cp /opt/software/fastdfs-5.11/conf/http.conf /etc/fdfs/
```

### 1.7 安装Nginx

1. 安装依赖

```bash
yum install -y gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel
```

2. 解压编译

```bash
cd /opt/software
wget http://nginx.org/download/nginx-1.20.2.tar.gz
tar zxvf nginx-1.20.2.tar.gz -C /opt/software
cd nginx-1.20.2
./configure  --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module --add-module=/opt/software/fastdfs-nginx-module-1.20/src
make && make install
```

3. 修改配置


```bash
vi /usr/local/nginx/conf/nginx.conf
```

```conf
server {
        listen 8888;
        server_name localhost;
        location ~/group[0-9]/ {
            ngx_fastdfs_module;
        }
    }
```

4. 启动nginx

```bash
/usr/local/nginx/sbin/nginx -t
/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
```

访问地址：http://192.168.2.201/group1/M00/00/00/rBERoWWJQ0GAUTbuAAB79bkxfWY442.jpg

## 2. 集群


| **节点名称** | **IP地址** | **服务名称** | **端口** |
| :----------: | :----------: | :----------: | :----------: |
| node10  | node10 192.168.2.201 | libfastcommon | - |
|   |  | tracker | 22122 |
|   |  | storage-group1 | 23000 |
|   |  | storage-group2 | 33000 |
|   |  | fastdfs-nginx-module | - |
|   |  | nginx | 8888 |
| node20  | node20 192.168.2.202 | libfastcommon | - |
|   |  | tracker | 22122 |
|   |  | storage-group1 | 23000 |
|   |  | storage-group2 | 33000 |
|   |  | fastdfs-nginx-module | - |
|   |  | nginx | 8888 |


### 2.1 节点1

#### 2.1.1 Tracker配置

1. 创建存储日志和数据的根目录

```bash
mkdir -p /data/fastdfs/tracker
```

2. 修改配置

```bash
vi /etc/fdfs/tracker.conf
```

```conf
disabled=false
bind_addr=
port=22122
connect_timeout=30
network_timeout=60
base_path=/data/fastdfs/tracker
max_connections=1024
accept_threads=1
work_threads=20
min_buff_size = 8KB
max_buff_size = 128KB
store_lookup=2
store_group=group1
store_server=0
store_path=0
download_server=0
reserved_storage_space = 10%
log_level=info
run_by_group=
run_by_user=
allow_hosts=*
sync_log_buff_interval = 10
check_active_interval = 120
thread_stack_size = 64KB
storage_ip_changed_auto_adjust = true
storage_sync_file_max_delay = 86400
storage_sync_file_max_time = 300
use_trunk_file = false 
slot_min_size = 256
slot_max_size = 16MB
trunk_file_size = 64MB
trunk_create_file_advance = false
trunk_create_file_time_base = 02:00
trunk_create_file_interval = 86400
trunk_create_file_space_threshold = 20G
trunk_init_check_occupying = false
trunk_init_reload_from_binlog = false
trunk_compress_binlog_min_interval = 0
use_storage_id = false
storage_ids_filename = storage_ids.conf
id_type_in_filename = ip
store_slave_file_use_link = false
rotate_error_log = false
error_log_rotate_time=00:00
rotate_error_log_size = 0
log_file_keep_days = 0
use_connection_pool = true
connection_pool_max_idle_time = 3600
http.server_port=8080
http.check_alive_interval=30
http.check_alive_type=tcp
http.check_alive_uri=/status.html
```

3. 启动Tracker

```bash
/usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf
tail -f /data/fastdfs/tracker/logs/trackerd.log
ps aux | grep fdfs
```


#### 2.1.2 Storage配置

1. 创建存储日志和数据的根目录

```bash
mkdir -p /data/fastdfs/storage/group1/base
mkdir -p /data/fastdfs/storage/group1/data01
mkdir -p /data/fastdfs/storage/group1/data02
mkdir -p /data/fastdfs/storage/group1/data03
mkdir -p /data/fastdfs/storage/group1/data04
mkdir -p /data/fastdfs/storage/group1/data05
mkdir -p /data/fastdfs/storage/group1/data06

mkdir -p /data/fastdfs/storage/group2/base
mkdir -p /data/fastdfs/storage/group2/data01
mkdir -p /data/fastdfs/storage/group2/data02
mkdir -p /data/fastdfs/storage/group2/data03
mkdir -p /data/fastdfs/storage/group2/data04
mkdir -p /data/fastdfs/storage/group2/data05
mkdir -p /data/fastdfs/storage/group2/data06
```

2. 新建配置group1

```bash
vi /etc/fdfs/storage-group1.conf
```

```conf
disabled=false
group_name=group1
bind_addr=
client_bind=true
port=23000
connect_timeout=30
network_timeout=60
heart_beat_interval=30
stat_report_interval=60
base_path=/data/fastdfs/storage/group1/base
max_connections=1024
buff_size = 256KB
accept_threads=1
work_threads=20
disk_rw_separated = true
disk_reader_threads = 5
disk_writer_threads = 5
sync_wait_msec=50
sync_interval=0
sync_start_time=00:00
sync_end_time=23:59
write_mark_file_freq=500
store_path_count=6
store_path0=/data/fastdfs/storage/group1/data01
store_path1=/data/fastdfs/storage/group1/data02
store_path2=/data/fastdfs/storage/group1/data03
store_path3=/data/fastdfs/storage/group1/data04
store_path4=/data/fastdfs/storage/group1/data05
store_path5=/data/fastdfs/storage/group1/data06
subdir_count_per_path=256
tracker_server=192.168.2.201:22122
tracker_server=192.168.2.202:22122
log_level=info
run_by_group=
run_by_user=
allow_hosts=*
file_distribute_path_mode=0
file_distribute_rotate_count=100
fsync_after_written_bytes=0
sync_log_buff_interval=10
sync_binlog_buff_interval=10
sync_stat_file_interval=300
thread_stack_size=512KB
upload_priority=10
if_alias_prefix=
check_file_duplicate=0
file_signature_method=hash
key_namespace=FastDFS
keep_alive=0
use_access_log = false
rotate_access_log = false
access_log_rotate_time=00:00
rotate_error_log = false
error_log_rotate_time=00:00
rotate_access_log_size = 0
rotate_error_log_size = 0
log_file_keep_days = 0
file_sync_skip_invalid_record=false
use_connection_pool = true
connection_pool_max_idle_time = 3600
http.domain_name=
http.server_port=8888
```

3. 新建配置group2

```bash
vi /etc/fdfs/storage-group2.conf
```

```conf
disabled=false
group_name=group2
bind_addr=
client_bind=true
port=33000
connect_timeout=30
network_timeout=60
heart_beat_interval=30
stat_report_interval=60
base_path=/data/fastdfs/storage/group2/base
max_connections=1024
buff_size = 256KB
accept_threads=1
work_threads=20
disk_rw_separated = true
disk_reader_threads = 5
disk_writer_threads = 5
sync_wait_msec=50
sync_interval=0
sync_start_time=00:00
sync_end_time=23:59
write_mark_file_freq=500
store_path_count=6
store_path0=/data/fastdfs/storage/group2/data01
store_path1=/data/fastdfs/storage/group2/data02
store_path2=/data/fastdfs/storage/group2/data03
store_path3=/data/fastdfs/storage/group2/data04
store_path4=/data/fastdfs/storage/group2/data05
store_path5=/data/fastdfs/storage/group2/data06
subdir_count_per_path=256
tracker_server=192.168.2.201:22122
tracker_server=192.168.2.202:22122
log_level=info
run_by_group=
run_by_user=
allow_hosts=*
file_distribute_path_mode=0
file_distribute_rotate_count=100
fsync_after_written_bytes=0
sync_log_buff_interval=10
sync_binlog_buff_interval=10
sync_stat_file_interval=300
thread_stack_size=512KB
upload_priority=10
if_alias_prefix=
check_file_duplicate=0
file_signature_method=hash
key_namespace=FastDFS
keep_alive=0
use_access_log = false
rotate_access_log = false
access_log_rotate_time=00:00
rotate_error_log = false
error_log_rotate_time=00:00
rotate_access_log_size = 0
rotate_error_log_size = 0
log_file_keep_days = 0
file_sync_skip_invalid_record=false
use_connection_pool = true
connection_pool_max_idle_time = 3600
http.domain_name=
http.server_port=8888
```

4. 启动Storage

```bash
/usr/bin/fdfs_storaged /etc/fdfs/storage-group1.conf
tail -f /data/fastdfs/storage/group1/base/logs/storaged.log

/usr/bin/fdfs_storaged /etc/fdfs/storage-group2.conf
tail -f /data/fastdfs/storage/group2/base/logs/storaged.log
ps aux | grep fdfs
```


#### 2.1.3 Client配置

1. 修改配置

```bash
vim /etc/fdfs/client.conf
```

```conf
connect_timeout=30
network_timeout=60
base_path=/data/fastdfs/tracker
tracker_server=192.168.2.201:22122
tracker_server=192.168.2.202:22122
log_level=info
use_connection_pool = false
connection_pool_max_idle_time = 3600
load_fdfs_parameters_from_tracker=false
use_storage_id = false
storage_ids_filename = storage_ids.conf
http.tracker_server_port=8080
```

2. 上传

```bash
fdfs_upload_file /etc/fdfs/client.conf /opt/M001.jpg
```

#### 2.1.4 配置插件

1. 下载解压

```bash
cd /opt/software
wget  https://github.com/happyfish100/fastdfs-nginx-module/archive/V1.20.tar.gz
tar -zxvf fastdfs-nginx-module-1.20.tar.gz
```

2. 配置config

```bash
cd fastdfs-nginx-module-1.20/src
vi config
```

修改以下两个参数

```conf
ngx_module_incs="/usr/include/fastdfs /usr/include/fastcommon/"
CORE_INCS="$CORE_INCS /usr/include/fastdfs /usr/include/fastcommon/"
```

3. 配置mod_fastdfs.conf

```bash
cp mod_fastdfs.conf /etc/fdfs/
vi /etc/fdfs/mod_fastdfs.conf
```

```conf
connect_timeout=30
network_timeout=30
base_path=/data/fastdfs
load_fdfs_parameters_from_tracker=true
storage_sync_file_max_delay = 86400
use_storage_id = false
storage_ids_filename = storage_ids.conf
tracker_server=192.168.2.201:22122
tracker_server=192.168.2.202:22122
group_name=group1/group2
storage_server_port=23000
url_have_group_name = true
store_path_count=1
store_path0=/data/fastdfs/storage/store
log_level=info
log_filename=
response_mode=proxy
if_alias_prefix=
flv_support = true
flv_extension = flv
group_count = 2
[group1]
group_name=group1
storage_server_port=23000
store_path_count=6
store_path0=/data/fastdfs/storage/group1/data01
store_path1=/data/fastdfs/storage/group1/data02
store_path2=/data/fastdfs/storage/group1/data03
store_path3=/data/fastdfs/storage/group1/data04
store_path4=/data/fastdfs/storage/group1/data05
store_path5=/data/fastdfs/storage/group1/data06
[group2]
group_name=group2
storage_server_port=33000
store_path_count=6
store_path0=/data/fastdfs/storage/group2/data01
store_path1=/data/fastdfs/storage/group2/data02
store_path2=/data/fastdfs/storage/group2/data03
store_path3=/data/fastdfs/storage/group2/data04
store_path4=/data/fastdfs/storage/group2/data05
store_path5=/data/fastdfs/storage/group2/data06
#include http.conf
```

4. 复制映射文件

```bash
cp /opt/software/fastdfs-5.11/conf/mime.types /etc/fdfs/
cp /opt/software/fastdfs-5.11/conf/http.conf /etc/fdfs/
```

#### 2.1.5 安装Nginx

1. 安装依赖

```bash
yum install -y gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel
```

2. 解压编译

```bash
cd /opt/software
wget http://nginx.org/download/nginx-1.20.2.tar.gz
tar zxvf nginx-1.20.2.tar.gz -C /opt/software
cd nginx-1.20.2
./configure  --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module --add-module=/opt/software/fastdfs-nginx-module-1.20/src
make && make install
```

3. 修改配置


```bash
vi /usr/local/nginx/conf/nginx.conf
```

```conf
server {
        listen 8888;
        server_name localhost;
        location ~/group[0-9]/ {
            ngx_fastdfs_module;
        }
    }
```

4. 启动nginx

```bash
/usr/local/nginx/sbin/nginx -t
/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
```

访问地址：http://192.168.2.201/group1/M00/00/00/rBERoWWJQ0GAUTbuAAB79bkxfWY442.jpg


### 2.2 节点2

#### 2.2.1 Tracker配置

1. 创建存储日志和数据的根目录

```bash
mkdir -p /data/fastdfs/tracker
```

2. 修改配置

```bash
vi /etc/fdfs/tracker.conf
```

```conf
disabled=false
bind_addr=
port=22122
connect_timeout=30
network_timeout=60
base_path=/data/fastdfs/tracker
max_connections=1024
accept_threads=1
work_threads=20
min_buff_size = 8KB
max_buff_size = 128KB
store_lookup=2
store_group=group1
store_server=0
store_path=0
download_server=0
reserved_storage_space = 10%
log_level=info
run_by_group=
run_by_user=
allow_hosts=*
sync_log_buff_interval = 10
check_active_interval = 120
thread_stack_size = 64KB
storage_ip_changed_auto_adjust = true
storage_sync_file_max_delay = 86400
storage_sync_file_max_time = 300
use_trunk_file = false 
slot_min_size = 256
slot_max_size = 16MB
trunk_file_size = 64MB
trunk_create_file_advance = false
trunk_create_file_time_base = 02:00
trunk_create_file_interval = 86400
trunk_create_file_space_threshold = 20G
trunk_init_check_occupying = false
trunk_init_reload_from_binlog = false
trunk_compress_binlog_min_interval = 0
use_storage_id = false
storage_ids_filename = storage_ids.conf
id_type_in_filename = ip
store_slave_file_use_link = false
rotate_error_log = false
error_log_rotate_time=00:00
rotate_error_log_size = 0
log_file_keep_days = 0
use_connection_pool = true
connection_pool_max_idle_time = 3600
http.server_port=8080
http.check_alive_interval=30
http.check_alive_type=tcp
http.check_alive_uri=/status.html
```

3. 启动Tracker

```bash
/usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf
tail -f /data/fastdfs/tracker/logs/trackerd.log
ps aux | grep fdfs
```


#### 2.2.2 Storage配置

1. 创建存储日志和数据的根目录

```bash
mkdir -p /data/fastdfs/storage/group1/base
mkdir -p /data/fastdfs/storage/group1/data01
mkdir -p /data/fastdfs/storage/group1/data02
mkdir -p /data/fastdfs/storage/group1/data03
mkdir -p /data/fastdfs/storage/group1/data04
mkdir -p /data/fastdfs/storage/group1/data05
mkdir -p /data/fastdfs/storage/group1/data06

mkdir -p /data/fastdfs/storage/group2/base
mkdir -p /data/fastdfs/storage/group2/data01
mkdir -p /data/fastdfs/storage/group2/data02
mkdir -p /data/fastdfs/storage/group2/data03
mkdir -p /data/fastdfs/storage/group2/data04
mkdir -p /data/fastdfs/storage/group2/data05
mkdir -p /data/fastdfs/storage/group2/data06
```

2. 新建配置group1

```bash
vi /etc/fdfs/storage-group1.conf
```

```conf
disabled=false
group_name=group1
bind_addr=
client_bind=true
port=23000
connect_timeout=30
network_timeout=60
heart_beat_interval=30
stat_report_interval=60
base_path=/data/fastdfs/storage/group1/base
max_connections=1024
buff_size = 256KB
accept_threads=1
work_threads=20
disk_rw_separated = true
disk_reader_threads = 5
disk_writer_threads = 5
sync_wait_msec=50
sync_interval=0
sync_start_time=00:00
sync_end_time=23:59
write_mark_file_freq=500
store_path_count=6
store_path0=/data/fastdfs/storage/group1/data01
store_path1=/data/fastdfs/storage/group1/data02
store_path2=/data/fastdfs/storage/group1/data03
store_path3=/data/fastdfs/storage/group1/data04
store_path4=/data/fastdfs/storage/group1/data05
store_path5=/data/fastdfs/storage/group1/data06
subdir_count_per_path=256
tracker_server=192.168.2.201:22122
tracker_server=192.168.2.202:22122
log_level=info
run_by_group=
run_by_user=
allow_hosts=*
file_distribute_path_mode=0
file_distribute_rotate_count=100
fsync_after_written_bytes=0
sync_log_buff_interval=10
sync_binlog_buff_interval=10
sync_stat_file_interval=300
thread_stack_size=512KB
upload_priority=10
if_alias_prefix=
check_file_duplicate=0
file_signature_method=hash
key_namespace=FastDFS
keep_alive=0
use_access_log = false
rotate_access_log = false
access_log_rotate_time=00:00
rotate_error_log = false
error_log_rotate_time=00:00
rotate_access_log_size = 0
rotate_error_log_size = 0
log_file_keep_days = 0
file_sync_skip_invalid_record=false
use_connection_pool = true
connection_pool_max_idle_time = 3600
http.domain_name=
http.server_port=8888
```

3. 新建配置group2

```bash
vi /etc/fdfs/storage-group2.conf
```

```conf
disabled=false
group_name=group2
bind_addr=
client_bind=true
port=33000
connect_timeout=30
network_timeout=60
heart_beat_interval=30
stat_report_interval=60
base_path=/data/fastdfs/storage/group2/base
max_connections=1024
buff_size = 256KB
accept_threads=1
work_threads=20
disk_rw_separated = true
disk_reader_threads = 5
disk_writer_threads = 5
sync_wait_msec=50
sync_interval=0
sync_start_time=00:00
sync_end_time=23:59
write_mark_file_freq=500
store_path_count=6
store_path0=/data/fastdfs/storage/group2/data01
store_path1=/data/fastdfs/storage/group2/data02
store_path2=/data/fastdfs/storage/group2/data03
store_path3=/data/fastdfs/storage/group2/data04
store_path4=/data/fastdfs/storage/group2/data05
store_path5=/data/fastdfs/storage/group2/data06
subdir_count_per_path=256
tracker_server=192.168.2.201:22122
tracker_server=192.168.2.202:22122
log_level=info
run_by_group=
run_by_user=
allow_hosts=*
file_distribute_path_mode=0
file_distribute_rotate_count=100
fsync_after_written_bytes=0
sync_log_buff_interval=10
sync_binlog_buff_interval=10
sync_stat_file_interval=300
thread_stack_size=512KB
upload_priority=10
if_alias_prefix=
check_file_duplicate=0
file_signature_method=hash
key_namespace=FastDFS
keep_alive=0
use_access_log = false
rotate_access_log = false
access_log_rotate_time=00:00
rotate_error_log = false
error_log_rotate_time=00:00
rotate_access_log_size = 0
rotate_error_log_size = 0
log_file_keep_days = 0
file_sync_skip_invalid_record=false
use_connection_pool = true
connection_pool_max_idle_time = 3600
http.domain_name=
http.server_port=8888
```

4. 启动Storage

```bash
/usr/bin/fdfs_storaged /etc/fdfs/storage-group1.conf
tail -f /data/fastdfs/storage/group1/base/logs/storaged.log

/usr/bin/fdfs_storaged /etc/fdfs/storage-group2.conf
tail -f /data/fastdfs/storage/group2/base/logs/storaged.log
ps aux | grep fdfs
```


#### 2.2.3 Client配置

1. 修改配置

```bash
vim /etc/fdfs/client.conf
```

```conf
connect_timeout=30
network_timeout=60
base_path=/data/fastdfs/tracker
tracker_server=192.168.2.201:22122
tracker_server=192.168.2.202:22122
log_level=info
use_connection_pool = false
connection_pool_max_idle_time = 3600
load_fdfs_parameters_from_tracker=false
use_storage_id = false
storage_ids_filename = storage_ids.conf
http.tracker_server_port=8080
```

2. 上传

```bash
fdfs_upload_file /etc/fdfs/client.conf /opt/M001.jpg
```

#### 2.3.4 配置插件

1. 下载解压

```bash
cd /opt/software
wget  https://github.com/happyfish100/fastdfs-nginx-module/archive/V1.20.tar.gz
tar -zxvf fastdfs-nginx-module-1.20.tar.gz
```

2. 配置config

```bash
cd fastdfs-nginx-module-1.20/src
vi config
```

修改以下两个参数

```conf
ngx_module_incs="/usr/include/fastdfs /usr/include/fastcommon/"
CORE_INCS="$CORE_INCS /usr/include/fastdfs /usr/include/fastcommon/"
```

3. 配置mod_fastdfs.conf

```bash
cp mod_fastdfs.conf /etc/fdfs/
vi /etc/fdfs/mod_fastdfs.conf
```

```conf
connect_timeout=30
network_timeout=30
base_path=/data/fastdfs
load_fdfs_parameters_from_tracker=true
storage_sync_file_max_delay = 86400
use_storage_id = false
storage_ids_filename = storage_ids.conf
tracker_server=192.168.2.201:22122
tracker_server=192.168.2.202:22122
group_name=group1/group2
storage_server_port=23000
url_have_group_name = true
store_path_count=1
store_path0=/data/fastdfs/storage/store
log_level=info
log_filename=
response_mode=proxy
if_alias_prefix=
flv_support = true
flv_extension = flv
group_count = 2
[group1]
group_name=group1
storage_server_port=23000
store_path_count=6
store_path0=/data/fastdfs/storage/group1/data01
store_path1=/data/fastdfs/storage/group1/data02
store_path2=/data/fastdfs/storage/group1/data03
store_path3=/data/fastdfs/storage/group1/data04
store_path4=/data/fastdfs/storage/group1/data05
store_path5=/data/fastdfs/storage/group1/data06
[group2]
group_name=group2
storage_server_port=33000
store_path_count=6
store_path0=/data/fastdfs/storage/group2/data01
store_path1=/data/fastdfs/storage/group2/data02
store_path2=/data/fastdfs/storage/group2/data03
store_path3=/data/fastdfs/storage/group2/data04
store_path4=/data/fastdfs/storage/group2/data05
store_path5=/data/fastdfs/storage/group2/data06
#include http.conf
```

4. 复制映射文件

```bash
cp /opt/software/fastdfs-5.11/conf/mime.types /etc/fdfs/
cp /opt/software/fastdfs-5.11/conf/http.conf /etc/fdfs/
```

#### 2.3.5 安装Nginx

1. 安装依赖

```bash
yum install -y gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel
```

2. 解压编译

```bash
cd /opt/software
wget http://nginx.org/download/nginx-1.20.2.tar.gz
tar zxvf nginx-1.20.2.tar.gz -C /opt/software
cd nginx-1.20.2
./configure  --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module --add-module=/opt/software/fastdfs-nginx-module-1.20/src
make && make install
```

3. 修改配置


```bash
vi /usr/local/nginx/conf/nginx.conf
```

```conf
server {
        listen 8888;
        server_name localhost;
        location ~/group[0-9]/ {
            ngx_fastdfs_module;
        }
    }
```

4. 启动nginx

```bash
/usr/local/nginx/sbin/nginx -t
/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
```

访问地址：http://192.168.2.202/group1/M00/00/00/rBERoWWJQ0GAUTbuAAB79bkxfWY442.jpg
