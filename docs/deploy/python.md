# Ptyhon 3.12.2

## 1. Windows

### 1.1 安装Ptyhon

下载地址：https://www.python.org/downloads

![](../../assets/_images/deploy/python/1.png)

![](../../assets/_images/deploy/python/2.png)

![](../../assets/_images/deploy/python/3.png)

![](../../assets/_images/deploy/python/4.png)

![](../../assets/_images/deploy/python/5.png)

![](../../assets/_images/deploy/python/6.png)

### 1.2 安装PyCharm

下载地址：https://www.jetbrains.com/zh-cn/pycharm/

![](../../assets/_images/deploy/python/10.png)

![](../../assets/_images/deploy/python/11.png)

![](../../assets/_images/deploy/python/12.png)

![](../../assets/_images/deploy/python/13.png)

![](../../assets/_images/deploy/python/14.png)

启动

![](../../assets/_images/deploy/python/20.png)

创建项目

![](../../assets/_images/deploy/python/21.png)

设置主题

![](../../assets/_images/deploy/python/22.png)

设置字体

![](../../assets/_images/deploy/python/23.png)

### 1.3 安装Anaconda3

下载地址：https://mirrors.tuna.tsinghua.edu.cn/anaconda/archive/Anaconda3-2021.05-Windows-x86_64.exe


## 2. Linux

### 2.1 下载安装

1. 安装依赖

```bash
yum install wget zlib-devel bzip2-devel openssl openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gcc make zlib zlib-devel libffi-devel -y
```

2. 下载编译

[Python requires a OpenSSL 1.1.1 or newer](deploy/haproxy?id=_15-安装openssl)
```bash
cd /opt/software
wget https://www.python.org/ftp/python/3.12.2/Python-3.12.2.tgz
tar -xvf Python-3.12.2.tgz
cd Python-3.12.2
# 配置
./configure --prefix=/usr/local/python3.12.2 --with-openssl=/usr/local/openssl
# 编译
make && make install
```

3. 设置环境变量

```bash
vi /etc/profile
```

```conf
export PYTHON_HOME=/usr/local/python3.12.2
export PATH=$PYTHON_HOME/bin:$PATH
```

使生效
```bash
source /etc/profile
```

4. 创建软连接

```bash
rm -f /usr/bin/python
ln -s /usr/local/python3.12.2/bin/python3.12 /usr/bin/python
# 创建软链接后，会破坏yum程序的正常使用，需要修改以下两个文件
/usr/bin/yum
/usr/libexec/urlgrabber-ext-down
```

使用vi编辑器，将这2个文件的第一行，从
```conf
#!/usr/bin/python
```
修改为：
```conf
#!/usr/bin/python2
```

### 2.2 虚拟环境

1. 配置pip3源

```bash
mkdir ~/.pip
touch ~/.pip/pip.conf
vi ~/.pip/pip.conf
```

添加配置

```conf
[global]
index-url=http://mirrors.aliyun.com/pypi/simple
[install]
trusted-host=mirrors.aliyun.com
```

2. 安装虚拟环境

```bash
pip3 install virtualenv
```

3. 创建虚拟环境

```bash
mkdir /data/python3/code -p
cd /data/python3/code
# 创建环境
virtualenv --python=python3 jmp_venv1
# 激活环境
source /data/python3/code/jmp_venv1/bin/activate
# 退出环境
deactivate
```
