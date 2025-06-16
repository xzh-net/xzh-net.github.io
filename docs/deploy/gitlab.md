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

1. 修改IP地址和端口

```bash
vi /etc/gitlab/gitlab.rb
# 修改内容
external_url 'http://192.168.3.200'
nginx['listen_port'] = 80
```

2. 域名访问（可选）

开启SSL，程序自动监听443端口

```bash
vi /etc/gitlab/gitlab.rb
# 修改内容
external_url 'https://git.xuzhihao.net'
nginx['ssl_certificate'] = "/etc/gitlab/ssl/git.xuzhihao.net.crt"
nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/git.xuzhihao.net.key"
```

如果使用Nginx做为一级代理，只需要将域名映射到本机的443端口即可

```conf
server {
    listen 80;
    listen [::]:80;
    server_name git.xuzhihao.net;
    charset utf-8;

    location / {
      proxy_pass https://192.168.3.200/;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```


### 1.5 启动服务

```bash
gitlab-ctl reconfigure  # 每次修改配置需要执行
gitlab-ctl restart
```

## 2. 应用设置

### 2.1 系统登录

访问地址：http://192.168.3.200 ，首次进入重置root账号密码

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

### 2.7 设置SSH密钥

1. 本地生成新的密钥

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

2. 添加SSH公钥到GitLab

![](../../assets/_images/deploy/gitlab/add_ssh.png)

> 仓库地址必须配置域名，IP下测试未通过。原因配置待查。

### 2.8 设置GPG密钥

1. 客户端生成密钥

```bash
# 安装GPG
sudo apt install gpg

# 生成密钥
gpg --full-generate-key
# 1. 在提示时，指定要生成的密钥类型，或按 Enter 键接受默认的 RSA and RSA。
# 2. 输入所需的密钥长度。 密钥必须至少是 4096 位。
# 3. 输入密钥的有效时长。 按 Enter 键将指定默认选择，表示该密钥不会过期。
# 4. 验证您的选择是否正确。
# 5. 输入您的用户 ID 信息。注：要求您输入电子邮件地址时，请确保输入您 GitHub 帐户的经验证电子邮件地址。
# 6. 输入安全密码。
# 7. 再次确认输入安全密码。

# 列出您拥有其公钥和私钥的长形式 GPG 密钥
gpg --list-secret-keys --keyid-format LONG <your_email>
# 查看公钥
gpg --armor --export 8108F798EFF7AFD8
# 删除密钥（可选）
gpg --delete-secret-keys 8108F798EFF7AFD8
```

密钥格式示例：
```lua
sec   ed25519/8108F798EFF7AFD8 2025-01-24 [SC]
      5A1E8611606C4AA765F991E38108F798EFF7AFD8
uid                 [ultimate] xcg <xcg@163.com>
ssb   cv25519/CF7625F4DE027403 2025-01-24 [E]
```

2. 添加GPG公钥到GitLab

![](../../assets/_images/deploy/gitlab/add_gpg.png)


3. 客户端配置Key

```bash
# 设置提交代码时使用的Key
git config --global user.signingkey 8108F798EFF7AFD8
# 默认情况下，运行此命令可签署所有git提交
git config --global commit.gpgsign true
```

4. 设置成功后，提交记录显示已验证

![](../../assets/_images/deploy/gitlab/gpg.png)



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

4. HTTP免密（可选）

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


### 3.3 提交代码

```bash
git add .                     # 提交文件到暂存区
git rm README.md              # 删除文件
git commit -m "remark"        # 提交代码以...为注释
git push                      # 推送到远程仓库
git pull                      # 拉取代码

# 克隆远程仓库的dev分支到本地
git clone --branch dev http://192.168.3.200/xzh-group/xzh-spring-boot.git  
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