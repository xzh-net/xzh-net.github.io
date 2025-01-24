# Gitlab 12.4.2

GitLab是一个用于仓库管理系统的开源项目，使用Git作为代码管理工具，并在此基础上搭建起来的Web服务

## 1. 安装

### 1.1 安装依赖

```bash
yum -y install policycoreutils openssh-server openssh-clients postfix   # 安装依赖
```

### 1.2 初始化环境

```bash
systemctl enable sshd && sudo systemctl start sshd      # 启动ssh服务&设置为开机启动
systemctl enable postfix && systemctl start postfix     # 设置postfix开机自启，并启动，postfix支持gitlab发信功能
firewall-cmd --add-service=ssh --permanent              # 开放ssh以及http服务，然后重新加载防火墙列表
firewall-cmd --add-service=http --permanent
firewall-cmd --reload
```

### 1.3 下载安装

```bash
wget https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-12.4.2-ce.0.el7.x86_64.rpm --no-check-certificate
# 安装
rpm -i gitlab-ce-12.4.2-ce.0.el7.x86_64.rpm --force --nodeps
```

### 1.4 修改配置

```bash
vi /etc/gitlab/gitlab.rb
# 修改内容
external_url 'http://192.168.3.200:82'
nginx['listen_port'] = 82
```

### 1.5 启动服务

```bash
gitlab-ctl reconfigure  # 重载配置及启动gitlab
gitlab-ctl restart
```

## 2. 应用设置

### 2.1 系统登录

访问地址：http://192.168.3.200:82/ 首次进入重置root账号密码

### 2.2 添加群组

![](../../assets/_images/deploy/gitlab/create_group.png)

### 2.3 创建用户

![](../../assets/_images/deploy/gitlab/create_user.png)


![](../../assets/_images/deploy/gitlab/create_user2.png)

### 2.4 修改密码

![](../../assets/_images/deploy/gitlab/update_user.png)


### 2.5 用户添加到群组中

![](../../assets/_images/deploy/gitlab/group_add_user.png)


### 2.6 创建项目

![](../../assets/_images/deploy/gitlab/create_project.png)

### 2.7 设置SSH免密

1. 本地生成新的密钥

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

2. 添加SSH公钥到GitLab

![](../../assets/_images/deploy/gitlab/add_ssh.png)

> 仓库地址必须配置域名，IP下测试未通过。原因配置待查。


### 2.8 设置HTTP免密

免密实现基于文件存储的持久化方式，将用户名和密码存储在文件中，然后在每次执行git操作时，自动读取文件中的凭证信息。

```bash
#!/bin/bash

# 从环境变量中获取Git用户名和密码
username="${GIT_USERNAME}"
password="${GIT_PASSWORD}"

# 设置Git凭证存储路径
credential_file="$HOME/.git-credentials"

# 将用户名和密码写入凭证文件
echo "https://${username}:${password}@github.com" > $credential_file

# 配置Git使用凭证存储
git config --global credential.helper store
```


```bash
export GIT_USERNAME="your_username"
export GIT_PASSWORD="your_password"
```

> 用户名和密码必须进行编码，否则会报错

## 3. 客户端

### 3.1 安装

下载地址：https://github.com/git-for-windows/git/releases/download/v2.45.2.windows.1/Git-2.45.2-64-bit.exe
        

### 3.2 下载项目

1. 复制连接

![](../../assets/_images/deploy/gitlab/project_clone.png)

2. 打开命令行

![](../../assets/_images/deploy/gitlab/gitlab_base_cmd.png)

3. 设置用户名密码

![](../../assets/_images/deploy/gitlab/gitlab_auth.png)


### 3.3 提交代码

```bash
git add .                     # 提交文件到暂存区
git rm README.md              # 删除文件
git commit -m "remark"        # 提交代码以...为注释
git push                      # 推送到远程仓库
git pull                      # 拉取代码

# 克隆远程仓库的dev分支到本地
git clone --branch dev http://192.168.3.200:82/xzh-group/xzh-spring-boot.git  
git checkout -b develop_xzh     # 创建新分支，并把当前分支内容复制到新分支中
git push origin develop_xzh     # 将新分支推送到远程仓库
```

### 3.4 常用命令

```bash
git config --global user.name "xuzhihao"
git config --global user.email "xuzhihao@163.com"
git config --global credential.helper store     # 持久化

git config --global --list      # 查看全局配置
git config --global --edit      # 编辑全局配置，windows按下 Win + R 键，然后输入 control keymgr.dll 来打开凭据管理器

git branch                      # 查看分支
git status                      # 查看本地仓库文件状况
git log                         # 查看提交记录

# 统计特定时间段内、由指定作者所做的代码更改的统计数据
git log --since='2024-11-14 09:00:00' --until='2024-11-14 23:59:59'  --author="xuzhihao"  --pretty=tformat: --numstat | awk '{ add += $1; subs += $2; loc += $1 - $2 } END { printf "增加数: %s, 删除的行数: %s, 净增加行数: %s\n", add, subs, loc }'
```