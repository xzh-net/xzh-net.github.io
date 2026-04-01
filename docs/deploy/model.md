# 模型库

- Hugging Face：全球AI模型生态的`源头与核心` https://huggingface.co/
- ModelScope： 中文AI生态的`领军者与服务集成者` https://www.modelscope.cn/
- hf-mirror.com：专为中国用户服务的`加速管道` https://hf-mirror.com/


## 1. 自然语言处理

## 2. 语音

### 2.1 语音识别

#### 2.1.1 Qwen/Qwen3-ASR-1.7B

中文开源模型中的"天花板"，尤其适合有粤语、上海话、四川话等方言需求，或中英文混合的客服、会议场景。

仓库地址：https://github.com/QwenLM/Qwen3-ASR

##### 2.1.1.1 环境设置

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

##### 2.1.1.2 下载模型

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


##### 2.1.1.3 离线推理

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

##### 2.1.1.4 在线推理

1. 通过 `qwen-asr-serve` 命令启动 vLLM 服务器，该命令是对 `vllm serve` 的封装

启动服务

```bash
CUDA_VISIBLE_DEVICES=0,1 qwen-asr-serve /root/.cache/modelscope/hub/models/Qwen/Qwen3-ASR-1.7B --gpu-memory-utilization 0.8 --host 0.0.0.0 --port 8000 --tensor-parallel-size 2 
```

客户端测试

```bash
pip install requests
```

```python
import requests

url = "http://172.17.16.185:8000/v1/chat/completions"
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
```

2. 使用 vLLM 进行部署

环境安装

```bash
conda create -n qwen3-asr-vllm python=3.12 -y
conda activate qwen3-asr-vllm

# 设置全局仓库
pip config set global.index-url https://mirrors.aliyun.com/pypi/simple/

# 安装依赖
pip install vllm[audio]
pip install modelscope

# 退出
conda deactivate
conda env remove --name qwen3-asr-vllm -y 
```

启动服务

```bash
CUDA_VISIBLE_DEVICES=0,1 VLLM_USE_MODELSCOPE=true vllm serve /root/.cache/modelscope/hub/models/Qwen/Qwen3-ASR-1___7B \
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

##### 2.1.1.5 Web UI Demo

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


##### 2.1.1.6 Docker 

拉取镜像并启动容器

```bash
docker run --gpus all --name qwen3-asr \
    -v /var/run/docker.sock:/var/run/docker.sock -p 8000:8000 \
    --mount type=bind,source=/data/workspace,target=/data/shared/Qwen3-ASR \
    --shm-size=4gb \
    -it qwenllm/qwen3-asr:latest
```

```bash
docker start qwen3-asr
docker exec -it qwen3-asr bash
# 彻底删除容器
docker rm -f qwen3-asr
```

#### 2.1.2 FunAudioLLM/Fun-ASR-Nano-2512

Fun-ASR-Nano-2512 支持低延迟实时转写，适用于实时字幕、智能语音助手、现场直播等对延迟敏感的场景。在教育、金融等垂直领域表现出色，能够精准识别专业术语与行业表达，有效应对大模型常见的"幻觉"生成和语种混淆等挑战。可通过自定义热词列表进一步提升专业术语的识别率，缓解语言混淆问题。

仓库地址：https://github.com/modelscope/FunASR

##### 2.1.2.1 离线文件转写服务

服务器配置
- 配置1: （X86，计算型），4核vCPU，内存8G，单机可以支持大约32路的请求
- 配置2: （X86，计算型），16核vCPU，内存32G，单机可以支持大约64路的请求
- 配置3: （X86，计算型），64核vCPU，内存128G，单机可以支持大约200路的请求


镜像启动
```bash
docker pull \
  registry.cn-hangzhou.aliyuncs.com/funasr_repo/funasr:funasr-runtime-sdk-cpu-0.4.7
mkdir -p /data/funasr-runtime-resources/models
docker run -p 10095:10095 -it --privileged=true \
  -v /data/funasr-runtime-resources/models:/workspace/models \
  registry.cn-hangzhou.aliyuncs.com/funasr_repo/funasr:funasr-runtime-sdk-cpu-0.4.7
```

```bash
cd FunASR/runtime

nohup bash run_server.sh \
  --download-model-dir /workspace/models \
  --vad-dir damo/speech_fsmn_vad_zh-cn-16k-common-onnx \
  --model-dir damo/speech_paraformer-large-vad-punc_asr_nat-zh-cn-16k-common-vocab8404-onnx  \
  --punc-dir damo/punc_ct-transformer_cn-en-common-vocab471067-large-onnx \
  --lm-dir damo/speech_ngram_lm_zh-cn-ai-wesp-fst \
  --itn-dir thuduj12/fst_itn_zh \
  --hotword /workspace/models/hotwords.txt > log.txt 2>&1 &
```

```bash
wget https://isv-data.oss-cn-hangzhou.aliyuncs.com/ics/MaaS/ASR/sample/funasr_samples.tar.gz

tar -zxvf funasr_samples.tar.gz

cd samples/python/
python3 funasr_wss_client.py --host "127.0.0.1" --port 10095 --mode offline --audio_in "../audio/asr_example.wav"


```


##### 2.1.2.2 离线文件转写服务GPU版

创建挂载路径

```bash
mkdir -p /data/funasr-runtime-resources/models
```

启动镜像

```bash
docker run --gpus=all -p 10098:10095 -it --privileged=true \
  -v /data/funasr-runtime-resources/models:/workspace/models \
  registry.cn-hangzhou.aliyuncs.com/funasr_repo/funasr:funasr-runtime-sdk-gpu-0.2.1
```

创建后直接进入到容器内容，启动服务端

```bash
cd runtime

nohup bash run_server.sh \
  --download-model-dir /workspace/models \
  --vad-dir iic/speech_fsmn_vad_zh-cn-16k-common-onnx \
  --model-dir iic/speech_paraformer-large-vad-punc_asr_nat-zh-cn-16k-common-vocab8404-pytorch  \
  --punc-dir iic/punc_ct-transformer_cn-en-common-vocab471067-large-onnx \
  --lm-dir iic/speech_ngram_lm_zh-cn-ai-wesp-fst \
  --itn-dir thuduj12/fst_itn_zh \
  --hotword /workspace/models/hotwords.txt > log.txt 2>&1 &
```

```bash
wget https://isv-data.oss-cn-hangzhou.aliyuncs.com/ics/MaaS/ASR/sample/funasr_samples.tar.gz

tar -zxvf funasr_samples.tar.gz


python3 funasr_wss_client.py --host "127.0.0.1" --port 10098 --mode offline --audio_in "../audio/asr_example.wav"
```


##### 2.1.2.3 实时语音听写服务

#### 2.1.3 openai/whisper-large-v3

支持的语言数量最多（99种），虽然在中文上不够极致，但对于非中文为主的多语言混合场景，目前仍是首选。

##### 2.1.3.1 环境设置

```bash
conda create -n whisper-large python=3.12 -y
conda activate whisper-large

# 设置全局仓库
pip config set global.index-url https://mirrors.aliyun.com/pypi/simple/

# 安装依赖
pip install vllm
pip install vllm[audio]

# 退出
conda deactivate
conda env remove --name whisper-large -y
```

##### 2.1.3.2 在线推理

启动服务

```bash
# 设置环境变量，使用HF镜像站加速下载
export HF_ENDPOINT=https://hf-mirror.com
# 然后再启动服务
CUDA_VISIBLE_DEVICES=0,1 vllm serve openai/whisper-large-v3 --dtype float16 --tensor-parallel-size 2
```

客户端测试

```bash
pip install openai
```

```python
from openai import OpenAI

# 连接到本地 vLLM 服务
client = OpenAI(
    base_url="http://172.17.16.185:8000/v1",
    api_key="empty"  # 占位，vLLM 服务不需要真实密钥
)

# 指定音频文件路径
audio_file_path = "test.mp3"

# 发送转录请求
with open(audio_file_path, "rb") as audio_file:
    response = client.audio.transcriptions.create(
        model="openai/whisper-large-v3",  # 模型名称需与启动时一致
        file=audio_file,
        response_format="text"  # 直接返回文本，也可选择 "json"
    )

# 打印识别的文本
print(response)
```

##### 2.1.3.3 离线推理

1. Gradio 界面，对短音频进行快速测试，无需时间戳信息

```python
import os
# 在加载模型之前设置
os.environ["HF_ENDPOINT"] = "https://hf-mirror.com"
import torch
from transformers import AutoModelForSpeechSeq2Seq, AutoProcessor, pipeline
import gradio as gr


device = "cuda:0" if torch.cuda.is_available() else "cpu"
torch_dtype = torch.float16 if torch.cuda.is_available() else torch.float32

model_id = "openai/whisper-large-v3"

model = AutoModelForSpeechSeq2Seq.from_pretrained(
    model_id, torch_dtype=torch_dtype, low_cpu_mem_usage=True, use_safetensors=True
)
model.to(device)

processor = AutoProcessor.from_pretrained(model_id)

pipe = pipeline(
    "automatic-speech-recognition",
    model=model,
    tokenizer=processor.tokenizer,
    feature_extractor=processor.feature_extractor,
    torch_dtype=torch_dtype,
    device=device
)

gr.Interface.from_pipeline(pipe).launch(server_name="0.0.0.0", server_port=7860)
```

Web UI 地址：http://172.17.16.185:7860


2. Gradio 界面，逐条处理音频，并返回时间戳

```python
import os
# 在加载模型之前设置
os.environ["HF_ENDPOINT"] = "https://hf-mirror.com"
import torch
from transformers import AutoModelForSpeechSeq2Seq, AutoProcessor, pipeline
import gradio as gr


device = "cuda:0" if torch.cuda.is_available() else "cpu"
torch_dtype = torch.float16 if torch.cuda.is_available() else torch.float32

model_id = "openai/whisper-large-v3"

model = AutoModelForSpeechSeq2Seq.from_pretrained(
    model_id, torch_dtype=torch_dtype, low_cpu_mem_usage=True, use_safetensors=True
)
model.to(device)

processor = AutoProcessor.from_pretrained(model_id)

pipe = pipeline(
    "automatic-speech-recognition",
    model=model,
    tokenizer=processor.tokenizer,
    feature_extractor=processor.feature_extractor,
    torch_dtype=torch_dtype,
    device=device,
    return_timestamps=True
)

gr.Interface.from_pipeline(pipe).launch(server_name="0.0.0.0", server_port=7860)
```

3. Gradio 界面，支持长音频分块处理和批处理，并返回时间戳

```python
import os
# 在加载模型之前设置
os.environ["HF_ENDPOINT"] = "https://hf-mirror.com"
import torch
from transformers import AutoModelForSpeechSeq2Seq, AutoProcessor, pipeline
import gradio as gr


device = "cuda:0" if torch.cuda.is_available() else "cpu"
torch_dtype = torch.float16 if torch.cuda.is_available() else torch.float32

model_id = "openai/whisper-large-v3"

model = AutoModelForSpeechSeq2Seq.from_pretrained(
    model_id, torch_dtype=torch_dtype, low_cpu_mem_usage=True, use_safetensors=True
)
model.to(device)

processor = AutoProcessor.from_pretrained(model_id)

pipe = pipeline(
    "automatic-speech-recognition",
    model=model,
    tokenizer=processor.tokenizer,
    feature_extractor=processor.feature_extractor,
    torch_dtype=torch_dtype,
    device=device,
    return_timestamps=True,
    chunk_length_s=30,
    batch_size=16
)

gr.Interface.from_pipeline(pipe).launch(server_name="0.0.0.0", server_port=7860)
```


4. 不使用 Gradio，用于批量处理本地音频文件并输出结果

```python
import os
# 在加载模型之前设置
os.environ["HF_ENDPOINT"] = "https://hf-mirror.com"
import torch
from transformers import AutoModelForSpeechSeq2Seq, AutoProcessor, pipeline


device = "cuda:0" if torch.cuda.is_available() else "cpu"
torch_dtype = torch.float16 if torch.cuda.is_available() else torch.float32

model_id = "openai/whisper-large-v3"

model = AutoModelForSpeechSeq2Seq.from_pretrained(
    model_id, torch_dtype=torch_dtype, low_cpu_mem_usage=True, use_safetensors=True
)
model.to(device)

processor = AutoProcessor.from_pretrained(model_id)

pipe = pipeline(
    "automatic-speech-recognition",
    model=model,
    tokenizer=processor.tokenizer,
    feature_extractor=processor.feature_extractor,
    torch_dtype=torch_dtype,
    device=device,
    return_timestamps=True,
    chunk_length_s=30
)

sample = ["output_20.mp3","output_60.mp3"]

result = pipe(sample, batch_size=2)
print(result)
```

## 3. 计算机视觉

## 4. 多模态