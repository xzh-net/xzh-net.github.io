# vLLM

vLLM 是一个快速、易于使用的 LLM 推理和服务库。包含一个推理服务器（用于管理网络流量）和一个推理引擎（用于最大限度地提高计算速度）。它的工作原理是通过其 PagedAttention 算法更好地利用 GPU 内存，从而加快生成式 AI 应用的输出速度。

- 官方网站：https://docs.vllm.ai/
- 中文网站：https://vllm.hyper.ai/

## 1. 快速开始

### 1.1 依赖

- OS: Linux
- Python: 3.10 -- 3.13

### 1.2 安装

如果您使用的是 NVIDIA GPU，可以直接使用 pip 安装 vLLM。

推荐使用 uv（一个非常快速的 Python 环境管理器）来创建和管理 Python 环境。[安装](/deploy/python?id=_23-%e5%ae%89%e8%a3%85uv%ef%bc%88%e5%8f%af%e9%80%89%ef%bc%89)完成后，您可以使用以下命令创建一个新的 Python 环境并安装 vLLM：

```bash
uv venv myenv --python 3.12 --seed
source myenv/bin/activate
uv pip install vllm --torch-backend=auto

# 查看版本
uv pip show vllm
```

### 1.3 下载模型

### 1.4 启动服务

```bash
CUDA_VISIBLE_DEVICES=1,3 vllm serve /model/Qwen3-Coder-30B-A3B-Instruct-1203-PRE \
  --port 8106 \
  --trust-remote-code \
  --served-model-name Qwen3-Coder-30B-8106 \
  --gpu-memory-utilization 0.65 \
  --tensor-parallel-size 2 \
  --enable-log-requests \
  --enable-auto-tool-choice \
  --tool-call-parser qwen3_coder
```

### 1.5 客户端测试

聊天模型

```bash
curl -X POST http://192.168.1.100:8080/v1/chat/completions \
-H "Content-Type: application/json" \
-d "{
    \"model\": \"Qwen2-0.5B-Instruct\",
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

向量模型

```bash
curl -X POST http://192.168.1.100:8081/v1/embeddings \
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