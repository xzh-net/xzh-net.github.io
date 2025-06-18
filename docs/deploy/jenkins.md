# Jenkins 2.426 LTS

Jenkins是一个开源软件项目，是基于Java开发的一种持续集成工具，用于监控持续重复的工作，旨在提供一个开放易用的软件平台，使软件项目可以进行持续集成

下载地址：https://get.jenkins.io/redhat-stable/

## 1. 环境搭建

| **名称**  | **IP地址**  | **安装的软件**  |
| :---------- | :---------- | :---------------------------------- |
| 代码托管服务器    | 172.17.17.196 | Gitlab-12.4.2 |
| 持续集成服务器    | 172.17.17.200 | Jenkins-2.426，JDK17，Maven3.6.3，Git，SonarQube |
| 应用测试服务器    | 172.17.17.196 | JDK8，Tomcat8.5.100 |

### 1.1 Gitlab安装

[Gitlab](deploy/gitlab)

### 1.2 Jenkins安装

#### 1.2.1 下载安装

```bash
# 安装依赖
yum -y install epel-release daemonize
# 安装jdk17
wget https://download.oracle.com/java/17/latest/jdk-17_linux-x64_bin.rpm
rpm -ivh jdk-17_linux-x64_bin.rpm
# jenkins
wget https://get.jenkins.io/redhat-stable/jenkins-2.426.1-1.1.noarch.rpm
rpm -ivh jenkins-2.426.1-1.1.noarch.rpm
```

> 如果安装过，可以卸载

```bash
rpm -qa |grep jenkins
rpm -e --nodeps jenkins-2.426.1-1.1.noarch
find / -iname jenkins | xargs -n 1000 rm -rf
```

> 在Ubuntu 22.04上安装Jenkins时可能遇到 `Fontconfig head is null` 或者 `Could not execute systemctl：at /usr/bin/deb-systemd-invoke`，通常是由于系统缺少必要的字体或字体配置问题导致的，可以通过以下步骤解决：

```bash
sudo apt-get install -y fonts-dejavu-core fonts-freefont-ttf
sudo apt-get install -y fontconfig
```

#### 1.2.2 修改配置

1. 修改启动用户

生产环境不建议修改，会导致安全问题

```bash
vim /usr/lib/systemd/system/jenkins.service
```

```conf
User=jenkins
Group=jenkins
```

2. 关闭csrf保护

```bash
vi /usr/lib/systemd/system/jenkins.service
```

找到`JAVA_OPTS`参数，添加`-Dhudson.security.csrf.GlobalCrumbIssuerConfiguration.DISABLE_CSRF_PROTECTION=true`

```conf
Environment="JAVA_OPTS=-Djava.awt.headless=true -Dhudson.security.csrf.GlobalCrumbIssuerConfiguration.DISABLE_CSRF_PROTECTION=true"
```

重启服务

```bash
systemctl daemon-reload
systemctl restart jenkins
```

3. 关闭更新中心，跳过签名验证

离线安装的时候，需要关闭更新中心，跳过签名验证，否则会报错

```bash
vi /usr/lib/systemd/system/jenkins.service
```

找到`JAVA_OPTS`参数，追加以下内容
```conf
-Dhudson.model.UpdateCenter.never=true -Dhudson.model.DownloadService.noSignatureCheck=true
```

重启服务

```bash
systemctl daemon-reload
systemctl restart jenkins
```


#### 1.2.3 启动服务

```bash
systemctl daemon-reload
systemctl start jenkins
```

访问地址：http://172.17.17.200:8080/


![](../../assets/_images/deploy/jenkins/jenkins_skip1.png)

查看管理员密码
```bash
/var/lib/jenkins/secrets/initialAdminPassword
```

因为Jenkins插件需要连接默认官网下载，速度非常慢，所以我们暂时先跳过插件安装

![](../../assets/_images/deploy/jenkins/jenkins_skip2.png)

![](../../assets/_images/deploy/jenkins/jenkins_skip3.png)

![](../../assets/_images/deploy/jenkins/jenkins_skip4.png)

![](../../assets/_images/deploy/jenkins/jenkins_skip5.png)

![](../../assets/_images/deploy/jenkins/jenkins_skip6.png)


### 1.3 插件管理

#### 1.3.1 修改插件下载地址

Dashboard -> Manage Jenkins -> Plugins，点击`Advanced settings`，将`Update Site`地址改为`https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json`

![](../../assets/_images/deploy/jenkins/jenkins_plugin_tuna.png)


修改地址文件

```bash
cd /var/lib/jenkins/updates
sed -i 's/http:\/\/updates.jenkinsci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json
sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' default.json
```

重启服务：http://172.17.17.200:8888/restart

#### 1.3.2 下载中文汉化插件

安装`Localization: Chinese (Simplified)`插件

![](../../assets/_images/deploy/jenkins/jenkins_plugin_center.png)

![](../../assets/_images/deploy/jenkins/jenkins_plugin_chinese.png)


#### 1.3.3 插件安装方式（可选）

1. 在线安装

安装过程可能失败，点击下面的重启按钮，重启服务一般都能解决此种问题

2. 离线安装

下载插件到本地

https://mirrors.tuna.tsinghua.edu.cn/jenkins/plugins/publish-over-ssh/latest/publish-over-ssh.hpi

在插件管理中，点击高级，然后找到上传插件页面，点击上传刚刚下载的文件，找到后点击上传按钮

![](../../assets/_images/deploy/jenkins/jenkins_upload_plugin.png)

上传后，会自动开始安装插件及依赖关系，安装完成后仍然点击安装完成后重启，待重启完成后，就可以在插件中心看到。

3. 解压安装

因为主程序版本的迭代更新，插件会出现兼容问题导致无法安装，此时又不能升级主程序版本号，这时候就需要我们把之前部署好的环境中插件全部备份，在新环境还原即可。

```bash
# 备份
cd /var/lib/jenkins
tar -zcvf plugins.tar.gz plugins/

# 还原
tar -zxvf /opt/jenkins/plugins.tar.gz -C /var/lib/jenkins/
```


### 1.4 权限管理

#### 1.4.1 忘记密码

编辑config.xml文件，替换passwordHash行的内容123456

```bash
vim /var/lib/jenkins/users/admin_1679382529066310837/config.xml
<passwordHash>#jbcrypt:$2a$10$MiIVR0rr/UhQBqT.bBq0QehTiQVqgNpUGyWW2nJObaVAM/2xSQdSq</passwordHash>
systemctl restart jenkins
```

#### 1.4.2 关闭安全功能

```bash
vim /var/lib/jenkins/config.xml
<useSecurity>false</useSecurity>
systemctl restart jenkins
```

#### 1.4.3 下载权限插件

安装`Role-based Authorization Strategy`插件

![](../../assets/_images/deploy/jenkins/jenkins_plugin_role.png)

#### 1.4.4 开启权限全局安全配置

![](../../assets/_images/deploy/jenkins/jenkins_plugin_set.png)

授权策略切换为`Role-Based Strategy`

![](../../assets/_images/deploy/jenkins/jenkins_plugin_set2.png)

#### 1.4.5 创建角色

![](../../assets/_images/deploy/jenkins/jenkins_create_role.png)

![](../../assets/_images/deploy/jenkins/jenkins_create_role2.png)


添加以下三个角色：
- baseRole：该角色为全局角色。这个角色需要绑定Overall下面的Read权限，是为了给所有用户绑定最基本的Jenkins访问权限。注意：如果不给后续用户绑定这个角色，会报错误：用户名 is
missing the Overall/Read permission
- role1：该角色为项目角色。使用正则表达式绑定"xuzhihao.*"，意思是只能操作xuzhihao开头的项目。
- role2：该角色也为项目角色。绑定"xzh.*"，意思是只能操作xzh开头的项目。

![](../../assets/_images/deploy/jenkins/jenkins_role2.png)

#### 1.4.6 创建用户

![](../../assets/_images/deploy/jenkins/jenkins_create_user.png)

![](../../assets/_images/deploy/jenkins/jenkins_create_user2.png)

#### 1.4.7 用户角色绑定

系统管理页面进入Manage and Assign Roles，点击Assign Roles，绑定规则如下：

![](../../assets/_images/deploy/jenkins/jenkins_create_user3.png)

#### 1.4.8 创建项目测试权限

以admin管理员账户创建两个项目，分别为xuzhihao01和xzh01,zhangsan只能看到xuzhihao01，lisi只能看到xzh01


### 1.5 凭证管理

凭据可以用来存储需要密文保护的数据库密码、Gitlab密码信息、Docker私有仓库密码等，以便Jenkins可以和这些第三方的应用进行交互

#### 1.5.1 下载凭证插件

安装`Credentials Binding`插件

![](../../assets/_images/deploy/jenkins/jenkins_plugin_credentials.png)

安装成功后，多了一个凭证的菜单，可以管理所有凭证

![](../../assets/_images/deploy/jenkins/jenkins_plugin_credentials2.png)

![](../../assets/_images/deploy/jenkins/jenkins_plugin_credentials3.png)

![](../../assets/_images/deploy/jenkins/jenkins_plugin_credentials4.png)

可以添加的凭证有5种：
- Username with password：用户名和密码
- SSH Username with private key： 使用SSH用户和密钥
- Secret file：需要保密的文本文件，使用时Jenkins会将文件复制到一个临时目录中，再将文件路径设置到一个变量中，等构建结束后，所复制的Secret file就会被删除。
- Secret text：需要保存的一个加密的文本串，如钉钉机器人或Github的api token
- Certificate：通过上传证书文件的方式

常用的凭证类型有：`Username with password（用户密码）`和`SSH Username with private key（SSH密钥）`


#### 1.5.2 下载Git插件

安装`Git`插件

![](../../assets/_images/deploy/jenkins/jenkins_plugin_git.png)

CentOS7上安装Git工具

```bash
yum install git -y  # 安装
git --version       # 查看版本
```

#### 1.5.3 用户密码类型凭证

创建凭证

![](../../assets/_images/deploy/jenkins/jenkins_plugin_git2.png)

选择"Username with password"，输入Gitlab的用户名和密码，点击"确定"

![](../../assets/_images/deploy/jenkins/jenkins_plugin_git3.png)

测试凭证是否可用，创建一个FreeStyle项目：新建Item->FreeStyle Project->确定

![](../../assets/_images/deploy/jenkins/jenkins_create_project.png)

找到`源码管理`->`Git`，在`Repository URL`复制Gitlab中的项目URL

![](../../assets/_images/deploy/jenkins/jenkins_clone_project.png)

这时会报错说无法连接仓库！在Credentials选择刚刚添加的凭证就不报错啦

![](../../assets/_images/deploy/jenkins/jenkins_clone_project2.png)

保存配置后，点击构建”Build Now“ 开始构建项目

![](../../assets/_images/deploy/jenkins/jenkins_project_bulid.png)

![](../../assets/_images/deploy/jenkins/jenkins_project_bulid_console.png)

查看/var/lib/jenkins/workspace/，发现已经从Gitlab成功拉取了代码到Jenkins中。

#### 1.5.4 SSH密钥类型凭证

![](../../assets/_images/deploy/jenkins/jenkins_ssh_rsa.png)

jenkins服务器使用root用户生成公钥和私钥，并存放在/root/.ssh/目录下

```bash
ssh-keygen -t rsa
```

在Gitlab中添加凭证，配置公钥

`账户登录->点击头像->用户设置->SSH密钥`复制刚才id_rsa.pub文件的内容到这里，点击`添加密钥`

![](../../assets/_images/deploy/jenkins/jenkins_git_secret.png)

在Jenkins中添加凭证，配置私钥

![](../../assets/_images/deploy/jenkins/jenkins_plugin_credentials2.png)

![](../../assets/_images/deploy/jenkins/jenkins_git_ssh.png)

测试凭证是否可用

![](../../assets/_images/deploy/jenkins/jenkins_git_ssh2.png)

如果Jenkins新建任务时，输入远程仓库地址设置，选择凭证后一直提示连不上。提示：jenkins stderr: No ECDSA host key is known for gitee.com and you have requested strict checking. Host key verification failed。需要在jenkins的主机执行一下命令访问git上的仓库地址，把git的主机添加到/root/.ssh/known_hosts（执行命令前known_hosts这个文件是不存在的，执行后就有了）。

```bash
git ls-remote -h git@git.xxx.cn:root/website.git HEAD
```

### 1.6 Maven安装和配置

#### 1.6.1 Maven安装

上传`apache-maven-3.6.3-bin.tar.gz`到Jenkins服务器

```bash
cd /opt/software
tar -zxf apache-maven-3.6.3-bin.tar.gz -C /opt # 解压

vi /etc/profile
export JAVA_HOME=/usr/local/jdk1.8.0_202
export MAVEN_HOME=/opt/apache-maven-3.6.3
export PATH=$PATH:$JAVA_HOME/bin:$MAVEN_HOME/bin

source /etc/profile                   # 配置生效
mvn -v                                # 查找Maven版本
```

#### 1.6.2 全局配置关联JDK和Maven

Dashboard -> Manage Jenkins -> Tools，新增JDK和Maven配置如下：

![](../../assets/_images/deploy/jenkins/jenkins_jdk.png)

![](../../assets/_images/deploy/jenkins/jenkins_maven.png)

#### 1.6.3 添加Jenkins全局变量

Dashboard -> Manage Jenkins -> System - >全局属性 ，添加三个全局变量`JAVA_HOME`、`M2_HOME`、`PATH+EXTRA`

![](../../assets/_images/deploy/jenkins/jenkins_env.png)


#### 1.6.4 修改Maven配置文件

```bash
mkdir /opt/repository # 仓库目录
vi /opt/apache-maven-3.6.3/conf/settings.xml
```

本地仓库改为：/opt/repository/

添加阿里云私服地址：http://maven.aliyun.com/nexus/content/groups/public

```xml
<localRepository>/opt/repository/</localRepository>
<mirrors>
    <mirror>
            <id>nexus-aliyun</id>
            <mirrorOf>*,!jeecg,!jeecg-snapshots</mirrorOf>
            <name>Nexus aliyun</name>
            <url>http://maven.aliyun.com/nexus/content/groups/public</url>
    </mirror>
</mirrors>
```

#### 1.6.5 测试Maven是否配置成功

使用之前的测试项目->配置->构建步骤->增加构建步骤->Execute Shell

![](../../assets/_images/deploy/jenkins/jenkins_mvn_build.png)

![](../../assets/_images/deploy/jenkins/jenkins_mvn_build2.png)

![](../../assets/_images/deploy/jenkins/jenkins_mvn_build3.png)

> 输入命令：mvn clean package


### 1.7 Tomcat安装和配置

#### 1.7.1 安装Tomcat8.5

上传`apache-tomcat-8.5.100.tar.gz`到应用服务器

```bash
cd /opt/software/
tar -zxf apache-tomcat-8.5.100.tar.gz -C /opt    # 解压
/opt/apache-tomcat-8.5.100/bin/startup.sh        # 启动
```

> 地址为：http://192.168.3.201/8080

![](../../assets/_images/deploy/jenkins/jenkins_tomcat.png)

默认情况下Tomcat是没有配置用户角色权限的，后续Jenkins部署项目到Tomcat服务器，需要用到Tomcat的用户，所以修改tomcat以下配置，添加用户及权限

```bash
vi /opt/apache-tomcat-8.5.100/conf/tomcat-users.xml
```

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

用户和密码都是：tomcat，为了能够刚才配置的用户登录到Tomcat，还需要修改以下配置，将其注释掉以后重启服务

```bash
vi /opt/apache-tomcat-8.5.100/webapps/manager/META-INF/context.xml
```

```conf
<!--
<Valve className="org.apache.catalina.valves.RemoteAddrValve"
allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />
-->
```

> 访问： http://192.168.3.201:8080/manager/html ，输入tomcat和tomcat，看到以下页面代表成功

![](../../assets/_images/deploy/jenkins/jenkins_tomcat2.png)



## 2. 项目构建

Jenkins中自动构建项目的类型有很多，常用的有以下三种：
- 自由风格软件项目（FreeStyle Project）
- Maven项目（Maven Project）
- 流水线项目（Pipeline Project）

推荐使用流水线类型，因为灵活度非常高

### 2.1 自由风格项目

#### 2.1.1 拉取代码

![](../../assets/_images/deploy/jenkins/jenkins_create_project.png)

![](../../assets/_images/deploy/jenkins/jenkins_free_build.png)

![](../../assets/_images/deploy/jenkins/jenkins_free_build2.png)

#### 2.1.2 下载部署插件

Jenkins本身无法实现远程部署到Tomcat的功能，需要安装`Deploy to container`插件实现

![](../../assets/_images/deploy/jenkins/jenkins_plugin_deploy.png)

#### 2.1.3 添加Tomcat用户凭证

![](../../assets/_images/deploy/jenkins/jenkins_deploy_tomcat_credentials.png)

#### 2.1.4 添加构建后操作

![](../../assets/_images/deploy/jenkins/jenkins_tomcat_deploy.png)

![](../../assets/_images/deploy/jenkins/jenkins_tomcat_deploy2.png)

点击`Build Now`，开始构建过程

![](../../assets/_images/deploy/jenkins/jenkins_tomcat_deploy3.png)

#### 2.1.5 访问项目

![](../../assets/_images/deploy/jenkins/jenkins_tomcat_deploy4.png)


### 2.2 Maven项目

#### 2.2.1 下载Maven Integration插件

安装`Maven Integration`插件

![](../../assets/_images/deploy/jenkins/jenkins_plugin_maven.png)

#### 2.2.2 创建Maven项目

![](../../assets/_images/deploy/jenkins/jenkins_project_maven.png)

#### 2.2.3 配置项目

拉取代码和远程部署的过程和自由风格项目一样，只是`构建`部分不同

![](../../assets/_images/deploy/jenkins/jenkins_maven_build.png)


### 2.3 流水线项目

- Pipeline 脚本是由 Groovy 语言实现的
- Pipeline 支持两种语法：Declarative Pipeline（声明式） 和 Scripted Pipeline（脚本式）
- Pipeline 也有两种创建方法：可以直接在Jenkins的Web UI 界面中输入脚本；也可以通过创建一个Jenkinsfile脚本文件放入项目源码库中（一般我们都推荐在 Jenkins 中直接从源代码控制（SCM）中直接载入 Jenkinsfile Pipeline 这种方法）。


#### 2.3.1 下载Pipeline插件

安装`Pipeline`插件

![](../../assets/_images/deploy/jenkins/jenkins_plugin_pipeline.png)


#### 2.3.2 语法快速入门

创建项目

![](../../assets/_images/deploy/jenkins/jenkins_project_pipeline.png)

1. Declarative Pipeline 声明式快速入门

![](../../assets/_images/deploy/jenkins/jenkins_project_pipeline1.png)

选择模板或者使用片段生成器

![](../../assets/_images/deploy/jenkins/jenkins_project_pipeline2.png)


```shell
pipeline {
   agent any

   stages {
      stage('代码拉取') {
         steps {
            echo '代码拉取'
         }
      }
      stage('编译构建') {
         steps {
            echo '编译构建'
         }
      }
      stage('项目部署') {
         steps {
            echo '项目部署'
         }
      }
   }
}
```

- stages：代表整个流水线的所有执行阶段。通常stages只有1个，里面包含多个stage
- stage：代表流水线中的某个阶段，可能出现n个。一般分为拉取代码，编译构建，部署等阶段。
- steps：代表一个阶段内需要执行的逻辑。steps里面是shell脚本，git拉取代码，ssh远程发布等任意内容。

![](../../assets/_images/deploy/jenkins/jenkins_project_pipeline3.png)


2. Scripted Pipeline 脚本式快速入门

![](../../assets/_images/deploy/jenkins/jenkins_project_pipeline4.png)


```shell
node {
    def mvnHome
    stage('拉取代码') { // for display purposes
        echo '拉取代码'
    }
    stage('编译构建') {
        echo '编译构建'
    }
    stage('项目部署') {
        echo '项目部署'
    }
}
```

- Node：节点，一个 Node 就是一个 Jenkins 节点，Master 或者 Agent，是执行 Step 的具体运行环境，后续讲到Jenkins的Master-Slave架构的时候用到。
- Stage：阶段，一个 Pipeline 可以划分为若干个 Stage，每个 Stage 代表一组操作，比如：Build、Test、Deploy，Stage 是一个逻辑分组的概念。

#### 2.3.3 拉取代码

使用语法生成器

```shell
pipeline {
   agent any

   stages {
      stage('代码拉取') {
         steps {
            checkout([$class: 'GitSCM', branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[credentialsId: '4cba62e6-1a9f-4664-97f9-0f814dc728c9', url: 'ssh://git@172.17.17.196:222/xzh-group/xzh-spring-boot.git']]])
         }
      }
   }
}
```

#### 2.3.5 编译打包

```shell
pipeline {
   agent any

   stages {
      stage('代码拉取') {
         steps {
            checkout([$class: 'GitSCM', branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[credentialsId: '4cba62e6-1a9f-4664-97f9-0f814dc728c9', url: 'ssh://git@172.17.17.196:222/xzh-group/xzh-spring-boot.git']]])
         }
      }
      stage('编译构建') {
         steps {
            sh 'mvn clean package -Dmaven.test.skip=true '
         }
      }
   }
}
```

#### 2.3.6 部署

```shell
pipeline {
   agent any

   stages {
      stage('代码拉取') {
         steps {
            checkout([$class: 'GitSCM', branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[credentialsId: '4cba62e6-1a9f-4664-97f9-0f814dc728c9', url: 'ssh://git@172.17.17.196:222/xzh-group/xzh-spring-boot.git']]])
         }
      }
      stage('编译构建') {
         steps {
            sh 'mvn clean package -Dmaven.test.skip=true '
         }
      }
      stage('项目部署') {
         steps {
            deploy adapters: [tomcat8(credentialsId: '9cfdcd8f-7b51-428d-ae79-25d64e70455a', path: '', url: 'http://172.17.17.196:8080')], contextPath: '/ddd', war: 'target/*.war'
         }
      }
   }
}
```

构建后查看结果

#### 2.3.7 Pipeline Script from SCM

刚才我们都是直接在Jenkins的UI界面编写Pipeline代码，这样不方便脚本维护，建议把Pipeline脚本放在项目中（一起进行版本控制）,在项目根目录建立Jenkinsfile文件，把内容复制到该文件中并上传到Gitlab


![](../../assets/_images/deploy/jenkins/jenkins_scm1.png)


在项目根目录创建该文件，名为`Jenkinsfile`

```shell
pipeline {
   agent any

   stages {
      stage('代码拉取') {
         steps {
            checkout([$class: 'GitSCM', branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[credentialsId: '4cba62e6-1a9f-4664-97f9-0f814dc728c9', url: 'ssh://git@172.17.17.196:222/xzh-group/xzh-spring-boot.git']]])
         }
      }
      stage('编译构建') {
         steps {
            sh 'mvn clean package -Dmaven.test.skip=true -Pdev'
         }
      }
      stage('项目部署') {
         steps {
            deploy adapters: [tomcat8(credentialsId: '9cfdcd8f-7b51-428d-ae79-25d64e70455a', path: '', url: 'http://172.17.17.196:8080')], contextPath: '/ddd', war: 'target/*.war'
         }
      }
   }
}
```

## 3. 高级构建

### 3.1 构建触发器

Jenkins内置4种构建触发器：
- 触发远程构建
- 其他工程构建后触发（Build after other projects are build）
- 定时构建（Build periodically）
- 轮询SCM（Poll SCM）


#### 3.1.1 触发远程构建

![](../../assets/_images/deploy/jenkins/jenkins_build_triger1.png)

触发构建url：http://172.17.17.200:8080/job/jenkins_project_pipeline/build?token=123456

如何触发构建但是不需要登录可以借助`Build Authorization Token Root Plugin`插件实现

#### 3.1.2 其他工程构建后触发

![](../../assets/_images/deploy/jenkins/jenkins_build_after.png)

#### 3.1.3 定时构建

![](../../assets/_images/deploy/jenkins/jenkins_build_task.png)

```bash
# 每十五分钟（可能在 :07, :22, :37, :52）：
H/15 * * * *
# 每小时前半段每十分钟一次（三次，可能在 :04, :14, :24）：
H(0-29)/10 * * * *
# 每两小时一次，从每周日上午 9 点 45 分开始，到下午 3 点 45 分结束，每小时 45 分钟：
45 9-16/2 * * 1-5
# 每个工作日上午 8 点到下午 4 点之间每两小时一次（可能在上午 9 点 38 分、上午 11 点 38 分、下午 1 点 38 分、下午 3 点 38 分）：
HH(8-15)/2 * * 1-5
# 除 12 月外，每月 1 日和 15 日每天一次：
HH 1,15 1-11 *
```

#### 3.1.4 轮询SCM

轮询SCM，是指定时扫描本地代码仓库的代码是否有变更，如果代码有变更就触发项目构建。

> 注意：这次构建触发器，Jenkins会定时扫描本地整个项目的代码，增大系统的开销，不建议使用

![](../../assets/_images/deploy/jenkins/jenkins_build_scm.png)

### 3.2 Git hook自动触发构建

利用Gitlab的webhook实现代码push到仓库，立即触发项目自动构建

#### 3.2.1 下载GitLab插件

安装`Generic Webhook Trigger`和`GitLab`插件

![](../../assets/_images/deploy/jenkins/jenkins_plugin_gitwebhook.png)


![](../../assets/_images/deploy/jenkins/jenkins_plugin_hook2.png)

#### 3.2.2 Jenkins设置自动构建

![](../../assets/_images/deploy/jenkins/jenkins_project_webhook.png)

> 注意：以下设置必须完成，否则会报错！Manage Jenkins -> System

![](../../assets/_images/deploy/jenkins/jenkins_plugin_gitwebhook2.png)

#### 3.2.3 Gitlab配置webhook

1. 开启webhook功能

使用root账户登录到后台，点击Admin Area -> Settings -> Network，勾选`Allow requests to the local network from web hooks and services`

![](../../assets/_images/deploy/jenkins/jenkins_gitlab_webhook.png)

2. 在项目添加webhook

![](../../assets/_images/deploy/jenkins/jenkins_gitlab_webhook2.png)


### 3.3 参数化构建

#### 3.3.1 新增构建参数

有时在项目构建的过程中，我们需要根据用户的输入动态传入一些参数，从而影响整个构建结果，这时我们可以使用参数化构建。

Jenkins支持非常丰富的参数类型，在Jenkins添加字符串类型参数

![](../../assets/_images/deploy/jenkins/jenkins_build_param0.png)

![](../../assets/_images/deploy/jenkins/jenkins_build_param1.png)

#### 3.3.2 修改流水线代码

![](../../assets/_images/deploy/jenkins/jenkins_build_param2.png)

点击Build with Parameters

![](../../assets/_images/deploy/jenkins/jenkins_build_param3.png)


### 3.4 配置邮箱服务器发送构建结果

#### 3.4.1 下载发送邮件插件

安装`Email Extension Template`插件

![](../../assets/_images/deploy/jenkins/jenkins_plugin_email.png)


#### 3.4.2 Jenkins设置邮箱发件人

Manage Jenkins -> System

![](../../assets/_images/deploy/jenkins/jenkins_mail_admin.png)


#### 3.4.3 Jenkins设置邮件参数

Manage Jenkins -> System -> Extended E-mail Notification

![](../../assets/_images/deploy/jenkins/jenkins_mail_admin2.png)

![](../../assets/_images/deploy/jenkins/jenkins_mail_admin3.png)


#### 3.4.4 Jenkins设置默认邮箱信息

![](../../assets/_images/deploy/jenkins/jenkins_mail_admin4.png)

> 邮件通知和凭证中使用的密码均为`授权码`


#### 3.4.5 设置邮件模板

准备邮件内容,在项目根目录编写email.html，并把文件推送到Gitlab，内容如下

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>${ENV, var="JOB_NAME"}-第${BUILD_NUMBER}次构建日志</title>
</head>

<body leftmargin="8" marginwidth="0" topmargin="8" marginheight="4"
      offset="0">
<table width="95%" cellpadding="0" cellspacing="0"
       style="font-size: 11pt; font-family: Tahoma, Arial, Helvetica, sans-serif">
    <tr>
        <td>(本邮件是程序自动下发的，请勿回复！)</td>
    </tr>
    <tr>
        <td><h2>
            <font color="#0000FF">构建结果 - ${BUILD_STATUS}</font>
        </h2></td>
    </tr>
    <tr>
        <td><br />
            <b><font color="#0B610B">构建信息</font></b>
            <hr size="2" width="100%" align="center" /></td>
    </tr>
    <tr>
        <td>
            <ul>
                <li>项目名称&nbsp;：&nbsp;${PROJECT_NAME}</li>
                <li>构建编号&nbsp;：&nbsp;第${BUILD_NUMBER}次构建</li>
                <li>触发原因：&nbsp;${CAUSE}</li>
                <li>构建日志：&nbsp;<a href="${BUILD_URL}console">${BUILD_URL}console</a></li>
                <li>构建&nbsp;&nbsp;Url&nbsp;：&nbsp;<a href="${BUILD_URL}">${BUILD_URL}</a></li>
                <li>工作目录&nbsp;：&nbsp;<a href="${PROJECT_URL}ws">${PROJECT_URL}ws</a></li>
                <li>项目&nbsp;&nbsp;Url&nbsp;：&nbsp;<a href="${PROJECT_URL}">${PROJECT_URL}</a></li>
            </ul>
        </td>
    </tr>
    <tr>
        <td><b><font color="#0B610B">Changes Since Last
            Successful Build:</font></b>
            <hr size="2" width="100%" align="center" /></td>
    </tr>
    <tr>
        <td>
            <ul>
                <li>历史变更记录 : <a href="${PROJECT_URL}changes">${PROJECT_URL}changes</a></li>
            </ul> ${CHANGES_SINCE_LAST_SUCCESS,reverse=true, format="Changes for Build #%n:<br />%c<br />",showPaths=true,changesFormat="<pre>[%a]<br />%m</pre>",pathFormat="&nbsp;&nbsp;&nbsp;&nbsp;%p"}
        </td>
    </tr>
    <tr>
        <td><b>Failed Test Results</b>
            <hr size="2" width="100%" align="center" /></td>
    </tr>
    <tr>
        <td><pre
                style="font-size: 11pt; font-family: Tahoma, Arial, Helvetica, sans-serif">$FAILED_TESTS</pre>
            <br /></td>
    </tr>
    <tr>
        <td><b><font color="#0B610B">构建日志 (最后 100行):</font></b>
            <hr size="2" width="100%" align="center" /></td>
    </tr>
    <tr>
        <td><textarea cols="80" rows="30" readonly="readonly"
                      style="font-family: Courier New">${BUILD_LOG, maxLines=100}</textarea>
        </td>
    </tr>
</table>
</body>
</html>
```

#### 3.4.6 流水线增加构建后发送邮件

编写Jenkinsfile添加构建后发送邮件

```shell
pipeline {
   agent any

   stages {
      stage('代码拉取') {
         steps {
            checkout([$class: 'GitSCM', branches: [[name: '*/${branch}']], extensions: [], userRemoteConfigs: [[credentialsId: '4cba62e6-1a9f-4664-97f9-0f814dc728c9', url: 'ssh://git@172.17.17.196:222/xzh-group/xzh-spring-boot.git']]])
         }
      }
      stage('编译构建') {
         steps {
            sh 'mvn clean package -Dmaven.test.skip=true '
         }
      }
      stage('项目部署') {
         steps {
            deploy adapters: [tomcat8(credentialsId: '9cfdcd8f-7b51-428d-ae79-25d64e70455a', path: '', url: 'http://172.17.17.196:8080')], contextPath: '/aaa', war: 'target/*.war'
         }
      }
   }
   post {
         always {
            emailext(
               subject: '构建通知：${PROJECT_NAME} - Build # ${BUILD_NUMBER} - ${BUILD_STATUS}!',
               body: '${FILE,path="email.html"}',
               to: 'xcg992224@163.com'
            )
         }
   }
}
```

### 3.5 代码审查


#### 3.5.1 安装SonarQube

SonarQube是一个用于管理代码质量的开放平台，可以快速的定位代码中潜在的或者明显的错误。目前支持java,C#,C/C++,Python,PL/SQL,Cobol,JavaScrip,Groovy等二十几种编程语言的代码质量管理与检测。

#### 3.5.2 下载SonarQube Scanner插件

安装`SonarQube Scanner`插件

![](../../assets/_images/deploy/jenkins/jenkins_plugin_sonarqube.png)

#### 3.5.3 添加SonarQube凭证

![](../../assets/_images/deploy/jenkins/jenkins_sonarqube_credentials1.png)

![](../../assets/_images/deploy/jenkins/jenkins_sonarqube_credentials2.png)

#### 3.5.4 全局配置关联SonarScanner扫描器

Dashboard -> Manage Jenkins -> Tools，新增SonarScanner配置如下：

![](../../assets/_images/deploy/jenkins/jenkins_sonar_scanner.png)

#### 3.5.5 添加Jenkins全局变量

Dashboard -> Manage Jenkins -> System，新增SonarQube配置如下：

![](../../assets/_images/deploy/jenkins/jenkins_sonarqube_system.png)

#### 3.5.6 SonaQube关闭审查结果上传到SCM功能

![](../../assets/_images/deploy/jenkins/jenkins_close_scm.png)

#### 3.5.7 代码中增加SonarScanner扫描规则

项目根目录添加`sonar-project.properties`文件

```properties
sonar.projectKey=web_demo
sonar.projectName=web_demo
sonar.java.binaries=/
```

> 项目中如果无法添加`sonar-project.properties`文件，可以在Jenkins流水线中声明规则并添加如下代码

```bash
stage('构建规则') {
    // 定义文件的路径和名称
    def filePath = 'sonar-project.properties'
    // 定义文件的内容
    def fileContent = 'sonar.projectKey=web_demo\n sonar.projectName=web_demo\n sonar.java.binaries=/\n'
    // 使用 writeFile 步骤来创建文件
    writeFile file: filePath, text: fileContent
    
    echo "构建规则完毕"
}
    
stage('代码审查') {
    script {
        scannerHome = tool 'sonarqube-scanner'
    }
    withSonarQubeEnv('sonarqube7.8') {
        sh "${scannerHome}/bin/sonar-scanner"
    }
}
```



#### 3.5.8 修改流水线代码

```shell
pipeline {
   agent any

   stages {
      stage('代码拉取') {
         steps {
            checkout([$class: 'GitSCM', branches: [[name: '*/${branch}']], extensions: [], userRemoteConfigs: [[credentialsId: '4cba62e6-1a9f-4664-97f9-0f814dc728c9', url: 'ssh://git@172.17.17.196:222/xzh-group/xzh-spring-boot.git']]])
         }
      }
      stage('代码审查') {
         script {
            scannerHome = tool 'sonarqube-scanner'
         }
        withSonarQubeEnv('sonarqube7.8') {
            sh "${scannerHome}/bin/sonar-scanner"
        }
      }
      stage('编译构建') {
         steps {
            sh 'mvn clean package -Dmaven.test.skip=true '
         }
      }
      stage('项目部署') {
         steps {
            deploy adapters: [tomcat8(credentialsId: '9cfdcd8f-7b51-428d-ae79-25d64e70455a', path: '', url: 'http://172.17.17.196:8080')], contextPath: '/aaa', war: 'target/*.war'
         }
      }
   }
   post {
         always {
            emailext(
               subject: '构建通知：${PROJECT_NAME} - Build # ${BUILD_NUMBER} - ${BUILD_STATUS}!',
               body: '${FILE,path="email.html"}',
               to: 'xcg992224@163.com'
            )
         }
   }
}
```

![](../../assets/_images/deploy/jenkins/jenkins_build_sonar.png)


## 4. 微服务持续集成

### 4.1 静态网站发布

#### 4.1.1 下载NodeJS插件

安装`NodeJS`插件

![](../../assets/_images/deploy/jenkins/jenkins_plugin_node.png)

#### 4.1.2 全局配置关联NodeJS

Dashboard -> Manage Jenkins -> Tools，新增NodeJS配置如下：

![](../../assets/_images/deploy/jenkins/jenkins_plugin_node2.png)

#### 4.1.3 建立Jenkinsfile构建脚本

```shell
node {
   stage('代码拉取') {
		checkout([$class: 'GitSCM', branches: [[name: '*/${branch}']], extensions: [], userRemoteConfigs: [[credentialsId: '4cba62e6-1a9f-4664-97f9-0f814dc728c9', url: 'git@git.vjspnet.cn:root/cogent-auto-platform-web.git']]])
   }
   stage('编译构建') {
      nodejs('NodeJS16'){
         sh '''
               npm install --registry=https://registry.npmmirror.com
               npm run build:test
         '''
      }
   }
}
```

构建结果

![](../../assets/_images/deploy/jenkins/jenkins_node_suscess.png)

#### 4.1.4 安装插件Publish Over SSH

安装`Publish Over SSH`插件

![](../../assets/_images/deploy/jenkins/jenkins_plugin_ssh.png)


#### 4.1.5 添加SSH全局服务

Dashboard -> Manage Jenkins -> System -> 全局属性 ，配置SSH服务器站点，ip，账号，密码

![](../../assets/_images/deploy/jenkins/jenkins_plugin_ssh2.png)

#### 4.1.6 修改流水线代码

增加了`项目部署`的内容，脚本内容可以使用片段生成器生成。

```shell
node {
   stage('代码拉取') {
		checkout([$class: 'GitSCM', branches: [[name: '*/${branch}']], extensions: [], userRemoteConfigs: [[credentialsId: '4cba62e6-1a9f-4664-97f9-0f814dc728c9', url: 'git@git.vjspnet.cn:root/cogent-auto-platform-web.git']]])
   }
   stage('编译构建') {
      nodejs('NodeJS16'){
         sh '''
               npm install --registry=https://registry.npmmirror.com
               npm run build:test
         '''
      }
   }
   stage('项目部署') {
        sshPublisher(publishers: [sshPublisherDesc(configName: 'localhost161',transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: '/usr/local/nginx/sbin/nginx -s reload',execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes:false, patternSeparator: '\\n', remoteDirectory:'/data/html/'+'${JOB_NAME}'+'/'+'${BUILD_NUMBER}',remoteDirectorySDF: false, removePrefix: 'dist', sourceFiles: 'dist/**')],usePromotionTimestamp: false,useWorkspaceInPromotion: false, verbose: false)])
    }

}
```

#### 4.1.7 安装插件SSH Pipeline Steps

> `SSH Pipeline Steps`另一款发布插件 。`Publish Over SSH`插件依赖全局SSH配置，无法使用密钥，注重于文件传输，而`SSH Pipeline Steps`插件可以配置密钥，注重于命令执行。脚本如下：

```shell
pipeline {
    agent any
    stages {
        stage('用户名密码方式验证 ssh 连接') {
            steps {
                script {
                    remoteConfig = [:]
                    remoteConfig.name = "my-remote-server01"
                    remoteConfig.host = "172.17.17.161"
                    remoteConfig.allowAnyHosts = true
                    remoteConfig.port = 22
                    remoteConfig.user = "root"
                    // SSH 登录密码
                    remoteConfig.password = "123456"
                    writeFile(file: 'test.sh', text: 'ls')
                    sshCommand(remote: remoteConfig, command: 'for i in {1..5}; do echo -n \"Loop \$i \"; date ; sleep 1; done')
                    sshScript(remote: remoteConfig, script: 'test.sh')
                    sshPut(remote: remoteConfig, from: 'test.sh', into: '.')
                    sshGet(remote: remoteConfig, from: 'test.sh', into: 'test_new.sh', override: true)
                    sshRemove(remote: remoteConfig, path: 'test.sh')
                }
            }
        }

        stage("凭证方式验证 ssh 连接") {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: "3628ad76-f4c1-4999-ac39-fd6fd3674ac5",
                    keyFileVariable: "identity"
                )]) {
                    script {
                        remoteConfig = [:]
                        remoteConfig.name = "my-remote-server02"
                        remoteConfig.host = "172.17.17.161"
                        remoteConfig.allowAnyHosts = true
                        remoteConfig.user = "root"
                        remoteConfig.port = 22
                        // SSH 私钥文件地址
                        remoteConfig.identityFile = identity
                        writeFile(file: 'test.sh', text: 'ls')
                        sshCommand(remote: remoteConfig, command: 'for i in {1..5}; do echo -n \"Loop \$i \"; date ; sleep 1; done')
                        sshScript(remote: remoteConfig, script: 'test.sh')
                        sshPut(remote: remoteConfig, from: 'test.sh', into: '.')
                        sshGet(remote: remoteConfig, from: 'test.sh', into: 'test_new.sh', override: true)
                        sshRemove(remote: remoteConfig, path: 'test.sh')
                    }
                }  
            }
        }
    }
}
```

> 认证机制：Jenkins需要提供的能匹配目标机器上某个用户公钥的私钥，该用户对应的公钥在目标机器`~/.ssh/authorized_keys`文件中存放
- 私钥来源：目标机器上对应用户的私钥（如 ~/.ssh/id_rsa）。
- 公钥位置：目标机器的 ~/.ssh/authorized_keys中必须有对应的公钥。

目标机器生成密钥对
```bash
ssh-keygen -t rsa -f ~/.ssh/jenkins_agent_key
```

将公钥添加到目标机器的`authorized_keys`文件中
```bash
cat ~/.ssh/jenkins_agent_key.pub >> ~/.ssh/authorized_keys
```

Jenkins配置Credentials，选择`SSH Username with private key`，将目标机器私钥粘贴到`Enter directly`中，保存即可。

### 4.2 Maven项目构建发布

#### 4.2.1 下载参数选择插件

安装`Extended Choice Parameter`插件

![](../../assets/_images/deploy/jenkins/jenkins_plugin_choice_parameter.png)

#### 4.2.2 添加参数

添加参数选择`Extended Choice Parameter`类型

![](../../assets/_images/deploy/jenkins/jenkins_choice_parameter_add.png)

新建项目名称参数

![](../../assets/_images/deploy/jenkins/jenkins_choice_parameter_add2.png)

设置参数值和显示文字

![](../../assets/_images/deploy/jenkins/jenkins_choice_parameter_add3.png)


#### 4.2.2 修改流水线代码

```shell
node {
    //全局配置
    //备份目录
    def backup_dir = "/opt/app/backup"

    //项目配置
    //项目编译默认根路径
    def service_path = '';
    //目标机器ssh信息
    def target_ssh = "root@192.168.3.201"
    //目标传输路径
    def target_dir = "/data/application/microservices-platform"

    //参数配置
    //获取当前选择的项目名称
    def apps = "${project_name}".split(",")

    //逻辑正文
    stage('代码拉取') {
        checkout([$class: 'GitSCM', branches: [[name: '*/${branch}']], extensions: [], userRemoteConfigs: [[credentialsId: '97c1f104-6dca-4792-993a-3b2b418344a1', url: 'git@git.vjspnet.cn:root/cogent-auto-platform-service.git']]])
    }
    stage('编译并安装公共工程') {
        sh "mvn -f commons clean install"
    }

    stage('编译，部署，重启') {
        for (int i = 0; i < apps.size(); ++i) {
            //自定义不同子模块构建路径
            if ("${apps[i]}" == 'sc-gateway') {
                service_path = ''
            } else {
                service_path = 'business/'
            }
            //编译
            sh "mvn -f ${service_path}${apps[i]} clean package -Dmaven.test.skip=true"
            //备份
            sh "cp " + service_path + "${apps[i]}/target/*.jar ${backup_dir}/${project_version}/"
            //保留历史处理
            sh ""
            //传输,基于免密
            sh "scp " + service_path + "${apps[i]}/target/*.jar ${target_ssh}:${target_dir}"
            //重启
            sh "ssh ${target_ssh} 'cd ${target_dir};sh shutdown.sh ${apps[i]}'"
        }  
    }
}
```

#### 4.2.3 密钥设置

> jenkins服务器需要免密登录到目标服务器

1. 获取jenkins服务器公钥

```bash
cat ~/.ssh/id_rsa.pub
```

2. 如果上一步没有这个文件我们就创建一个

```bash
ssh-keygen -t rsa
```

3. 把生成的公钥发送到对方的主机上去

```bash
cd /root/.ssh
ssh-copy-id -i id_rsa.pub root@192.168.3.201
```

4. 测试

```bash
ssh 192.168.3.201
```

#### 4.2.4 构建项目

![](../../assets/_images/deploy/jenkins/jenkins_choice_parameter_add4.png)


### 4.3 Dockerfile镜像构建发布

安装插件`Docker Pipeline`


```yaml
pipeline {
    agent any
    stages {
        stage('代码拉取') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[credentialsId: '210e7696-63a7-4ee4-ac31-48141432330d', url: 'http://172.17.17.136:18080/xuzhihao/spring.git']]])
                sh 'mvn clean package -Dmaven.test.skip=true '
            }
        }
        stage('构建镜像') {
            environment {
                // harbor密钥
                HARBOR_CRED = "3379dde3-91e7-401d-9023-5d8afd2f7303"
                // harbor仓库地址
                HARBOR_URL = "172.17.17.37:80"
                // 租户信息
                HARBOR_PROJECT = "xzh"
                // 镜像信息
                IMAGE_APP = "demo"
                IMAGE_TAG = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                IMAGE_NAME = "${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_APP}:${IMAGE_TAG}"
            }
            steps {
                echo '开始构建镜像'
                script {
                    docker.build "${IMAGE_NAME}"
                }
                echo '构建镜像完成'
                echo '开始推送镜像'
                script {
                    docker.withRegistry("http://${HARBOR_URL}", "${HARBOR_CRED}") {
                        docker.image("${IMAGE_NAME}").push()
                    }
                }
                echo '推送镜像完成'
                echo '开始删除镜像'
                script {
                    sh "docker rmi -f ${IMAGE_NAME}"
                }
                echo '删除镜像完成'
            }
        }
    }
}
```

> 非root用户启动服务，这个过程会出现 `/var/run/docker.sock: connect: permission denied` 没有权限的问题，需要将Jenkins用户加入Docker用户组

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
id jenkins  # 检查输出中是否包含 "docker"
```


## 5. 基于K8S构建Jenkins持续集成平台