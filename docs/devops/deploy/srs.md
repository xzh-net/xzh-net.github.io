# Simple RTMP Server 4.0

SRS是一个采用MIT协议授权的国产简单RTMP/HLS直播服务器。最新版还支持FLV模式，同时具备了RTMP的实时性，以及HLS中属于HTTP协议对各种网络环境高度适应性，并且支持更多播放器。它的功能与nginx-rtmp-module类似, 可以实现RTMP/HLS的分发,可用于直播/录播/视频客服等多种场景，其定位是运营级的互联网直播服务器集群

## 1. 安装

https://codeload.github.com/ossrs/srs/tar.gz/refs/tags/v4.0.187

```bash
cd /home
tar -xvf srs-4.0.187.tar.gz     # 解压
cd /home/srs-4.0.187/trunk      # 切换目录
./configure
make

./objs/srs -c conf/srs.conf     # 启动
./etc/init.d/srs status         # 查看状态
tail -n 30 -f ./objs/srs.log    # 查看日志
```

控制台网址 http://127.0.0.1:1985/console/ng_index.html


## 2. 配置

vi http.hls.conf

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

## 3. 测试

### 3.1 推流

- FFmpeg推流，使用rtmp协议时，默认使用1935端口 http://ffmpeg.org/
```bash
ffmpeg -re -i time.flv -vcodec copy -acodec copy -f flv -y rtmp://127.0.0.1/live/time
```

- OBS推流 https://obsproject.com/

### 3.2 拉流

- rtmp

```
rtmp://127.0.0.1/live/livestream
rtmp://127.0.0.1:8080/live/livestream.flv
rtmp://127.0.0.1:8080/live/livestream.m3u8

RTMP流地址：rtmp://127.0.0.1/live/time                # 端口号1935
HTTP FLV地址：http://127.0.0.1:8080/live/time.flv     # 端口号8080
HLS流地址：http://127.0.0.1:8080/live/time.m3u8       # 端口号8080
WebRTC流地址: webrtc://127.0.0.1/live/time            # 没有端口号

webrtc使用的是udp,默认监听8000,不需要设置端口号
```

- 在线播放器

http://127.0.0.1:8080/players/srs_player.html
