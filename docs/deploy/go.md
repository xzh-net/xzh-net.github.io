# Go 1.13.4

- 官方网站：https://go.dev/

## 1. Windows

### 1.1 安装Go

1. 下载解压

https://dl.google.com/go/go1.22.0.windows-amd64.zip

2. 设置环境变量

配置GOROOT

![](../../assets/_images/deploy/go/1.png)

配置GOPATH

![](../../assets/_images/deploy/go/2.png)

配置PATH

![](../../assets/_images/deploy/go/3.png)

GO111MODULE设置

```bash
go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.cn,direct
```

### 1.2 安装GoLand

下载地址：https://www.jetbrains.com/zh-cn/go/

![](../../assets/_images/deploy/go/11.png)

![](../../assets/_images/deploy/go/12.png)

![](../../assets/_images/deploy/go/13.png)

![](../../assets/_images/deploy/go/14.png)

启动

![](../../assets/_images/deploy/go/15.png)

创建项目

![](../../assets/_images/deploy/go/16.png)

设置主题

![](../../assets/_images/deploy/go/17.png)

设置字体

![](../../assets/_images/deploy/go/18.png)

## 2. Linux

### 2.1 安装Go

1. 下载解压

```bash
cd /opt/software
wget https://dl.google.com/go/go1.13.4.linux-amd64.tar.gz
tar zxvf go1.13.4.linux-amd64.tar.gz -C /usr/local/
```

2. 设置环境变量

```bash
vi /etc/profile
```

```conf
export GOROOT=/usr/local/go                     # go 安装目录
export GOPATH=/data/go                          # go 项目代码目录
export GOBIN=$GOROOT/bin                        # go install后生成的可执行命令存放路径
export PATH=$PATH:$GOBIN                        # 全局配置
export GO111MODULE=on                           # 模块支持
export GOPROXY=https://goproxy.cn               # 模块代理
export http_proxy=http://172.17.17.165:7890     # 翻墙
export https_proxy=http://172.17.17.165:7890
```

```bash
source /etc/profile
```

3. 环境验证

```bash
go version
go env
```

### 2.2 第一个程序

```bash
cd /data/gocode
mkdir markdown && cd markdown       # 创建项目文件夹
go mod init markdown-renderer       # 初始项目
vi markdown_trest.go
```

```go
package main

import "fmt"

func init() {
    fmt.Println("初始化成功")
}
func main() {
    fmt.Println("markdown渲染器启动成功")
}
```

运行、编译、安装
```bash
go run markdown_trest.go
go build
go install
```