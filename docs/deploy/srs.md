# Simple RTMP Server 4.0

SRS是一个采用MIT协议授权的国产简单RTMP/HLS直播服务器。最新版还支持FLV模式，同时具备了RTMP的实时性，以及HLS中属于HTTP协议对各种网络环境高度适应性，并且支持更多播放器。它的功能与nginx-rtmp-module类似, 可以实现RTMP/HLS的分发,可用于直播/录播/视频客服等多种场景，其定位是运营级的互联网直播服务器集群

- 下载地址：https://github.com/ossrs/srs/releases/download/v4.0-r5/srs-server-4.0-r5.tar.gz


## 1. 安装

### 1.1 解压编译

```bash
cd /opt/software
tar -zxvf srs-server-4.0-r5.tar.gz -C /opt
cd /opt/srs-server-4.0-r5/trunk
./configure
make
```

### 1.2 修改配置

```bash
vi /opt/srs-server-4.0-r5/trunk/conf/http.hls.conf
```

```conf
listen              1935;
max_connections     1000;
daemon              on;
srs_log_tank        file;
srs_log_level       error;
srs_log_file        ./objs/srs.log;

http_api {
    enabled         on;
    listen          1985;
}

http_server {
    enabled         on;
    listen          8080;
    dir             ./objs/nginx/html;
}

vhost __defaultVhost__ {
    hls {
        enabled         on;
        hls_fragment    10;
        hls_window      60;
        hls_path        ./objs/nginx/html;
        hls_m3u8_file   [app]/[stream].m3u8;
        hls_ts_file     [app]/[stream]-[seq].ts;
        hls_cleanup     on;
        hls_dispose     30;
        hls_on_error    continue;
        hls_storage     disk;
        hls_wait_keyframe       on;
        hls_acodec      aac;
        hls_vcodec      h264;
    }
}
```

### 1.3 启动服务

```bash
cd /opt/srs-server-4.0-r5/trunk
./objs/srs -c conf/srs.conf         # 启动
./etc/init.d/srs status             # 查看状态
tail -n 30 -f ./objs/srs.log        # 查看日志
```

访问网站：http://192.168.3.201:8080/


## 2. 客户端测试

- RTMP流地址：rtmp://192.168.3.201/live/time （端口号1935）
- FLV地址：http://192.168.3.201:8080/live/time.flv （端口号8080）
- HLS流地址：http://192.168.3.201:8080/live/time.m3u8 （端口号8080）
- WebRTC流地址: webrtc://192.168.3.201/live/time （webrtc使用的是udp,默认监听8000,不需要设置端口号）

### 2.1 FFmpeg

下载地址：http://ffmpeg.org 

FFmpeg推流，使用rtmp协议时，默认使用1935端口，这里使用windows10安装ffmpeg来测试。

#### 2.1.1 配置环境

path环境变量添加`D:\tools\ffmpeg\bin`

#### 2.1.2 推流

```bash
ffmpeg -re -i time.flv -vcodec copy -acodec copy -f flv -y rtmp://192.168.3.201/live/time
```

#### 2.1.3 拉流

- rtmp://192.168.3.201/live/time
- rtmp://192.168.3.201:8080/live/time.flv
- rtmp://192.168.3.201:8080/live/time.m3u8

```bash
ffplay rtmp://192.168.3.201/live/time
```

### 2.2 VLC

下载地址：https://get.videolan.org/vlc/3.0.12/win64/vlc-3.0.12-win64.exe

#### 2.2.1 拉流

媒体 -> 打开网络串流 -> 输入rtmp地址

### 2.3 OBS

下载地址：https://obsproject.com/

#### 2.3.1 推流

1. 设置捕获源

![](../../assets/_images/deploy/srs/1.png)
![](../../assets/_images/deploy/srs/2.png)
![](../../assets/_images/deploy/srs/3.png)
![](../../assets/_images/deploy/srs/4.png)

2. 设置视频

![](../../assets/_images/deploy/srs/5.png)
![](../../assets/_images/deploy/srs/6.png)


3. 设置音频

【注意】如果只想进行桌面共享，不想传输声音，则将方框中选项全部选择已禁用。如果想进行桌面共享及传输声音，则按照图示设置

![](../../assets/_images/deploy/srs/7.png)

4. 设置输出

![](../../assets/_images/deploy/srs/8.png)

5. 设置推流

![](../../assets/_images/deploy/srs/9.png)

6. 开始推流

![](../../assets/_images/deploy/srs/10.png)
![](../../assets/_images/deploy/srs/11.png)


### 2.4 在线播放器

http://192.168.3.201:8080/players/srs_player.html





