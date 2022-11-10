# Prometheus 2.27.0 + grafana-7.3.7-1.x86_64

Prometheus是由SoundCloud开发的开源监控报警系统和时序列数据库(TSDB)。Prometheus使用Go语言开发，是Google BorgMon监控系统的开源版本

- Prometheus 下载地址：https://prometheus.io/download/
- Grafana 下载地址：https://grafana.com/grafana/download
- Grafana 控制台：https://grafana.com/grafana/dashboards?dataSource=prometheus
- node_exporter 下载地址：https://prometheus.io/download/#node_exporter


## 1. Prometheus

### 1.1 下载解压

```bash
cd /opt/software
wget https://github.com/prometheus/prometheus/releases/download/v2.27.0/prometheus-2.27.0.linux-amd64.tar.gz
tar zxvf prometheus-2.27.0.linux-amd64.tar.gz -C /opt
mv /opt/prometheus-2.27.0.linux-amd64/ /opt/prometheus
/opt/prometheus/prometheus --version  # 查看版本号
```

### 1.2 修改配置

```bash
vi /opt/prometheus/prometheus.yml
```

```conf
# 全局配置（如果有内部单独设定，会覆盖这个参数）
global:
  scrape_interval: 15s      # 默认15s 全局每次数据收集的间隔
  evaluation_interval: 15s  # 规则扫描时间间隔是15秒，默认不填写是 1分钟
  scrape_timeout: 5s        # 超时时间 默认10s

# 告警插件定义。这里会设定alertmanager这个报警插件
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# 告警规则。 按照设定参数进行扫描加载，用于自定义报警规则，其报警媒介和route路由由alertmanager插件实现
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# 采集配置。配置数据源，包含分组job_name以及具体target。又分为静态配置和服务发现
scrape_configs:
  - job_name: 'prometheus'
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
      - targets: ['localhost:9090']
        labels:
          instance: prometheus
  - job_name: linux
    static_configs:
      - targets: ['172.17.17.201:9100','172.17.17.200:9100']
        labels:
          instance: Linux
  - job_name: docker
    static_configs:
      - targets: ['172.17.17.80:18080']
        labels:
          instance: docker18
  - job_name: mysql
    static_configs:
      - targets: ['172.17.17.80:9104']
        labels:
          instance: mysql8
  - job_name: redis_192
    static_configs:
      - targets: ['172.17.17.192:9121']
        labels:
          instance: redis_192
```


### 1.3 添加系统服务

1. 创建启动文件

```bash
cd /usr/lib/systemd/system
vim prometheus.service
# 添加内容
[Unit]
Description=https://prometheus.io

[Service]
Restart=on-failure
ExecStart=/opt/prometheus/prometheus --config.file=/opt/prometheus/prometheus.yml --web.listen-address=:9090

[Install]
WantedBy=multi-user.target
```

2. 重载守护进程

```bash
systemctl daemon-reload
```

### 1.4 启动服务

```bash
systemctl start prometheus
systemctl enable prometheus

# 也可使用脚本启动
cd /opt/prometheus
./prometheus --config.file="/opt/prometheus/prometheus.yml" &
./prometheus --help       # 启动参数帮助文档
```

### 1.5 客户端

?> 访问地址：http://ip:9090/targets


## 2. node_exporter

### 2.1 linux监控


#### 2.1.1 下载解压

```bash
cd /opt/software
wget https://github.com/prometheus/node_exporter/releases/download/v1.1.2/node_exporter-1.1.2.linux-amd64.tar.gz
tar zxvf node_exporter-1.1.2.linux-amd64.tar.gz -C /usr/local/
mv /usr/local/node_exporter-1.1.2.linux-amd64/ /usr/local/node_exporter
```

#### 2.1.2 添加系统服务

1. 创建启动文件

```bash
vim /usr/lib/systemd/system/node_exporter.service
# 添加内容
[Unit]
Description=node_exporter
After=network.target

[Service]
ExecStart=/usr/local/node_exporter/node_exporter
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

2. 重载守护进程

```bash
systemctl daemon-reload
```

#### 2.1.3 启动服务

```bash
systemctl start node_exporter
# 或者使用脚本启动
nohup /usr/local/node_exporter/node_exporter  --web.listen-address=":9100" & # 后台启动
lsof -i:9100
```

### 2.2 mysql监控

#### 2.2.1 下载解压

```bash
cd /opt/software
wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.14.0/mysqld_exporter-0.14.0.linux-amd64.tar.gz
tar zxvf mysqld_exporter-0.14.0.linux-amd64.tar.gz -C /usr/local/
mv /usr/local/mysqld_exporter-0.14.0.linux-amd64 /usr/local/mysqld_exporter
vim /usr/local/mysqld_exporter/.my.cnf
# 编辑
[client]
host=172.17.17.137
port=3306
user=root
password=root
```

#### 2.2.2 添加系统服务

1. 创建启动文件

```bash
vim /usr/lib/systemd/system/mysqld_exporter.service
```

```bash
[Unit]
Description=mysqld_exporter
After=network.target

[Service]
ExecStart=/usr/local/mysqld_exporter/mysqld_exporter --config.my-cnf=/usr/local/mysqld_exporter/.my.cnf
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

2. 重载守护进程

```bash
systemctl daemon-reload
```

#### 2.2.3 启动服务

```bash
systemctl start mysqld_exporter
# 或者使用脚本启动
nohup /usr/local/mysqld_exporter/mysqld_exporter --config.my-cnf=/usr/local/mysqld_exporter/.my.cnf & # 后台启动
lsof -i:9104
```

### 2.3 容器监控

1. 安装

```bash
docker run -d \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:rw \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --publish=18080:8080 \
  --detach=true \
  --name=cadvisor \
  google/cadvisor:latest
```

2. 访问地址

?> http://localhost:8090/containers/

### 2.4 redis监控

#### 2.4.1 下载解压 

```bash
cd /opt/software
wget https://github.com/oliver006/redis_exporter/releases/download/v1.6.1/redis_exporter-v1.6.1.linux-amd64.tar.gz
tar zxvf redis_exporter-v1.6.1.linux-amd64.tar.gz -C /usr/local/
mv /usr/local/redis_exporter-v1.6.1.linux-amd64 /usr/local/redis_exporter
```

#### 2.4.2 添加系统服务

1. 创建启动文件

```bash
vim /usr/lib/systemd/system/redis_exporter.service
```

```bash
[Unit]
Description=redis_exporter
After=network.target

[Service]
ExecStart=/usr/local/redis_exporter/redis_exporter -redis.addr 172.17.17.192:16396 -redis.password "redis16396"
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

2. 重载守护进程

```bash
systemctl daemon-reload
```

#### 2.4.3 启动服务

```bash
systemctl start redis_exporter
# 或者使用脚本启动
nohup /usr/local/redis_exporter/redis_exporter -redis.addr 172.17.17.192:16396 -redis.password "redis16396" &   # 后台启动
lsof -i:9121
```

## 3. Grafana

### 3.1 下载

```bash
wget https://dl.grafana.com/oss/release/grafana-7.3.7-1.x86_64.rpm
```

### 3.2 安装

```bash
yum localinstall grafana-7.3.7-1.x86_64.rpm -y
```

### 3.3 启动服务

```bash
systemctl start grafana-server
systemctl enable grafana-server
```

### 3.4 客户端

?> 访问地址：http://ip:3000 账号密码：admin/admin

#### 2.4.1 linux监控

?> 监控模板：https://grafana.com/grafana/dashboards/8919

1. 添加Prometheus数据源

Configuration -> Data Sources ->add data source -> Prometheus

![](../../assets/_images/devops/deploy/prometheus/grafana1.png)
![](../../assets/_images/devops/deploy/prometheus/grafana2.png)
![](../../assets/_images/devops/deploy/prometheus/grafana3.png)

2. 导入模板

![](../../assets/_images/devops/deploy/prometheus/grafana4.png)

选择数据源

![](../../assets/_images/devops/deploy/prometheus/grafana5.png)


#### 2.4.2 mysql监控

?> 监控模板：https://grafana.com/grafana/dashboards/7362

#### 2.4.3 容器监控

?> 监控模板：https://grafana.com/grafana/dashboards/893

#### 2.4.4 redis监控

?> 监控模板：https://grafana.com/grafana/dashboards/763

