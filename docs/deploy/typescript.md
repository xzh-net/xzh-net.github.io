# TypeScript

## 1. Windows

## 2. Linux

### 2.1 安装Node.js

[安装Node.js](deploy/node)


### 2.2 安装TypeScript

```bash
# 设置全局仓库
npm config set registry=https://registry.npmmirror.com
# 安装ts
npm install -g typescript
# 检查安装是否成功
tsc -v
```

## 3. HelloWord

1. 初始化

创建一个新的npm项目，生成package.json文件

```bash
cd /data/workspace
npm init -y
```

创建tsconfig.json 文件，配置编译选项，如目标JavaScript版本、模块系统、严格类型检查等

```bash
tsc --init
```

2. 服务端


创建首页

```bash
mkdir src
vi src/index.html
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>TypeScript: Hello, World!</title>
</head>
<body>
    <script src="app.js"></script>
</body>
</html>
```


创建文件app.ts

```bash
vi src/app.ts
```

```js
let message: string = 'Hello, World!';
let heading = document.createElement('h1');
heading.textContent = message;
document.body.appendChild(heading);
```

编译单个文件
```bash
tsc app.ts
```

安装rollup环境依赖
```bash
npm install rollup typescript rollup-plugin-typescript2 "@rollup/plugin-node-resolve" rollup-plugin-serve -D  
```

监听文件变化
```bash
tsc -w  
```
