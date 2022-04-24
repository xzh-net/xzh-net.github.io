# Windows_10

## 1. VirtualBox

重置uuid

```bash
cd C:\Program Files\Oracle\VirtualBox
VBoxManage internalcommands sethduuid "D:\VirtualBox VMs\mediasoup\c7.vdi"
```

## 2. SSL证书

```bash
keytool -genkey -alias tomcat -keyalg RSA -keystore d:/tomcat.keystore
```


## 3. 开发环境

### 3.1 Java

```java
JAVA_HOME
D:\java\jdk1.8.0_40
MAVEN_HOME
D:\java\apache-maven-3.6.0
GRADLE_HOME
D:\java\gradle-6.8.3
GRADLE_USER_HOME
D:\Repositories\Maven
PATH
%JAVA_HOME%\bin
%MAVEN_HOME%\bin
%GRADLE_HOME%\bin
```

### 3.2 Golang

https://golang.google.cn/dl/

```lua
GO111MODULE
on

GOPROXY
https://goproxy.cn
```

模块初始化`go mod init xzh-net/markdown-renderer`