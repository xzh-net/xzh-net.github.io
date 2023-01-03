# Windows 10

Windows 10 22H2 精简版基于微软官方原版安装包简化而成，为了使用户获得最佳的性能和舒适体验，本安装包删除、禁用、阻止了所有不需要的软件、后台服务以及对性能等产生影响的因素。因此精简系统可以提高电脑的性能、增强系统的稳定性，减少硬件资源的浪费，并且可以使配置低或老旧的电脑变的流畅。本系统的安装包仅内置了英文及俄文，但系统安装完成后，可通过安装简体中文语言包，将它切换成简体中文

## 1. 安装系统

1. 选择语言

![](../../assets/_images/deploy/win10/1.png)

![](../../assets/_images/deploy/win10/2.png)

2. 完全安装

![](../../assets/_images/deploy/win10/3.png)

3. 磁盘格式化

![](../../assets/_images/deploy/win10/4.png)

4. 等待安装

![](../../assets/_images/deploy/win10/5.png)

5. 选择地区

![](../../assets/_images/deploy/win10/6.png)

6. 选择键盘布局，然后单击跳过

![](../../assets/_images/deploy/win10/7.png)

![](../../assets/_images/deploy/win10/8.png)

7. 创建用户和输入密码

![](../../assets/_images/deploy/win10/9.png)

![](../../assets/_images/deploy/win10/10.png)

![](../../assets/_images/deploy/win10/11.png)

![](../../assets/_images/deploy/win10/12.png)

8. 进入桌面

![](../../assets/_images/deploy/win10/13.png)

9. 网络设置

![](../../assets/_images/deploy/win10/14.png)

10. 安装中文包

![](../../assets/_images/deploy/win10/15.png)
![](../../assets/_images/deploy/win10/16.png)
![](../../assets/_images/deploy/win10/17.png)
![](../../assets/_images/deploy/win10/18.png)
![](../../assets/_images/deploy/win10/19.png)

11. 时区设置

![](../../assets/_images/deploy/win10/20.png)
![](../../assets/_images/deploy/win10/21.png)
![](../../assets/_images/deploy/win10/22.png)
![](../../assets/_images/deploy/win10/23.png)
![](../../assets/_images/deploy/win10/24.png)
![](../../assets/_images/deploy/win10/25.png)
![](../../assets/_images/deploy/win10/26.png)

12. 激活

下载地址：https://github.com/massgravel/Microsoft-Activation-Scripts

## 2. 开发环境

### 2.1 Java

#### 2.1.1 设置环境变量

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

### 2.2 Tomcat

#### 2.2.1 设置jdk

```bash
# 修改catalina.sh
set JAVA_HOME=C:\Program Files\Java\jdk1.8.0_202
```

#### 2.2.2 生成证书

```bash
keytool -genkey -alias tomcat -keyalg RSA -keystore d:/tomcat.keystore
```

#### 2.2.3 ssl设置

```xml
<Connector port="443" protocol="org.apache.coyote.http11.Http11NioProtocol"
        maxThreads="150" SSLEnabled="true" scheme="https" secure="true"
        clientAuth="false" sslProtocol="TLS"
        keystoreFile="D:\tomcat.keystore" 
        keystorePass="123456" /> 
```

### 2.3 Golang

#### 2.3.1 下载

https://golang.google.cn/dl/

#### 2.3.2 设置环境变量

```java
GO111MODULE
on
GOPROXY
https://goproxy.cn
```

#### 2.3.2 模块初始化

```bash
go mod init xzh-net/markdown-renderer
```

### 2.4 Python Anaconda

#### 2.4.1 下载

https://mirrors.tuna.tsinghua.edu.cn/anaconda/miniconda/

#### 2.4.2 设置环境变量

```java
PATH
C:\ProgramData\Anaconda3
C:\ProgramData\Anaconda3\Scripts
C:\ProgramData\Anaconda3\Library\bin
C:\ProgramData\Anaconda3\Library\mingw-w64
```

#### 2.4.3 更换服务器源

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

#### 2.4.4 命令

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

### 2.5 Scala

下载地址：https://www.scala-lang.org/download/2.12.16.html

#### 2.5.1 设置环境变量

```java
SCALA_HOME
D:\scala\scala-2.12.16
PATH
%SCALA_HOME%\bin
```

## 3. 常用工具

### 3.1 VirtualBox

重置UUID

```bash
cd C:\Program Files\Oracle\VirtualBox
VBoxManage internalcommands sethduuid "D:\VirtualBox VMs\mediasoup\c7.vdi"
```