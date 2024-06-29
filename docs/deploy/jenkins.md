# Jenkins 2.426 LTS

Jenkins是一个开源软件项目，是基于Java开发的一种持续集成工具，用于监控持续重复的工作，旨在提供一个开放易用的软件平台，使软件项目可以进行持续集成

下载地址：https://get.jenkins.io/redhat-stable/

## 1. 安装

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

#### 1.2.2 修改配置

```bash
vim /usr/lib/systemd/system/jenkins.service
```

设置启动用户组

```conf
User=root
Group=root
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

![](../../assets/_images/deploy/jenkins/jenkins_plugin_chinese.png)

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

#### 1.4.5 设置授权策略

授权策略切换为`Role-Based Strategy`

![](../../assets/_images/deploy/jenkins/jenkins_plugin_set2.png)

#### 1.4.6 创建角色

![](../../assets/_images/deploy/jenkins/jenkins_create_role.png)

![](../../assets/_images/deploy/jenkins/jenkins_create_role2.png)

![](../../assets/_images/deploy/jenkins/jenkins_role1.png)

添加以下三个角色：
- baseRole：该角色为全局角色。这个角色需要绑定Overall下面的Read权限，是为了给所有用户绑定最基本的Jenkins访问权限。注意：如果不给后续用户绑定这个角色，会报错误：用户名 is
missing the Overall/Read permission
- role1：该角色为项目角色。使用正则表达式绑定"xuzhihao.*"，意思是只能操作xuzhihao开头的项目。
- role2：该角色也为项目角色。绑定"xzh.*"，意思是只能操作xzh开头的项目。

![](../../assets/_images/deploy/jenkins/jenkins_role2.png)

#### 1.4.7 创建用户

![](../../assets/_images/deploy/jenkins/jenkins_create_user.png)

![](../../assets/_images/deploy/jenkins/jenkins_create_user2.png)

#### 1.4.8 用户角色绑定

系统管理页面进入Manage and Assign Roles，点击Assign Roles，绑定规则如下：

![](../../assets/_images/deploy/jenkins/jenkins_create_user3.png)

#### 1.4.9 创建项目测试权限

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

为了让Jenkins支持从Gitlab拉取源码，需要安装Git插件以及在CentOS7上安装Git工具。

![](../../assets/_images/deploy/jenkins/jenkins_plugin_git.png)

CentOS7上安装Git工具

```bash
yum install git -y  # 安装
git --version       # 查看版本
```

#### 1.5.3 用户密码类型凭证

1. 创建凭证

![](../../assets/_images/deploy/jenkins/jenkins_plugin_git2.png)

选择"Username with password"，输入Gitlab的用户名和密码，点击"确定"

![](../../assets/_images/deploy/jenkins/jenkins_plugin_git3.png)

2. 测试凭证是否可用

创建一个FreeStyle项目：新建Item->FreeStyle Project->确定

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

![](../../assets/_images/deploy/jenkins/jenkins_git_ssh_test.png)

如果Jenkins新建任务时，输入远程仓库地址设置，选择凭证后一直提示连不上。提示：jenkins stderr: No ECDSA host key is known for gitee.com and you have requested strict checking. Host key verification failed。需要在jenkins的主机执行一下命令访问git上的仓库地址，把git的主机添加到/root/.ssh/known_hosts（执行命令前known_hosts这个文件是不存在的，执行后就有了）。

```bash
git ls-remote -h git@git.xxx.cn:root/website.git HEAD
```

### 1.6 Maven安装和配置

#### 1.6.1 Maven安装

上传Maven上传到持续集成服务器172.17.17.200

```bash
cd /opt/software
tar -zxf apache-maven-3.6.3-bin.tar.gz -C /opt # 解压

vi /etc/profile
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk
export MAVEN_HOME=/opt/apache-maven-3.6.3
export PATH=$PATH:$JAVA_HOME/bin:$MAVEN_HOME/bin

source /etc/profile                   # 配置生效
mvn -v                                # 查找Maven版本
```

#### 1.6.2 全局工具配置关联JDK和Maven

Jenkins->Global Tool Configuration->JDK->新增JDK，配置如下：

![](../../assets/_images/deploy/jenkins/jenkins_jdk.png)

Jenkins->Global Tool Configuration->Maven->新增Maven，配置如下：

![](../../assets/_images/deploy/jenkins/jenkins_maven.png)

#### 1.6.3 添加Jenkins全局变量

Manage Jenkins->Configure System->Global Properties ，添加三个全局变量JAVA_HOME、M2_HOME、PATH+EXTRA

![](../../assets/_images/deploy/jenkins/jenkins_env.png)

1. 修改Maven的settings.xml

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

#### 1.6.4 测试Maven是否配置成功

使用之前的gitlab密码测试项目，修改配置，构建->增加构建步骤->Execute Shell

![](../../assets/_images/deploy/jenkins/jenkins_mvn_build.png)

![](../../assets/_images/deploy/jenkins/jenkins_mvn_build2.png)

> 输入命令：mvn clean package


### 1.7 Tomcat安装和配置

#### 1.7.1 安装Tomcat8.5

上传tomcat上传到应用服务器172.17.17.196

```bash
yum install java-1.8.0-openjdk* -y              # 安装JDK（已完成）
cd /opt/software/
tar -zxf apache-tomcat-8.5.100.tar.gz -C /opt    # 解压
/opt/apache-tomcat-8.5.100/bin/startup.sh        # 启动
```

> 地址为：http://172.17.17.196/8080

![](../../assets/_images/deploy/jenkins/jenkins_tomcat.png)

默认情况下Tomcat是没有配置用户角色权限的，后续Jenkins部署项目到Tomcat服务器，需要用到Tomcat的用户，所以修改tomcat以下配置，添加用户及权限

```bash
vi /opt/apache-tomcat-8.5.100/conf/tomcat-users.xml

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

用户和密码都是：tomcat，为了能够刚才配置的用户登录到Tomcat，还需要修改以下配置
```bash
vi /opt/apache-tomcat-8.5.100/webapps/manager/META-INF/context.xml

<!--
<Valve className="org.apache.catalina.valves.RemoteAddrValve"
allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />
-->
```

把上面这行注释掉，重启

> 访问： http://192.168.66.102:8080/manager/html ，输入tomcat和tomcat，看到以下页面代表成功

![](../../assets/_images/deploy/jenkins/jenkins_tomcat2.png)


## 2. 构建项目

Jenkins中自动构建项目的类型有很多，常用的有以下三种：
- 自由风格软件项目（FreeStyle Project）
- Maven项目（Maven Project）
- 流水线项目（Pipeline Project）

推荐使用流水线类型，因为灵活度非常高

### 2.1 自由风格项目构建

#### 2.1.1 拉取代码

![](../../assets/_images/deploy/jenkins/jenkins_create_project.png)

![](../../assets/_images/deploy/jenkins/jenkins_free_build.png)

```bash
echo "开始编译和打包"
mvn clean package
echo "编译和打包结束"
```

#### 2.1.2 部署

1. 安装 Deploy to container插件

Jenkins本身无法实现远程部署到Tomcat的功能，需要安装Deploy to container插件实现

![](../../assets/_images/deploy/jenkins/jenkins_plugin_deploy.png)

2. 添加Tomcat用户凭证

![](../../assets/_images/deploy/jenkins/jenkins_tomcat_auth.png)

#### 2.1.3 添加构建后操作

![](../../assets/_images/deploy/jenkins/jenkins_tomcat_deploy.png)

![](../../assets/_images/deploy/jenkins/jenkins_tomcat_deploy2.png)

点击"Build Now"，开始构建过程

#### 2.1.4 部署成功后，访问项目

![](../../assets/_images/deploy/jenkins/jenkins_tomcat_deploy3.png)

![](../../assets/_images/deploy/jenkins/jenkins_tomcat_deploy4.png)


### 2.2 Maven项目

#### 2.2.1 安装Maven Integration插件

![](../../assets/_images/deploy/jenkins/jenkins_plugin_maven.png)

#### 2.2.2 创建Maven项目

![](../../assets/_images/deploy/jenkins/jenkins_project_maven.png)

#### 2.2.3 配置项目

拉取代码和远程部署的过程和自由风格项目一样，只是"构建"部分不同

![](../../assets/_images/deploy/jenkins/jenkins_maven_build.png)


### 2.3 流水线项目

- Pipeline 脚本是由 Groovy 语言实现的
- Pipeline 支持两种语法：Declarative(声明式)和 Scripted Pipeline(脚本式)语法
- Pipeline 也有两种创建方法：可以直接在 Jenkins 的 Web UI 界面中输入脚本；也可以通过创建一个 Jenkinsfile 脚本文件放入项目源码库中（一般我们都推荐在 Jenkins 中直接从源代码控制(SCM)中直接载入 Jenkinsfile Pipeline 这种方法）。


#### 2.3.1 安装 Pipeline 插件

![](../../assets/_images/deploy/jenkins/jenkins_plugin_pipeline.png)

![](../../assets/_images/deploy/jenkins/jenkins_project_pipeline.png)


#### 2.3.2 Pipeline语法快速入门

1. Declarative声明式-Pipeline
   - stages：代表整个流水线的所有执行阶段。通常stages只有1个，里面包含多个stage
   - stage：代表流水线中的某个阶段，可能出现n个。一般分为拉取代码，编译构建，部署等阶段。
   - steps：代表一个阶段内需要执行的逻辑。steps里面是shell脚本，git拉取代码，ssh远程发布等任意内容。

> 流水线->选择HelloWorld模板

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

![](../../assets/_images/deploy/jenkins/jenkins_pipeline_declarative.png)

点击构建，可以看到整个构建过程

2. Pipeline脚本式-Pipeline
   - Node：节点，一个 Node 就是一个 Jenkins 节点，Master 或者 Agent，是执行 Step 的具体运行环境，后续讲到Jenkins的Master-Slave架构的时候用到。
   - Stage：阶段，一个 Pipeline 可以划分为若干个 Stage，每个 Stage 代表一组操作，比如：Build、Test、Deploy，Stage 是一个逻辑分组的概念。
   - Step：步骤，Step 是最基本的操作单元，可以是打印一句话，也可以是构建一个 Docker 镜像，由各类 Jenkins 插件提供，比如命令：sh ‘make’，就相当于我们平时 shell 终端中执行 make 命令
一样。

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

![](../../assets/_images/deploy/jenkins/jenkins_pipeline_script.png)

构建结果和声明式一样！


#### 2.3.3 拉取代码

使用语法生成器

```
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

#### 2.3.4 编译打包

```
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
            sh 'mvn clean package'
         }
      }
   }
}
```

#### 2.3.5 部署

```
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
            sh 'mvn clean package'
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

#### 2.3.6 Pipeline Script from SCM

刚才我们都是直接在Jenkins的UI界面编写Pipeline代码，这样不方便脚本维护，建议把Pipeline脚本放在项目中（一起进行版本控制）,在项目根目录建立Jenkinsfile文件，把内容复制到该文件中并上传到Gitlab

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
            sh 'mvn clean package'
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

在项目中引用该文件

![](../../assets/_images/deploy/jenkins/jenkins_scm1.png)

![](../../assets/_images/deploy/jenkins/jenkins_scm2.png)


### 2.4 Jenkins项目构建参数

#### 2.4.1 常用的构建触发器

Jenkins内置4种构建触发器：
- 触发远程构建
- 其他工程构建后触发（Build after other projects are build）
- 定时构建（Build periodically）
- 轮询SCM（Poll SCM）


1.  触发远程构建

![](../../assets/_images/deploy/jenkins/jenkins_build_triger1.png)

触发构建url：http://172.17.17.200:8888/job/test_pipeline01/build?token=123456

2. 其他工程构建后触发

![](../../assets/_images/deploy/jenkins/jenkins_build_after.png)

3. 定时构建

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

4. 轮询SCM

轮询SCM，是指定时扫描本地代码仓库的代码是否有变更，如果代码有变更就触发项目构建。但是定时扫描本地整个项目的代码，增大系统的开销，不建议使用
![](../../assets/_images/deploy/jenkins/jenkins_build_scm.png)

#### 2.4.2 Git hook自动触发构建

利用Gitlab的webhook实现代码push到仓库，立即触发项目自动构建

安装插件：`Generic Webhook Trigger`和`GitLab`

![](../../assets/_images/deploy/jenkins/jenkins_plugin_gitwebhook.png)

1. Jenkins设置自动构建

![](../../assets/_images/deploy/jenkins/jenkins_project_webhook.png)

2. Gitlab配置webhook

1）开启webhook功能
使用root账户登录到后台，点击Admin Area -> Settings -> Network
勾选`Allow requests to the local network from web hooks and services`

![](../../assets/_images/deploy/jenkins/jenkins_gitlab_webhook.png)

2）在项目添加webhook
点击项目->Settings->Integrations

![](../../assets/_images/deploy/jenkins/jenkins_gitlab_webhook2.png)

> 注意：以下设置必须完成，否则会报错！Manage Jenkins->Configure System

![](../../assets/_images/deploy/jenkins/jenkins_plugin_gitwebhook2.png)



#### 2.4.3 Jenkins的参数化构建

有时在项目构建的过程中，我们需要根据用户的输入动态传入一些参数，从而影响整个构建结果，这时我们可以使用参数化构建。

Jenkins支持非常丰富的参数类型，在Jenkins添加字符串类型参数

![](../../assets/_images/deploy/jenkins/jenkins_build_param0.png)

![](../../assets/_images/deploy/jenkins/jenkins_build_param1.png)

创建分支：test，代码稍微改动下，然后提交到gitlab上。
```bash
git push -u origin master:test
```

改动pipeline流水线代码

![](../../assets/_images/deploy/jenkins/jenkins_build_param2.png)

点击Build with Parameters

![](../../assets/_images/deploy/jenkins/jenkins_build_param3.png)


#### 2.4.4 配置邮箱服务器发送构建结果

安装Email Extension Template插件

![](../../assets/_images/deploy/jenkins/jenkins_plugin_email.png)

Jenkins设置邮箱相关参数， Manage Jenkins->Configure System

![](../../assets/_images/deploy/jenkins/jenkins_mail_admin.png)

设置邮件参数，Extended E-mail Notification

![](../../assets/_images/deploy/jenkins/jenkins_mail_admin2.png)

![](../../assets/_images/deploy/jenkins/jenkins_mail_admin3.png)

![](../../assets/_images/deploy/jenkins/jenkins_mail_admin4.png)

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

编写Jenkinsfile添加构建后发送邮件

```bash
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
            sh 'mvn clean package'
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