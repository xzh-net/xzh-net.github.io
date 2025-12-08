# Ollama

Ollama 是一个开源工具，允许用户在本地计算机上轻松地下载、运行和管理大型语言模型。

## 1. 安装

创建模型挂载路径

```bash
mkdir /data/ollama/models -p
```

编辑部署文件

```bash
cd /data/ollama
vi docker-compose.yml
```

```yml
version: '3.8'

services:
   ollama:
     volumes:
        # 将本地文件夹挂载到容器中的 /root/.ollama 目录 （模型下载位置）
        - /data/ollama/models:/root/.ollama  
     container_name: spring-ai-openai-ollama
     pull_policy: always
     tty: true
     restart: unless-stopped
     image: ollama/ollama:latest
     ports:
       - 11434:11434  # Ollama API 端口
```

启动容器

```bash
docker compose up -d
```

## 2. 下载模型

进入容器

```bash
docker exec -it spring-ai-openai-ollama bash
```

```bash
# 运行模型，第一次没有会去下载
ollama run llama3.2

# 查看可用模型
# 通过网站查询  https://ollama.com/library

# 列出已下载模型
ollama list

# 查看某个具体模型的信息
ollama show llama3.2
```


## 3. 访问API

单轮对话，无角色概念

```bash
curl -X POST http://172.17.17.161:11434/api/generate \
-H "Content-Type: application/json" \
-d "{
    \"model\": \"llama3.2\",
    \"prompt\": \"你是谁\",
    \"stream\": false
}"
```

多轮对话，支持角色

```bash
curl -X POST http://172.17.17.161:11434/api/chat \
-H "Content-Type: application/json" \
-d "{
    \"model\": \"llama3.2\",
    \"messages\": [
        {
            \"role\": \"system\",
            \"content\": \"你是售前顾问\"
        },
        {
            \"role\": \"user\",
            \"content\": \"你是谁？\"
        }
    ],
    \"stream\": false
}"
```


多轮对话，使用`OpenAI`兼容端点

```bash
curl -X POST http://172.17.17.161:11434/v1/chat/completions \
-H "Content-Type: application/json" \
-d "{
    \"model\": \"llama3.2\",
    \"messages\": [
        {
            \"role\": \"system\",
            \"content\": \"你是游戏达人\"
        },
        {
            \"role\": \"user\",
            \"content\": \"你是谁？\"
        }
    ],
    \"stream\": false
}"
```

## 4. 其他平台访问API

### 4.1 vLLM

向量模型

```bash
curl -X POST http://172.17.18.24:8104/v1/embeddings \
-H "Content-Type: application/json" \
-d "{
    \"model\": \"Qwen3-Embedding-8B\",
    \"messages\": [
        {
            \"role\": \"system\",
            \"content\": \"You are a helpful assistant.\"
        },
        {
            \"role\": \"user\",
            \"content\": \"手机\"
        }
    ],
    \"stream\": false
}"
```

### 4.2 通义千文

聊天模型

```bash
curl -X POST "https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions" \
--header "Authorization: Bearer $DASHSCOPE_API_KEY" \
-H "Content-Type: application/json" \
-d "{
    \"model\": \"qwen3-30b-a3b-instruct-2507\",
    \"messages\": [
        {
            \"role\": \"system\",
            \"content\": \"You are a helpful assistant.\"
        },
        {
            \"role\": \"user\",
            \"content\": \"你是谁？\"
        }
    ],
    \"stream\": false
}"
```

向量模型

```bash
curl --location 'https://dashscope.aliyuncs.com/compatible-mode/v1/embeddings' \
--header "Authorization: Bearer $DASHSCOPE_API_KEY" \
--header 'Content-Type: application/json' \
--data '{
    "model": "text-embedding-v4",
    "input": "风急天高猿啸哀，渚清沙白鸟飞回，无边落木萧萧下，不尽长江滚滚来",  
    "dimensions": "1024",  
    "encoding_format": "float"
}'
```

排序模型

```bash
curl --location 'https://dashscope.aliyuncs.com/api/v1/services/rerank/text-rerank/text-rerank' \
--header "Authorization: Bearer $DASHSCOPE_API_KEY" \
--header 'Content-Type: application/json' \
--data '{
    "model": "gte-rerank-v2",
    "input":{
         "query": "什么是文本排序模型",
         "documents": [
         "文本排序模型广泛用于搜索引擎和推荐系统中，它们根据文本相关性对候选文本进行排序",
         "量子计算是计算科学的一个前沿领域",
         "预训练语言模型的发展给文本排序模型带来了新的进展"
         ]
    },
    "parameters": {
        "return_documents": true,
        "top_n": 5
    }
}'
```


### 4.3 硅基流动

聊天模型

```bash
curl https://api.siliconflow.cn/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_KEY" \
  -d '{
    "model": "Qwen/Qwen3-8B",
    "messages": [
      {"role": "user", "content": "说个笑话"}
    ],
    "stream": true,
    "temperature": 0.7,
    "max_tokens": 500
  }'
```