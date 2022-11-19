# Windows 10

## 1. 环境搭建

### 1.1 Java

#### 1.1.1 设置环境变量

```java
JAVA_HOME
D:\java\jdk1.8.0_202
MAVEN_HOME
D:\java\apache-maven-3.6.3
GRADLE_HOME
D:\java\gradle-6.8.3
GRADLE_USER_HOME
D:\Repositories\Maven
PATH
%JAVA_HOME%\bin
%MAVEN_HOME%\bin
%GRADLE_HOME%\bin
```

### 1.2 Tomcat

#### 1.2.1 设置jdk

```bash
# 修改catalina.sh
set JAVA_HOME=C:\Program Files\Java\jdk1.8.0_202
```

#### 1.2.2 生成证书

```bash
keytool -genkey -alias tomcat -keyalg RSA -keystore d:/tomcat.keystore
```

#### 1.2.3 ssl设置

```xml
<Connector port="443" protocol="org.apache.coyote.http11.Http11NioProtocol"
        maxThreads="150" SSLEnabled="true" scheme="https" secure="true"
        clientAuth="false" sslProtocol="TLS"
        keystoreFile="D:\tomcat.keystore" 
        keystorePass="123456" /> 
```

### 1.3 Golang

#### 1.3.1 下载

https://golang.google.cn/dl/

#### 1.3.2 设置环境变量

```java
GO111MODULE
on
GOPROXY
https://goproxy.cn
```

#### 1.3.2 模块初始化

```bash
go mod init xzh-net/markdown-renderer
```

### 1.4 Python Anaconda

#### 1.4.1 下载

https://mirrors.tuna.tsinghua.edu.cn/anaconda/miniconda/

#### 1.4.2 设置环境变量

```java
PATH
C:\ProgramData\Anaconda3
C:\ProgramData\Anaconda3\Scripts
C:\ProgramData\Anaconda3\Library\bin
C:\ProgramData\Anaconda3\Library\mingw-w64
```

#### 1.4.3 更换服务器源

打开`Anaconda Prompt`程序，执行`conda config --set show_channel_urls yes`

记事本打开`C:\Users\用户名\.condarc`文件, 将如下内容替换全部保存

```shell
channels:
  - defaults
show_channel_urls: true
default_channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/msys2
custom_channels:
  conda-forge: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  msys2: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  bioconda: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  menpo: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  pytorch: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  simpleitk: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
```

#### 1.4.4 命令

```bash
conda info -e                       # 查看当前系统中创建的虚拟环境，自带一个base环境
conda env list                      # 查看已创建的环境
conda create -n name python=3.8     # 创建名为name、Python版本为x.x的虚拟环境
conda activate name                 # 激活名为name的环境
conda install package_name          # 在当前环境中安装包
conda deactivate                    # 退出虚拟环境
conda remove -n name --all          # 删除名为name的虚拟环境
pip install pyhive pyspark jieba -i https://pypi.tuna.tsinghua.edu.cn/simple    # 在虚拟环境内安装包
```

### 1.5 Scala

下载地址：https://www.scala-lang.org/download/2.12.16.html

#### 1.1.1 设置环境变量

```java
SCALA_HOME
D:\scala\scala-2.12.16
PATH
%SCALA_HOME%\bin
```

## 2. VirtualBox

重置UUID

```bash
cd C:\Program Files\Oracle\VirtualBox
VBoxManage internalcommands sethduuid "D:\VirtualBox VMs\mediasoup\c7.vdi"
```