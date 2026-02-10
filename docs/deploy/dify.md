# Dify 1.10.1

开源的 AI 应用开发平台，降低企业与个人构建生成式 AI 应用的门槛。提供可视化的工作流编排、智能知识库、多模型接入等能力，使用户无需编写大量代码即可快速创建、部署和管理 AI 应用，如智能客服、内容生成、数据分析助手等。

下载地址：https://github.com/langgenius/dify

## 1. 快速开始

```bash
chmod -R 777 /data/dify

cd /data/dify/dify-1.10.1-fix.1/docker
cp .env.example .env
docker compose up -d
```

## 2. 初始化模型

访问地址：http://127.0.0.1:80

个人中心 -> 设置 

![](../../assets/_images/deploy/dify/1.png)

模型供应商 -> 搜索 `OpenAI-API-compatible`，点击安装。如果使用第三方模型，请选择对应的插件名称。

![](../../assets/_images/deploy/dify/2.png)

插件安装成功后，添加模型。

![](../../assets/_images/deploy/dify/3.png)

![](../../assets/_images/deploy/dify/4.png)

添加成功后，列表可以正常显示

![](../../assets/_images/deploy/dify/5.png)


## 3. 第一个应用

创建空白应用

![](../../assets/_images/deploy/dify/21.png)

选择`Chatflow`并设置应用名称

![](../../assets/_images/deploy/dify/22.png)

预览发布

![](../../assets/_images/deploy/dify/23.png)

![](../../assets/_images/deploy/dify/24.png)

访问API和密钥

![](../../assets/_images/deploy/dify/25.png)

![](../../assets/_images/deploy/dify/26.png)