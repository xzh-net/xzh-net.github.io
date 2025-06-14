
# 基于Nexus3快速搭建Maven私有仓库

> 为什么要搭建私有仓库

![](../../assets/_images/deploy/nexus3/maven.png)

通常都是通过本机的Maven直接访问到中央仓库，并没有使用到虚线标识的区域，直接使用中央仓库可能会给我们带来的问题

- `网络问题`：中央仓库在国外，需要科学上网，私服搭建好以后，本地开发环境优先连接私服，当私服没有的资源的时候去中央仓库下载，只需要私服从中央仓库下载一次即可
- `私有项目的管理`：企业内部资源，如封装的类库，框架等更新直接上传到私服即可
- `三方未上传到中央仓库的类库`：如：oracle.jar，paoding-analysis等，直接通过本地环境直接上传到私服，项目组所有成员可共享

默认仓库
   
- `maven-central`：maven中央库，默认`https://repo1.maven.org/maven2`
- `maven-releases`：私库发行版，首次安装请将Deployment policy设置为Allow redeploy
- `maven-snapshots`：私库快照版本
- `maven-public`：仓库组

Nexus仓库类型

- `hosted`：本地仓库，通常我们会部署自己的构件到这一类型的仓库。比如公司的第二方库。
- `proxy`：代理仓库，它们被用来代理远程的公共仓库，如maven中央仓库。
- `group`：仓库组，用来合并多个hosted/proxy仓库，当你的项目希望在多个repository使用资源时就不需要多次引用了，只需要引用一个group即可。


![](../../assets/_images/deploy/nexus3/repository_list.png)

## 1. 安装

```bash
docker pull sonatype/nexus3:3.38.0
docker volume create --name nexus-data
docker run -d -p 8081:8081 --name nexus -v nexus-data:/nexus-data sonatype/nexus3:3.38.0
```

使用默认管理员身份登录，帐号：**admin**，密码：`cat /nexus-data/admin.password`


## 2. 服务端

### 2.1 创建存储

文件类型选择`File`，路径选择`/nexus-data`

![](../../assets/_images/deploy/nexus3/create_store1.png)

![](../../assets/_images/deploy/nexus3/create_store2.png)

![](../../assets/_images/deploy/nexus3/create_store3.png)

### 2.2 创建三方混合库

![](../../assets/_images/deploy/nexus3/create_hosted1.png)

![](../../assets/_images/deploy/nexus3/create_hosted2.png)

![](../../assets/_images/deploy/nexus3/create_hosted3.png)

### 2.3 创建二方发布库

![](../../assets/_images/deploy/nexus3/create_hosted4.png)

### 2.4 创建二方快照库

![](../../assets/_images/deploy/nexus3/create_hosted5.png)

### 2.5 创建代理仓库

![](../../assets/_images/deploy/nexus3/create_proxy1.png)

![](../../assets/_images/deploy/nexus3/create_proxy2.png)

> 阿里云的maven中央仓库地址：http://maven.aliyun.com/nexus/content/groups/public/

### 2.6 创建仓库组

![](../../assets/_images/deploy/nexus3/create_group1.png)

![](../../assets/_images/deploy/nexus3/create_group2.png)

Hosted安全策略:

`Allow Redeploy` 允许重新部署应用程序。开发人员可以对应用程序进行修改、更改配置或更新代码，并将这些更改重新部署到运行中的应用程序中。重新部署通常在开发和测试环境中使用，以验证和应用修改的效果。

`Disable Redeploy` 禁用重新部署功能，阻止在应用程序运行时修改代码或更改配置。这通常用于生产环境，以确保系统的稳定性和安全性。禁用重新部署功能可以减少意外错误的风险，并提高应用程序的性能和安全性。

`Read-only` 只读模式，即应用程序或系统处于一种状态，其中任何用户或进程都无法对其进行修改或写入操作。在只读模式下，数据和配置文件是不可编辑的，只能进行读取或查询。

`Deploy by Replication Only` 仅通过复制机制进行部署。在这种部署方式下，多个节点具有相同的应用程序代码、配置和数据副本，每个节点都可以独立地提供服务。当一个节点失效时，其他节点可以接管其功能，以确保系统的可用性和稳定性。

### 2.7 设置代理（可选）

当服务器部署在内网的时候，如果需要访问外网，可以依赖nginx正向代理，将请求转发到外网，再由外网访问maven中央仓库，最后将结果返回给服务器。

![](../../assets/_images/deploy/nexus3/http_proxy.png)

## 3. 客户端

### 3.1 修改配置

修改Maven的settings.xml文件，加入认证机制

```xml
<!--nexus服务器,id为组仓库name-->
  <servers>  
    <server>  
        <id>xzh-group</id>  
        <username>admin</username>  
        <password>admin</password>  
    </server>
    <server>  
        <id>xzh-hosted</id>  
        <username>admin</username>  
        <password>admin</password>  
    </server>
    <server>  
        <id>xzh-snapshots</id>  
        <username>admin</username>  
        <password>admin</password>  
    </server>
    <server>  
        <id>xzh-releases</id>  
        <username>admin</username>  
        <password>admin</password>  
    </server>   
  </servers>  
<!--仓库组的url地址，id和name可以写组仓库name，mirrorOf的值设置为central-->  
  <mirrors>     
    <mirror>  
        <id>xzh-group</id>  
        <name>xzh-group</name>  
        <url>http://172.17.17.200:8081/repository/xzh-group/</url>  
        <mirrorOf>central</mirrorOf>  
    </mirror>     
  </mirrors>
```

### 3.2 手动上传

```bash
mvn install:install-file -Dfile=D:\paoding-analysis.jar -DgroupId=net.paoding -DartifactId=paoding-analysis -Dversion=1.0 -Dpackaging=jar -DgeneratePom=true -DcreateChecksum=true 
mvn install:install-file -Dfile=D:\ojdbc6.jar -DgroupId=com.oracle -DartifactId=ojdbc6 -Dversion=10.2.0.5.0 -Dpackaging=jar -DgeneratePom=true -DcreateChecksum=true  
mvn install:install-file -Dfile=D:/iTextAsian.jar -DgroupId=com.lowagie -DartifactId=itextasian -Dversion=1.0 -Dpackaging=jar 
mvn install:install-file -Dfile=D:/tools.jar -DgroupId=com.sun2 -DartifactId=tools -Dversion=1.6.0 -Dpackaging=jar
# 指定远程仓库地址
mvn deploy:deploy-file -DgroupId=net.xzh -DartifactId=spring-boot-email -Dversion=2.3.0.RELEASE -Dpackaging=jar -Dfile=spring-boot-email-2.3.0.RELEASE.jar -Durl=http://172.17.17.200:8081/repository/xzh-hosted/ -DrepositoryId=xzh-hosted
```

参数说明
- -DgroupId：jar的groupId
- -DartifactId：jar的artifactId
- -Dversion：jar的版本
- -Dpackaging：指定包为jar
- -Dfile：文件地址，可以设置绝对路径，由于我cmd进入了对应的目录，下面的指令使用的相对路径
- -Durl：本地仓库的地址
- -DgeneratePom：是否生成pom文件，ture:生成，false：不生成
- -DrepositoryId：仓库名称


### 2.9 开发工具上传

使用IntelliJ IDEA上传至私有库，在pom文件添加本地仓库的配置

```xml
<!-- 配置资源上传的仓库地址，与settings.xml中  -->
<distributionManagement>
    <repository>
        <id>xzh-release</id>
        <url>http://172.17.17.200:8081/repository/xzh-release/</url>
    </repository>
    <snapshotRepository>
        <id>xzh-snapshots</id>
        <url>http://172.17.17.200:8081/repository/xzh-snapshots/</url>
    </snapshotRepository>
</distributionManagement>
```

snapshots和releases的区别

snapshot：每次都会从服务器拉取一个最新的版本使用，不管本地是否存在，都会强制刷新，但是刷新并不意味着把jar重新下载一遍。只下载几个比较小的文件，通过这几个小文件确定本地和远程仓库的版本是否一致，再决定是否下载

releases：当检测到本地的maven仓库有相应的版本之后，不会去服务器拉取，就直接使用本地的版本了。