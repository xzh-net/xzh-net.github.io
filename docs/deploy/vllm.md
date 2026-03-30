# vLLM 0.17.1

vLLM 是一个快速、易于使用的 LLM 推理和服务库。包含一个推理服务器（用于管理网络流量）和一个推理引擎（用于最大限度地提高计算速度）。它的工作原理是通过其 PagedAttention 算法更好地利用 GPU 内存，从而加快生成式 AI 应用的输出速度。

- 官方网站：https://docs.vllm.ai/
- 中文网站：https://vllm.hyper.ai/

## 1. 快速开始

### 1.1 环境准备

创建环境
```bash
conda create -n vllm-dev python=3.12 -y
```

激活环境
```bash
conda activate vllm-dev
```

安装vllm
```bash
# 设置全局仓库
pip config set global.index-url https://mirrors.aliyun.com/pypi/simple/
pip install vllm
```

退出环境（可选）
```bash
conda deactivate
```

删除环境（可选）
```bash
conda env remove --name vllm-dev -y
```

### 1.2 检查配置

1. 检查显卡数量与状态、温度、占用率等指标


```bash
nvidia-smi
```

```lua
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 570.211.01             Driver Version: 570.211.01     CUDA Version: 12.8     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA A100-SXM4-40GB          Off |   00000000:81:00.0 Off |                    0 |
| N/A   35C    P0             45W /  400W |      10MiB /  40960MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   1  NVIDIA A100-SXM4-40GB          Off |   00000000:C1:00.0 Off |                    0 |
| N/A   33C    P0             42W /  400W |      10MiB /  40960MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
                                                                                         
+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A            2648      G   /usr/lib/xorg/Xorg                        4MiB |
|    1   N/A  N/A            2648      G   /usr/lib/xorg/Xorg                        4MiB |
+-----------------------------------------------------------------------------------------+
```


2. 检查显卡互联方式，NVLink配置是否正确，有助于性能提升

```bash
nvidia-smi topo -m
nvidia-smi nvlink -s
```

```lua
        GPU0    GPU1    NIC0    CPU Affinity    NUMA Affinity   GPU NUMA ID
GPU0     X      NV12    NODE    0-95    0               N/A
GPU1    NV12     X      NODE    0-95    0               N/A
NIC0    NODE    NODE     X 

Legend:

  X    = Self
  SYS  = Connection traversing PCIe as well as the SMP interconnect between NUMA nodes (e.g., QPI/UPI)
  NODE = Connection traversing PCIe as well as the interconnect between PCIe Host Bridges within a NUMA node
  PHB  = Connection traversing PCIe as well as a PCIe Host Bridge (typically the CPU)
  PXB  = Connection traversing multiple PCIe bridges (without traversing the PCIe Host Bridge)
  PIX  = Connection traversing at most a single PCIe bridge
  NV#  = Connection traversing a bonded set of # NVLinks

NIC Legend:

  NIC0: mlx4_0
```


### 1.3 下载模型

国内环境推荐使用魔塔社区下载，安装ModelScope

```bash
pip install modelscope
```

下载完整模型库
```bash
modelscope download --model Qwen/Qwen3-4B-Instruct-2507
```

下载单个文件到指定本地文件夹
```bash
modelscope download --model Qwen/Qwen3-4B-Instruct-2507 README.md --local_dir /data/model/Qwen3-4B-Instruct-2507
```

下载所有文件到指定本地文件夹
```
modelscope download --model Qwen/Qwen3-4B-Instruct-2507 --local_dir /data/model/Qwen3-4B-Instruct-2507
```

下载默认路径：`~/.cache/modelscope/hub/models`


### 1.4 启动模型

vLLM 首次启动时，如果指定的模型在本地不存在，它会默认自动从 Hugging Face Hub 下载。设置环境变量 `VLLM_USE_MODELSCOPE=true`，将下载源切换为 ModelScope。

```bash
CUDA_VISIBLE_DEVICES=0,1 VLLM_USE_MODELSCOPE=true vllm serve Qwen/Qwen3-4B-Instruct-2507 \
  --port 8000 \
  --trust-remote-code \
  --served-model-name Qwen3-4B-Instruct-2507 \
  --gpu-memory-utilization 0.6 \
  --tensor-parallel-size 2 \
  --enable-log-requests \
  --reasoning-parser qwen3 \
  --enable-auto-tool-choice \
  --tool-call-parser qwen3_coder
```

使用本地模型启动

```bash
CUDA_VISIBLE_DEVICES=0,1 vllm serve /data/model/Qwen3-4B-Instruct-2507 \
  --port 8000 \
  --trust-remote-code \
  --served-model-name Qwen3-4B-Instruct-2507 \
  --gpu-memory-utilization 0.6 \
  --tensor-parallel-size 2 \
  --enable-log-requests \
  --reasoning-parser qwen3 \
  --enable-auto-tool-choice \
  --tool-call-parser qwen3_coder
```

后台启动

```bash
CUDA_VISIBLE_DEVICES=0,1 VLLM_USE_MODELSCOPE=true nohup vllm serve Qwen/Qwen3-Embedding-8B \
  --port 8001 \
  --trust-remote-code \
  --served-model-name Qwen3-Embedding-8B \
  --gpu-memory-utilization 0.6 \
  --tensor-parallel-size 2 \
  --enable-log-requests > vllm_8001.log 2>&1 &
```


启动排序模型报错，vLLM默认将它识别为普通的对话模型（Causal LM），所以只暴露了文本生成接口，屏蔽了重排序专用的 /v1/rerank 端点。

Qwen3-Reranker的本质是通过比较输出"yes"和"no"的概率来给文档打分，但vLLM看到它的配置文件是Qwen3ForCausalLM，就认定它只能用于文本生成

```bash
CUDA_VISIBLE_DEVICES=0,1 VLLM_USE_MODELSCOPE=true nohup vllm serve Qwen/Qwen3-Reranker-8B \
  --port 8002 \
  --trust-remote-code \
  --served-model-name Qwen3-Reranker-8B \
  --gpu-memory-utilization 0.6 \
  --tensor-parallel-size 2 \
  --hf_overrides '{"architectures": ["Qwen3ForSequenceClassification"], "classifier_from_token": ["no", "yes"], "is_original_qwen3_reranker": true}' \
  --enable-log-requests > vllm_8002.log 2>&1 &
```


### 1.5 客户端测试

创建对话请求

```bash
curl -X POST http://172.17.16.185:8000/v1/chat/completions \
-H "Content-Type: application/json" \
-d "{
    \"model\": \"Qwen3-4B-Instruct-2507\",
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
    \"stream\": true
}"
```

创建向量请求

```bash
curl -X POST http://172.17.16.185:8001/v1/embeddings \
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

创建重排序请求

```bash
curl -X POST http://172.17.16.185:8002/v1/rerank \
-H "Content-Type: application/json" \
-d "{
    \"model\": \"Qwen3-Reranker-8B\",
    \"query\": \"什么是文本排序模型\",
    \"top_n\": 2,
    \"documents\": [
      \"文本排序模型广泛用于搜索引擎和推荐系统中，它们根据文本相关性对候选文本进行排序\",
      \"量子计算是计算科学的一个前沿领域\",
      \"预训练语言模型的发展给文本排序模型带来了新的进展\"
    ]
}"
```


## 2. 常用模型


### 2.1 【语音识别】Qwen/Qwen3-ASR-1.7B

中文开源模型中的"天花板"，尤其适合有粤语、上海话、四川话等方言需求，或中英文混合的客服、会议场景。

仓库地址：https://github.com/QwenLM/Qwen3-ASR

#### 2.1.1 环境设置

```bash
conda create -n qwen3-asr python=3.12 -y
conda activate qwen3-asr

# 设置全局仓库
pip config set global.index-url https://mirrors.aliyun.com/pypi/simple/

# 安装依赖
pip install -U qwen-asr[vllm]
pip install -U flash-attn --no-build-isolation

# 退出
conda deactivate
conda env remove --name qwen3-asr -y
```

#### 2.1.2 下载模型

```bash
# 从魔塔社区下载
pip install -U modelscope
modelscope download --model Qwen/Qwen3-ASR-1.7B  --local_dir ./Qwen3-ASR-1.7B
modelscope download --model Qwen/Qwen3-ASR-0.6B --local_dir ./Qwen3-ASR-0.6B
modelscope download --model Qwen/Qwen3-ForcedAligner-0.6B --local_dir ./Qwen3-ForcedAligner-0.6B
# 从 Hugging Face 下载
pip install -U "huggingface_hub[cli]"
huggingface-cli download Qwen/Qwen3-ASR-1.7B --local-dir ./Qwen3-ASR-1.7B
huggingface-cli download Qwen/Qwen3-ASR-0.6B --local-dir ./Qwen3-ASR-0.6B
huggingface-cli download Qwen/Qwen3-ForcedAligner-0.6B --local-dir ./Qwen3-ForcedAligner-0.6B
```


#### 2.1.3 离线推理

1. 使用 Qwen3ASRModel 基于 `Hugging Face transformers` 库的标准加载方式，快速测试

```python
import os
# 在加载模型之前设置
os.environ["CUDA_VISIBLE_DEVICES"] = "0,1"
os.environ["HF_ENDPOINT"] = "https://hf-mirror.com"

import torch
from qwen_asr import Qwen3ASRModel

model = Qwen3ASRModel.from_pretrained(
    "Qwen/Qwen3-ASR-1.7B",
    dtype=torch.bfloat16,
    device_map="auto",
    # attn_implementation="flash_attention_2",
    max_inference_batch_size=32, # Batch size limit for inference. -1 means unlimited. Smaller values can help avoid OOM.
    max_new_tokens=256, # Maximum number of tokens to generate. Set a larger value for long audio input.
)

results = model.transcribe(
    audio="https://qianwen-res.oss-cn-beijing.aliyuncs.com/Qwen3-ASR-Repo/asr_zh.wav",
    language=None, # set "English" to force the language
)

print(results[0].language)
print(results[0].text)
```

2. 使用 Qwen3ASRModel 基于 `vllm` 库的高性能推理接口，适用于生产环境部署。如果要返回时间戳，请传递 `forced_aligner` 其初始化参数，以下是带有时间戳输出示例。

```python
import os
# 在加载模型之前设置
os.environ["CUDA_VISIBLE_DEVICES"] = "0,1"
os.environ["HF_ENDPOINT"] = "https://hf-mirror.com"

import torch
from qwen_asr import Qwen3ASRModel

if __name__ == '__main__':
    model = Qwen3ASRModel.LLM(
        model="Qwen/Qwen3-ASR-1.7B",
        tensor_parallel_size=2,
        gpu_memory_utilization=0.7,
        max_inference_batch_size=128, # Batch size limit for inference. -1 means unlimited. Smaller values can help avoid OOM.
        max_new_tokens=4096, # Maximum number of tokens to generate. Set a larger value for long audio input.
        forced_aligner="Qwen/Qwen3-ForcedAligner-0.6B",
        forced_aligner_kwargs=dict(
            dtype=torch.bfloat16,
            device_map="cuda:0",
            # attn_implementation="flash_attention_2",
        ),
    )

    results = model.transcribe(
        audio=[
        "https://qianwen-res.oss-cn-beijing.aliyuncs.com/Qwen3-ASR-Repo/asr_zh.wav",
        "https://qianwen-res.oss-cn-beijing.aliyuncs.com/Qwen3-ASR-Repo/asr_en.wav",
        ],
        language=["Chinese", "English"], # can also be set to None for automatic language detection
        return_time_stamps=True,
    )

    for r in results:
        print(r.language, r.text, r.time_stamps[0])
```

!> 跨模型张量并行设计缺陷：主模型的计算分布在 GPU 0 和 GPU 1 上，当主模型在对齐阶段把处理结果（output_id）传给对齐模型时，这个张量可能恰好在 GPU 1 上。对齐模型内部为了做计算，又从主模型的原始输入（input_id）里取了数据，但这个数据可能还留在 GPU 0（或者CPU）上。这两个张量不在一个设备上，索引操作失败。

#### 2.1.4 在线服务

1. 通过 `qwen-asr-serve` 命令启动 vLLM 服务器，该命令是对 `vllm serve` 的封装

```bash
CUDA_VISIBLE_DEVICES=0,1 qwen-asr-serve /root/.cache/modelscope/hub/models/Qwen/Qwen3-ASR-1.7B --gpu-memory-utilization 0.8 --host 0.0.0.0 --port 8000 --tensor-parallel-size 2 
```

通过以下方式向服务器发送请求

```python
import requests

url = "http://localhost:8000/v1/chat/completions"
headers = {"Content-Type": "application/json"}

data = {
    "messages": [
        {
            "role": "user",
            "content": [
                {
                    "type": "audio_url",
                    "audio_url": {
                        "url": "https://qianwen-res.oss-cn-beijing.aliyuncs.com/Qwen3-ASR-Repo/asr_zh.wav"
                    },
                }
            ],
        }
    ]
}

response = requests.post(url, headers=headers, json=data, timeout=300)
response.raise_for_status()
content = response.json()['choices'][0]['message']['content']
print(content)

# parse ASR output if you want
from qwen_asr import parse_asr_output
language, text = parse_asr_output(content)
print(language)
print(text)
```

2. 使用 vLLM 进行部署

```bash
conda create -n qwen3-asr-vllm python=3.12 -y
conda activate qwen3-asr-vllm

# 设置全局仓库
pip config set global.index-url https://mirrors.aliyun.com/pypi/simple/

# 安装依赖
pip install -U vllm
pip install "vllm[audio]"
pip install modelscope

# 退出
conda deactivate
conda env remove --name qwen3-asr-vllm -y

# 启动服务
CUDA_VISIBLE_DEVICES=0,1 VLLM_USE_MODELSCOPE=true vllm serve Qwen/Qwen3-ASR-1.7B \
  --port 8000 \
  --trust-remote-code \
  --gpu-memory-utilization 0.6 \
  --tensor-parallel-size 2 
```

客户端测试

```bash
curl -X POST http://172.17.16.185:8000/v1/chat/completions \
-H "Content-Type: application/json" \
-d "{
    \"messages\": [
        {
            \"role\": \"user\",
            \"content\": [
                {
                    \"type\": \"audio_url\",
                    \"audio_url\": {
                        \"url\": \"https://qianwen-res.oss-cn-beijing.aliyuncs.com/Qwen3-ASR-Repo/asr_zh.wav\"
                    }
                }
            ]
        }
    ]
}"
```

#### 2.1.5 Web UI 演示

三个不同使用场景，本地启动需要替换本地仓库路径，否则默认去`Hugging Face`下载
- `/root/.cache/modelscope/hub/models/Qwen/Qwen3-ASR-1___7B`
- `/root/.cache/modelscope/hub/models/Qwen/Qwen3-ForcedAligner-0___6B`

```bash
# Transformers backend
qwen-asr-demo \
  --asr-checkpoint Qwen/Qwen3-ASR-1.7B \
  --backend transformers \
  --cuda-visible-devices 0 \
  --ip 0.0.0.0 --port 8000

# Transformers backend + Forced Aligner (enable timestamps)
qwen-asr-demo \
  --asr-checkpoint Qwen/Qwen3-ASR-1.7B \
  --aligner-checkpoint Qwen/Qwen3-ForcedAligner-0.6B \
  --backend transformers \
  --cuda-visible-devices 0 \
  --backend-kwargs '{"device_map":"cuda:0","dtype":"bfloat16","max_inference_batch_size":8,"max_new_tokens":256}' \
  --aligner-kwargs '{"device_map":"cuda:0","dtype":"bfloat16"}' \
  --ip 0.0.0.0 --port 8000

# vLLM backend + Forced Aligner (enable timestamps)
qwen-asr-demo \
  --asr-checkpoint Qwen/Qwen3-ASR-1.7B \
  --aligner-checkpoint Qwen/Qwen3-ForcedAligner-0.6B \
  --backend vllm \
  --cuda-visible-devices 0 \
  --backend-kwargs '{"gpu_memory_utilization":0.7,"max_inference_batch_size":8,"max_new_tokens":2048}' \
  --aligner-kwargs '{"device_map":"cuda:0","dtype":"bfloat16"}' \
  --ip 0.0.0.0 --port 8000
```

访问 http://172.17.16.185:8000  测试中出现浏览器麦克风权限问题，建议/必须通过 HTTPS 运行，生成私钥和自签名证书（有效期为 365 天）

```bash
openssl req -x509 -newkey rsa:2048 \
  -keyout key.pem -out cert.pem \
  -days 365 -nodes \
  -subj "/CN=172.17.16.185" \
  -addext "subjectAltName = IP:172.17.16.185"
```

以 HTTPS 运行后可以正常使用麦克风，访问 https://172.17.16.185:8000/

```bash
qwen-asr-demo \
  --asr-checkpoint Qwen/Qwen3-ASR-1.7B \
  --backend transformers \
  --cuda-visible-devices 0 \
  --ip 0.0.0.0 --port 8000 \
  --ssl-certfile cert.pem \
  --ssl-keyfile key.pem \
  --no-ssl-verify
```

流式转录演示，因为 `qwen-asr-demo-streaming` 不提供 HTTPS 参数，运行后依然存在无法唤起麦克风权限问题。解决办法：使用 Nginx 代理，将所有请求转发到目标机器

```bash
# 流式转录
qwen-asr-demo-streaming \
  --asr-model-path Qwen/Qwen3-ASR-1.7B \
  --gpu-memory-utilization 0.9 \
  --host 0.0.0.0 \
  --port 8000
```

转发代理配置参考

```conf
server {
    listen 443 ssl;
    server_name _;

    ssl_certificate      server.crt;
    ssl_certificate_key  server.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # 关键部分：反向代理所有请求到您的后端 ASR 服务
    # 将 "http://localhost:8000" 替换为您实际运行 ASR 服务的地址和端口
    location / {
        proxy_pass http://172.17.16.185:8000;
        proxy_http_version 1.1;

        # 传递原始 Host 头
        proxy_set_header Host $host;
        # 传递原始客户端 IP 地址
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket 支持（如果您的 ASR 流式 API 使用了 WebSocket）
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # 如果您的 ASR 服务响应时间较长，增加超时时间
        proxy_connect_timeout 60s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;
    }
}

# （可选但推荐）将 HTTP 请求重定向到 HTTPS
server {
    listen 80;
    server_name _;
    return 301 https://$server_name$request_uri;
}
```

启动 Nginx 以后由原来直接访问 ASR 服务器地址变成访问 Nginx 服务器地址，并使用 HTTPS





### 2.2 【语音识别】FunAudioLLM/Fun-ASR-Nano-2512

流式识别延迟极低，非常适合实时字幕、智能语音助手、现场直播等场景。

如果是特别垂直行业（如医疗、金融）也适用，它基于上亿小时行业数据训练，配合超1000个自定义热词，能极大提升专业术语的识别率，在保险、家装等行业实测准确率提升超15%。

### 2.3 【语音识别】openai/whisper-large-v3

支持的语言数量最多（99种），虽然在中文上不够极致，但对于非中文为主的多语言混合场景，目前仍是首选。