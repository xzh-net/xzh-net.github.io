# FastDFS 5.11

FastDFS 是一款轻量级的开源分布式文件系统，功能包括：文件存储、文件同步、文件上传、文件下载等，解决了文件大容量存储和高性能访问问题。特别适合以文件为载体的在线服务，如图片、视频、文档服务等等

## 1. 安装

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

安装完成后 /usr/lib64、/usr/lib 文件夹内会有 libfastcommon.so 和 libfdfsclient.so

```bash
ll /usr/lib | grep libfastcommon.so 
ll /usr/lib | grep libfdfsclient.so 
```

```bash
#V5.11可忽略
#FastDFS主程序设置的lib目录是/usr/local/lib，所以此处需要重新设置软链接
ln -s /usr/lib64/libfastcommon.so /usr/local/lib/libfastcommon.so
ln -s /usr/lib64/libfastcommon.so /usr/lib/libfastcommon.so
ln -s /usr/lib64/libfdfsclient.so /usr/local/lib/libfdfsclient.so
ln -s /usr/lib64/libfdfsclient.so /usr/lib/libfdfsclient.so
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

1. 创建Tracker的logs和data目录

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
```

4. 设置开机启动

```bash
chkconfig fdfs_trackerd on
systemctl enable fdfs_trackerd
```

#### 1.4.2 Storage配置

1. 创建Storage的logs和data目录

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
group_name=group1
bind_addr=
client_bind=true
port=23000
connect_timeout=30
network_timeout=60
heart_beat_interval=30
stat_report_interval=60
base_path=/data/fastdfs/storage/base
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
store_path_count=1
store_path0=/data/fastdfs/storage/store
subdir_count_per_path=256
tracker_server=192.168.2.3:22122
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

### 1.5 客户端测试

1. 修改配置

```bash
vim /etc/fdfs/client.conf
```

```conf
connect_timeout=30
network_timeout=60
base_path=/data/fastdfs/tracker
tracker_server=192.168.2.3:22122
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

### 1.6 配置fastdfs-nginx-module 

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
tracker_server=192.168.2.3:22122
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

4. 映射文件

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
location /group1/M00 {
   ngx_fastdfs_module;
}
```

4. 启动nginx

```bash
/usr/local/nginx/sbin/nginx -t
/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
```

访问地址：http://192.168.2.3/group1/M00/00/00/rBERoWWJQ0GAUTbuAAB79bkxfWY442.jpg

## 2. 备份

## 3. 高可用