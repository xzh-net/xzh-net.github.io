# Twemproxy 0.5.0

Twemproxy是Twitter开源的一个redis代理proxy，通过引入一个代理层隐藏后端服务，同时避免单点问题。支持失败节点自动删除、设置HashTag、状态监控。

- 下载地址：https://github.com/twitter/twemproxy/

## 1. 安装依赖

### 1.1 编译autoconf

```bash
cd /opt/software
wget http://ftp.gnu.org/gnu/autoconf/autoconf-2.69.tar.gz
tar zxvf autoconf-2.69.tar.gz
cd autoconf-2.69
./configure --prefix=/usr/local
make && make install
autoconf --version
```

> 编译过程提示报错 `checking for GNU M4 that supports accurate traces... configure: error: no acceptable m4 could be found in $PATH.` 需要安装m4

```bash
cd /opt/software
wget http://mirrors.kernel.org/gnu/m4/m4-1.4.18.tar.gz
tar -xvf m4-1.4.18.tar.gz
cd m4-1.4.18
./configure
make && make install
m4 --version
```

> 编译过程提示报错 `BEGIN failed--compilation aborted at ../lib/Autom4te/C4che.pm line 33` 需要安装perl的依赖包 `perl-Data-Dumper`

包地址：http://rpmfind.net/linux/rpm2html/search.php?query=perl-Data-Dumper(x86-64)

```bash
cd /opt/software
wget http://rpmfind.net/linux/centos/7.9.2009/os/x86_64/Packages/perl-Data-Dumper-2.145-3.el7.x86_64.rpm
rpm -ivh perl-Data-Dumper-2.145-3.el7.x86_64.rpm
perl -v
```

### 1.2 编译automake

```bash
cd /opt/software
wget http://ftp.gnu.org/gnu/automake/automake-1.15.tar.gz
tar -zvxf automake-1.15.tar.gz
cd automake-1.15
./configure
make && make install
aclocal --version
```

> 编译过程提示报错 `Try '--no-discard-stderr' if option outputs to stderr` 修改源码 `automake-1.15/Makefile.in`

```bash
# 找到下面这段
doc/aclocal-$(APIVERSION).1: $(aclocal_script) lib/Automake/Config.pm
        $(update_mans) aclocal-$(APIVERSION)
doc/automake-$(APIVERSION).1: $(automake_script) lib/Automake/Config.pm
        $(update_mans) automake-$(APIVERSION)
# 添加--no-discard-stderr选项
doc/automake-$(APIVERSION).1: $(automake_script) lib/Automake/Config.pm
        $(update_mans) automake-$(APIVERSION) --no-discard-stderr
```

### 1.3 编译libtool

```bash
cd /opt/software
wget https://ftp.gnu.org/gnu/libtool/libtool-2.4.6.tar.gz
tar -zvxf libtool-2.4.6.tar.gz
cd libtool-2.4.6
./configure
make && make install
libtool --version
```

## 2. 安装Twemproxy

```bash
cd /opt/software
wget https://github.com/twitter/twemproxy/releases/download/0.5.0/twemproxy-0.5.0.tar.gz
tar -zvxf twemproxy-0.5.0.tar.gz
cd twemproxy-0.5.0
autoreconf -fvi 
./configure --prefix=/usr/local/twemproxy/
make && make install
```

> 编译过程提示报错 `Can't locate Thread/Queue.pm in @INC`，需要安装依赖`yum install perl-Thread-Queue`

### 2.1 修改配置


```bash
cd /opt/software/twemproxy-0.5.0
cp -r ./conf /usr/local/twemproxy/
cd /usr/local/twemproxy/conf
vi nutcracker.yml
```

```yml
alpha:
  listen: 127.0.0.1:22121
  hash: fnv1a_64
  distribution: ketama
  auto_eject_hosts: true
  redis: true
  redis_auth: 123456
  server_retry_timeout: 2000
  server_failure_limit: 1
  servers:
   - 192.168.2.201:6379:1
```

### 2.2 启动服务

设置环境变量

```bash
echo "PATH=$PATH:/usr/local/twemproxy/sbin/" >> /etc/profile
source /etc/profile
```

启动服务

```bash
/usr/local/twemproxy/sbin/nutcracker
```

## 3. 客户端测试

```bash
redis-cli -h 192.168.2.201 -p 22121 -a 123456
```

