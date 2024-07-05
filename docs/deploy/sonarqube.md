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

#### 1.3.2 修改Sonar的JDK版本

```bash
vi /opt/sonarqube-7.8/conf/wrapper.conf
```

找到`wrapper.java.command`行，将路径修改为java目录

```conf
wrapper.java.command=/usr/local/jdk1.8.0_202/bin/java
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

### 1.5 创建启动用户

```bash
useradd sonar
useradd sonar
```

赋予启动用户执行权限

```bash
chown -R sonar:sonar /opt/sonarqube-7.8/
```

### 1.6 启动服务

```bash
su - sonar
cd /opt/sonarqube-7.8/bin/linux-x86-64
sh sonar.sh start
```


### 1.7 插件管理

#### 1.7.1 中文插件

把下载成功的插件复制到\extensions\plugins目录

```bash
cp /opt/software/sonarqube/sonar-l10n-zh-plugin-1.28.jar /opt/sonarqube-7.8/extensions/plugins
```

```bash
su - sonar
cd /opt/sonarqube-7.8/bin/linux-x86-64
sh sonar.sh restart
```

> 重启服务，如果是root用户上传后，需`chown -R sonar:sonar /opt/sonarqube-7.8/`，否则重启不成功

#### 1.7.2 p3c插件

P3C是根据《阿里巴巴Java开发手册》转化而成的自动化插件。

P3C原是海上巡逻机的型号。宽大机身可携带大量电子设备，翼下有十个武器外挂点，机腹下有八个内部炸弹舱，可携带AGM-65空地导弹、AGM-84反舰导弹、MK-46/50鱼雷和MU-90鱼雷以及深水炸弹、水雷等；被用来执行侦察、反潜、反水面、监视巡逻等海上任务。代码的世界里专治新手小毛病、老油条的各种不服

```bash
cp /opt/software/sonarqube/sonar-pmd-plugin-3.2.0-SNAPSHOT.jar /opt/sonarqube-7.8/extensions/plugins
```

重启服务

```bash
su - sonar
cd /opt/sonarqube-7.8/bin/linux-x86-64
sh sonar.sh restart
```

## 2. 应用设置

### 2.1 生成令牌

我的账号 -> 安全 -> 点击【生成】

![](../../assets/_images/deploy/sonarqube/create_token.png)

### 2.2 质量配置

管理员帐户登录，质量配置 -> 创建

![](../../assets/_images/deploy/sonarqube/add_rule.png)


激活规则

![](../../assets/_images/deploy/sonarqube/activate_rule.png)

![](../../assets/_images/deploy/sonarqube/activate_rule2.png)


设置规则

![](../../assets/_images/deploy/sonarqube/rule_set_default.png)

