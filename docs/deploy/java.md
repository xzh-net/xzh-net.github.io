# Java 8

## 1. Windows

### 1.1 Jdk

设置环境变量

```lua
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

1. 指定jdk版本

```bash
# 修改catalina.sh
set JAVA_HOME=D:\Java\jdk1.8.0_202
```

2. 自签名证书

生成

```bash
keytool -genkey -alias tomcat -keyalg RSA -keystore d:/tomcat.keystore
```

配置ssl

```conf
<Connector port="443" protocol="org.apache.coyote.http11.Http11NioProtocol"
        maxThreads="150" SSLEnabled="true" scheme="https" secure="true"
        clientAuth="false" sslProtocol="TLS"
        keystoreFile="D:\tomcat.keystore" 
        keystorePass="123456" /> 
```

### 1.3 Idea

![](../../assets/_images/deploy/java/1.png)

![](../../assets/_images/deploy/java/2.png)

![](../../assets/_images/deploy/java/3.png)

![](../../assets/_images/deploy/java/4.png)

![](../../assets/_images/deploy/java/5.png)

![](../../assets/_images/deploy/java/6.png)

![](../../assets/_images/deploy/java/11.png)

设置主题

![](../../assets/_images/deploy/java/12.png)

设置字体

![](../../assets/_images/deploy/java/13.png)

设置Maven

![](../../assets/_images/deploy/java/14.png)

## 2. Linux

### 2.1 Jdk

Oracle最后一个商用免费版本

```bash
cd /opt/software
tar -zxvf jdk-8u202-linux-x64.tar.gz -C /usr/local/
vim /etc/profile

export JAVA_HOME=/usr/local/jdk1.8.0_202
export PATH=$PATH:$JAVA_HOME/bin
source /etc/profile   # 配置生效

# 免密分发
scp /etc/profile root@node02:/etc/
scp -r /usr/local/jdk1.8.0_202  root@node02:/usr/local/
```

### 2.2 Maven

1. 上传解压

```bash
cd /opt/software
tar -xzf apache-maven-3.6.3-bin.tar.gz -C /opt    # 解压
```

2. 配置环境变量

```bash
vim /etc/profile
# 添加
export MAVEN_HOME=/opt/apache-maven-3.6.3
export MAVEN_OPTS="-Xms4096m -Xmx4096m"
export PATH=$PATH:$MAVEN_HOME/bin

# 生效
source /etc/profile
mvn -v
```

3. 配置仓库

```bash
vim /opt/apache-maven-3.6.3/conf/settings.xml
```

```xml
<localRepository>/opt/repository</localRepository>
<mirrors>
    <!-- 阿里云仓库 -->
    <mirror>
        <id>alimaven</id>
        <mirrorOf>central</mirrorOf>
        <name>aliyun maven</name>
        <url>http://maven.aliyun.com/nexus/content/repositories/central/</url>
    </mirror>
    <!-- 中央仓库1 -->
    <mirror>
        <id>repo1</id>
        <mirrorOf>central</mirrorOf>
        <name>Human Readable Name for this Mirror.</name>
        <url>http://repo1.maven.org/maven2/</url>
    </mirror>
    <!-- 中央仓库2 -->
    <mirror>
        <id>repo2</id>
        <mirrorOf>central</mirrorOf>
        <name>Human Readable Name for this Mirror.</name>
        <url>http://repo2.maven.org/maven2/</url>
    </mirror>
</mirrors>
```
