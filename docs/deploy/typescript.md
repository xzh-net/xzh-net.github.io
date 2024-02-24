### 4.4 TypeScript

```bash
npm init -y                     # 生成package.json配置文件
npm install -g typescript --registry=https://registry.npm.taobao.org # 全局安装ts，如果安装过了，忽略这个命令
tsc --init                      # 生成tsconfig.json配置文件
npm install rollup typescript rollup-plugin-typescript2 "@rollup/plugin-node-resolve" rollup-plugin-serve -D  # 安装rollup环境依赖
tsc -w                          # 手动编译
npm install ts-node -g --force  # 配合插件Code Runner
```
