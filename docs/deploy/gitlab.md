# Gitlab 12.4.2

GitLab是一个用于仓库管理系统的开源项目，使用Git作为代码管理工具，并在此基础上搭建起来的Web服务

## 1. 服务端部署

### 1.1 安装系统依赖

```bash
yum -y install policycoreutils openssh-server openssh-clients postfix   # 安装依赖
```

### 1.2 配置基础环境

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

### 1.4 配置 GitLab

修改IP地址和端口

```bash
vi /etc/gitlab/gitlab.rb
# 修改内容
external_url 'http://192.168.3.200'
nginx['listen_port'] = 80
```

域名访问，开启SSL，程序自动监听443端口（可选）

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


### 1.5 启动并应用配置

```bash
gitlab-ctl reconfigure  # 每次修改配置需要执行
gitlab-ctl restart
```

## 2. GitLab 初始设置

### 2.1 首次访问与 root 密码重置

访问地址：http://192.168.3.200

### 2.2 创建群组

![](../../assets/_images/deploy/gitlab/create_group.png)

### 2.3 创建用户并设置密码

![](../../assets/_images/deploy/gitlab/create_user.png)

![](../../assets/_images/deploy/gitlab/create_user2.png)

![](../../assets/_images/deploy/gitlab/update_user.png)


### 2.4 将用户加入群组

![](../../assets/_images/deploy/gitlab/group_add_user.png)


### 2.5 创建项目

![](../../assets/_images/deploy/gitlab/create_project.png)

### 2.6 配置 SSH 密钥（免密推送）

客户端生成密钥

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

添加 SSH 公钥到 GitLab 

![](../../assets/_images/deploy/gitlab/add_ssh.png)

!> 仓库地址必须配置域名，IP下测试未通过。原因配置待查。

### 2.7 配置 GPG 签名（提交验证）

客户端生成密钥

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

添加 GPG 公钥到 GitLab

![](../../assets/_images/deploy/gitlab/add_gpg.png)


客户端配置 Key

```bash
# 设置提交代码时使用的Key
git config --global user.signingkey 8108F798EFF7AFD8
# 默认情况下，运行此命令可签署所有git提交
git config --global commit.gpgsign true
```

设置成功后，提交记录显示已验证

![](../../assets/_images/deploy/gitlab/gpg.png)


## 3. 客户端使用

### 3.1 安装 Git for Windows

- 下载地址1：https://github.com/git-for-windows/git/releases/download/v2.45.2.windows.1/Git-2.45.2-64-bit.exe
- 下载地址2：https://git-scm.com/install/windows        

### 3.2 克隆项目

复制连接

![](../../assets/_images/deploy/gitlab/project_clone.png)

打开命令行

![](../../assets/_images/deploy/gitlab/gitlab_base_cmd.png)

设置用户名密码

![](../../assets/_images/deploy/gitlab/gitlab_auth.png)

### 3.3 VS Code 配置（可选）

!> 在使用 VS Code的 Git 功能前，需要先确保电脑上正确安装了Git客户端。

打开 VS Code，点击左侧活动栏的 "源代码管理"（Source Control）图标（一个类似小分支的图案）。如果界面正常，说明 Git 已被正确识别。

打开一个新终端（Terminal > New Terminal），输入以下命令配置全局用户名和邮箱

```bash
git config --global user.name "xuzhihao"
git config --global user.email "xuzhihao@163.com"
```

按下快捷键 Ctrl + Shift + P (Windows/Linux) 或 Cmd + Shift + P (macOS) 调出命令面板，输入 Git: Clone 并选择该命令。在弹出的输入框中，粘贴你刚才从GitLab复制的项目地址，然后按回车。

VS Code 会弹出文件夹选择窗口，选择你想把项目存放在哪个 "父文件夹" 里，然后点击 "选择仓库位置"。注意：不需要提前手动创建一个同名的空文件夹，VS Code 克隆时会自动生成。

- 创建分支：点击 VS Code左下角的分支名（如"main"），选择"创建新分支"，输入比如"dev0430"回车即可自动切换
- 暂存更改：所有变更会出现在"源代码管理"面板的"更改"列表里。编辑完，点击文件旁的 + 号将其加入暂存区
- 提交到本地仓库：在消息框中清晰描述变更内容（如"完成登录页面UI"），点击 ✓ 号提交（Commit）

### 3.4 HTTP 免密（可选）

免密实现基于文件存储的持久化方式，将用户名和密码存储在文件中，然后在每次执行 Git 操作时，自动读取文件中的凭证信息。

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

!> 用户名和密码必须进行编码，否则会报错


### 3.5 常用命令

提交代码

```bash
git branch                      # 查看所有本地分支
git status                      # 查看本地仓库文件状况
git log                         # 查看提交记录

# 克隆远程仓库的dev分支到本地
git clone --branch dev http://192.168.3.200/xzh-group/xzh-spring-boot.git

git branch new-feature          # 已当前分支创建一个新本地分支
git switch new-feature          # 切换本地分支
git push --set-upstream origin new-feature      # 将新分支推送到远程仓库

git add .                       # 提交文件到暂存区
git rm README.md                # 删除文件
git commit -m "remark"          # 提交代码以...为注释
git push                        # 推送到远程仓库
git pull                        # 拉取代码

git merge new-feature           # 将指定分支合并到当前分支  
git branch -d new-feature       # 删除一个已经合并的本地分支
git branch -D new-feature       # 强制删除一个本地分支（未合并的）	
git push origin --delete new-feature    # 删除远端分支
```

全局

```bash
git config --global user.name "xuzhihao"
git config --global user.email "xuzhihao@163.com"
git config --global credential.helper store     # 持久化
git config --global push.autoSetupRemote true   # 自动设置远程仓库

git config --global --list      # 查看全局配置
git config --global --edit      # 编辑全局配置，windows按下 Win + R 键，然后输入 control keymgr.dll 来打开凭据管理器
```

### 3.6 代码统计

```bash
# 统计特定时间段内、由指定作者所做的代码更改的统计数据
git log --since='2024-11-14 09:00:00' --until='2024-11-14 23:59:59'  --author="xuzhihao"  --pretty=tformat: --numstat | awk '{ add += $1; subs += $2; loc += $1 - $2 } END { printf "增加数: %s, 删除的行数: %s, 净增加行数: %s\n", add, subs, loc }'
```