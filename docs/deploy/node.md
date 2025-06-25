
#  Node.js 14.16.0

- 网站地址：https://nodejs.org/
- 下载地址：https://nodejs.org/download/release/
- Npm仓库: https://www.npmjs.com/

## 1. Windows

### 1.1 安装Node.js

![](../../assets/_images/deploy/node/1.png)

![](../../assets/_images/deploy/node/2.png)

![](../../assets/_images/deploy/node/3.png)

![](../../assets/_images/deploy/node/4.png)

![](../../assets/_images/deploy/node/5.png)

![](../../assets/_images/deploy/node/6.png)

![](../../assets/_images/deploy/node/7.png)

![](../../assets/_images/deploy/node/8.png)


### 1.2 常用命令

```bash
npm -v                          # 查看npm安装的版本
npm get global                  # 查看当前使用的安装模式
npm set global=true             # 设定全局安装模式

npm config get registry         # 查看仓库地址
npm config set registry http://mirrors.cloud.tencent.com/npm/     # 设定全局仓库地址
npm cache clean --force

npm init                        # 快速创建package.json
npm i erpress                   # 安装erpress模块
npm i express@4.17.1            # 安装指定版本模块

npm i –save                     # 将模块写入dependencies节点（生产环境）
npm i –save-dev                 # 将模块写入devDependencies节点（开发环境）
npm outdated                    # 检查包是否已经过时
npm update express              # 更新模块
npm uninstall express           # 卸载模块

npm root                        # 查看当前包的安装路径
npm root -g                     # 查看全局的包的安装路径
npm list                        # 查看当前目录下已安装的node包
npm list parseable=true         # 以目录的形式来展现当前安装的所有node包
```

> 如果安装过程中依赖文件因为网络问题下载失败，可以手动下载文件后上传至缓存路径中，然后再次使用`npm install`安装。 

```bash
npm config get cache                    # 查看缓存路径
C:\Users\CR7\AppData\Local\npm-cache    # Windows10 下缓存路径
https://github.com/lovell/sharp-libvips/releases/download/v8.14.5/libvips-8.14.5-win32-x64.tar.br
```



### 1.3 模块

#### 1.3.1 nodemon

```bash
npm i nodemon -g            # 安装nodemon模块
```

#### 1.3.2 forever

```bash
npm i forever -g            # 安装forever模块
forever start app.js        # 启动进程
forever stop  app.js        # 关闭进程
forever stopall             # 关闭所有进程
forever restart app.js      # 重启进程
forever list                # 查看服务进程
forever start -w app.js     # 监听文件改动
forever start -l forever.log -o out.log -e err.log app.js   # 日志输出
```

## 2. Linux

### 2.1 安装Node.js

#### 2.1.1 下载解压

```bash
cd /opt/software
wget https://nodejs.org/dist/v14.16.0/node-v14.16.0-linux-x64.tar.xz
tar -xvf node-v14.16.0-linux-x64.tar.xz
mv node-v14.16.0-linux-x64 /usr/local/node
```

#### 2.1.2 设置环境变量

```bash
vim /etc/profile
```

```bash
export NODE_HOME=/usr/local/node
export PATH=$NODE_HOME/bin:$PATH
# 使配置生效
source /etc/profile     
```

#### 2.1.3 环境验证

```bash
node -v
npm -v
```



## 3. Express

### 3.1 Hello World

1. 安装依赖

```bash
npm init -y
npm i express@4.17.1 --registry=https://registry.npmmirror.com
```

2. 服务端

创建并编辑启动文件`app.js`

```js
const express = require('express')
const app = express()

// 定义一个全局中间件
app.use((req, res, next) => {
  console.log('这是1个全局中间件')
  next()
})

// 定义一个路由
app.get('/', (req, res) => {
  res.send('欢迎来到xzh的主页')
})

app.listen(80, () => {
  console.log('http://127.0.0.1')
})
```

然后运行 `node app.js` ，并在浏览器输入 `http://localhost/` 即可看到页面效果。