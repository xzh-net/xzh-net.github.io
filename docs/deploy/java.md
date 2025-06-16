# Java

## 1. Windows

### 1.1 安装JDK

- 官方网站：https://www.oracle.com/java/technologies/downloads/archive/
- 基线地址：https://javadl-esd-secure.oracle.com/update/baseline.version

```lua
JAVA_HOME
D:\java\jdk1.8.0_202

PATH
%JAVA_HOME%\bin
```

> cmd窗口执行 `java -version`

### 1.2 安装Maven

- 下载地址：https://archive.apache.org/dist/maven/maven-3/
- 版本对应：http://maven.apache.org/docs/history.html

```lua
MAVEN_HOME
D:\java\apache-maven-3.6.3

PATH
%MAVEN_HOME%\bin
```

> cmd窗口执行 `mvn -v`



### 1.3 安装Gradle

- 下载地址：https://services.gradle.org/distributions/

```lua
GRADLE_HOME
D:\java\gradle-8.8
GRADLE_USER_HOME
D:\Repositories\Maven

PATH
%GRADLE_HOME%\bin
```

> cmd窗口执行 `gradle -v`

### 1.4 安装Tomcat

- 下载地址：https://archive.apache.org/dist/tomcat/

#### 1.4.1 配置JDK版本

```bash
# 修改setclasspath.sh
JAVA_HOME=D:\Java\jdk1.8.0_202
JRE_HOME=D:\Java\jdk1.8.0_202\jre
```

#### 1.4.2 自签名证书

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

#### 1.4.3 服务管理

Tomcat针对manager内置了6个角色，分别为 admin-gui , admin-script , manager-gui , manager-script , manager-status , manager-jmx

负责Server Status、Manager App 功能的角色
   - manager-gui：访问 HTML 页面
   - manager-status：仅访问 “服务器状态” 页面
   - manager-script：访问脚本页面，以及 “服务器状态” 页面
   - manager-jmx：访问 JMX 代理接口和 “服务器状态” 页面

负责 Host Manager 功能的角色
   - admin-gui：访问HTML页面
   - admin-script： 访问脚本页面


进入tomcat的/conf目录下，打开tomcat-users.xml文件，复制以下内容到配置文件中

```conf
<tomcat-users>  
    <role rolename="manager-gui"/>
    <role rolename="manager-status"/>
    <role rolename="manager-script"/>
    <role rolename="manager-jmx"/>
    <role rolename="admin-gui"/>
    <role rolename="admin-script"/>
    <user username="tomcat" password="tomcat" roles="manager-gui,manager-status,manager-script,manager-jmx,admin-gui,admin-script"/>
</tomcat-users>
```

然后将/webapps/manager/META-INF/和webapps/host-manager/META-INF/下的context.xml进行修改（manager和host-manager目录下的context.xml两个都要改）

将context.xml的只针对本地请求放行这一设置注释掉即可，如下：

```conf
<Context antiResourceLocking="false" privileged="true" >
<!--  <Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" /> -->
  <Manager sessionAttributeValueClassNameFilter="java\.lang\.(?:Boolean|Integer|Long|Number|String)|org\.apache\.catalina\.filters\.CsrfPreventionFilter\$LruCache(?:\$1)?|java\.util\.(?:Linked)?HashMap"/>
</Context>
```

### 1.5 安装IDEA

下载地址：https://www.jetbrains.com/idea/download/other.html

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

### 2.1 安装JDK

8u202是jdk1.8最后一个免费商用版本

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

> `.bashrc`和`.profile`都可以解决非root用户下配置环境变量。区别在于：`.bashrc`每次打开‌非登录shell‌（如新终端窗口、标签页）时都会执行‌，`.profile`仅在用户通过‌登录shell‌（如SSH登录、终端登录）时执行一次‌。


### 2.2 安装Maven

#### 2.2.1 上传解压

```bash
cd /opt/software
tar -zxf apache-maven-3.6.3-bin.tar.gz -C /opt    # 解压
```

#### 2.2.2 配置环境变量

```bash
vim /etc/profile
# 添加
export MAVEN_HOME=/opt/apache-maven-3.6.3
export MAVEN_OPTS="-Xms1024m -Xmx1024m"
export PATH=$PATH:$MAVEN_HOME/bin

# 生效
source /etc/profile
mvn -v
```

#### 2.2.3 更换默认仓库源

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

#### 2.2.4 配置代理 

> 构建环境无法连接互联网，需要借助Nginx正向代理实现依赖下载

Maven代理配置

```xml
<proxies>
    <proxy>
        <id>optional</id>
        <active>true</active>
        <protocol>http</protocol>
        <host>192.168.1.100</host>
        <port>3182</port>
        <username></username>
        <password></password>
        <nonProxyHosts>www.google.com|*.example.com</nonProxyHosts>
    </proxy>
</proxies>
```

Nginx正向代理配置

```conf
server {
    listen 3182;
    resolver 114.114.114.114;
    proxy_connect;
    proxy_connect_allow            443 80;
    proxy_connect_connect_timeout  10s;
    proxy_connect_read_timeout     10s;
    proxy_connect_send_timeout     10s;
    access_log logs/access_proxy_$logdate.log access;
    location / {
        proxy_pass $scheme://$host$request_uri;
    }
}
```

执行以下命令后，通过nginx日志可以看到依赖包已经通过nginx代理下载到本地。

```bash
mvn dependency:get -Dartifact=org.apache.commons:commons-lang3:3.12.0
```