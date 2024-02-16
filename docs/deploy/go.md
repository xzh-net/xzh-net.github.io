# Go 1.13.4

## 1. Windows

### 1.1 安装Go

下载地址：https://golang.google.cn/dl/

### 1.2 安装GoLand

下载地址：https://www.jetbrains.com/zh-cn/go/

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
export GOPATH=/data/gocode                      # go 项目代码目录
export GOBIN=$GOPATH/bin                        # go install后生成的可执行命令存放路径
export PATH=$PATH:$GOROOT/bin                   # 全局配置
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