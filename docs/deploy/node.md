
### 4.3 Node

```bash
yum install -y git
wget https://nodejs.org/dist/v14.16.0/node-v14.16.0-linux-x64.tar.xz
tar -xvf node-v14.16.0-linux-x64.tar.xz
mv node-v14.16.0-linux-x64 /usr/local/node

vim /etc/profile
export NODE_HOME=/usr/local/node
export PATH=$NODE_HOME/bin:$PATH

source /etc/profile
node -v
npm -v
```

```bash
npm install forever -g      #全局安装forever启动命令
forever start app.js        #启动进程
forever stop  app.js        #关闭进程
forever stopall             #关闭所有进程
forever restart app.js      #重启进程
forever list                #查看服务进程
forever start -w app.js     #监听文件改动
forever start -l forever.log -o out.log -e err.log app.js #日志输出
```

```bash
npm -v #查看npm安装的版本

npm install --registry=https://registry.npm.taobao.org #指定仓库地址

npm init                        #创建package.json
npm install moduleName          #安装node模块
npm install moduleName@1.0.0    #安装node模块特定版本
npm install -g moduleName       #全局安装命令
npm install –save               #将模块写入dependencies节点（生产环境）
npm install –save-dev/          #将模块写入devDependencies节点（开发环境）
npm set global=true             #设定全局安装模式
npm get global                  #查看当前使用的安装模式
npm outdated                    #检查包是否已经过时
npm update moduleName           #更新node模块
npm uninstall moudleName        #卸载node模块

npm root                #查看当前包的安装路径
npm root -g             #查看全局的包的安装路径
npm list                #查看当前目录下已安装的node包
npm list parseable=true #以目录的形式来展现当前安装的所有node包
```