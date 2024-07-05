# SonarQube 7.8

SonarQube是一个用于管理代码质量的开放平台，可以快速的定位代码中潜在的或者明显的错误。目前支持java,C#,C/C++,Python,PL/SQL,Cobol,JavaScrip,Groovy等二十几种编程语言的代码质量管理与检测。

- 官网地址：https://www.sonarqube.org/
- 下载地址：https://binaries.sonarsource.com/?prefix=Distribution/sonarqube/

| **组件**  | **版本**  | **描述**  |
| :---------- | :---------- | :---------------------------------- |
| jdk    | 1.8.0 | Java运行环境 |
| PostgreSQL    | 12.4 | 数据库 |
| SonarQube    | 社区版7.8 | SonarQube 7.9开始需要Java 11且不支持mysql |

## 1. 安装

### 1.1 下载解压

```bash
cd /opt/software/
unzip sonarqube-7.8.zip 
mv sonarqube-7.8 /opt/
```

### 1.2 创建数据库

```sql
create user "sonar" with password '123456';
create database "sonardb" template template1 owner "sonar";
grant all privileges on database "sonardb" to "sonar";
```

### 1.3 修改配置

#### 1.3.1 修改数据库连接

```bash
vi /opt/sonarqube-7.8/conf/sonar.properties
```

添加以下配置

```conf
# 数据库连接信息
sonar.jdbc.url=jdbc:postgresql://xxx.xxx.xxx.xxx:5432/sonardb
sonar.jdbc.username=sonar
sonar.jdbc.password=123456
sonar.sorceEncoding=UTF-8
# SonarQube的web页面登录信息
sonar.login=admin
sonar.password=admin
```

#### 1.3.2 修改SonarQube的JDK版本

```bash
vi /opt/sonarqube-7.8/conf/wrapper.conf
```

找到`wrapper.java.command`行，将路径修改为java目录

```conf
wrapper.java.command=/usr/local/jdk1.8.0_202/bin/java
```

#### 1.3.3 修改ES限制

```bash
vi /opt/sonarqube-7.8/elasticsearch/config/elasticsearch.yml
```

```bash

```



### 1.4 系统设置

#### 1.4.1 修改文件资源限制

```bash
vi /etc/security/limits.conf
# 末尾添加
* soft nofile 65536
* hard nofile 65536
* soft nproc 131072
* hard nproc 131072
```

#### 1.4.2 修改虚拟内存

```bash
vi /etc/sysctl.conf
# 末尾添加
vm.max_map_count=655360

# 使配置生效
sysctl -p
```

### 1.5 安装插件

### 1.6 创建启动用户

```bash
useradd sonar
useradd sonar
```

赋予启动用户执行权限

```bash
chown -R sonar:sonar /opt/sonarqube-7.8/
```

### 1.7 启动服务

```bash
su - sonar
cd /opt/sonarqube-7.8/bin/linux-x86-64
sh sonar.sh start
```


## 2. 应用设置

### 2.1 
