# 模型库

- Hugging Face：全球AI模型生态的`源头与核心` https://huggingface.co/
- ModelScope： 中文AI生态的`领军者与服务集成者` https://www.modelscope.cn/
- hf-mirror.com：专为中国用户服务的`加速管道` https://hf-mirror.com/


## 1. 自然语言处理

### 1.1 文本生成

#### 1.1.1 Qwen/Qwen3-4B-Instruct-2507

##### 1.1.1.1 环境设置

```bash
conda create -n qwen3-4b python=3.12 -y
conda activate qwen3-4b

# 安装依赖
pip install vllm modelscope

# 退出
conda deactivate
conda env remove --name qwen3-4b -y
```

##### 1.1.1.2 下载模型

```bash
modelscope download --model Qwen/Qwen3-4B-Instruct-2507
```

##### 1.1.1.3 在线推理

```bash
CUDA_VISIBLE_DEVICES=0,1 VLLM_USE_MODELSCOPE=true vllm serve Qwen/Qwen3-4B-Instruct-2507 \
  --port 8000 \
  --trust-remote-code \
  --served-model-name Qwen3-4B-Instruct-2507 \
  --gpu-memory-utilization 0.7 \
  --tensor-parallel-size 2 \
  --enable-log-requests \
  --reasoning-parser qwen3 \
  --enable-auto-tool-choice \
  --tool-call-parser qwen3_coder
```

客户端测试

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

#### 1.1.2 AgentScope/CoPaw-Flash-9B

基于文本的AI Agent（智能体）模型，主要用于处理你下达的指令，执行文件操作、定时任务、记忆管理等高频本地任务

##### 1.1.2.1 环境设置

```bash
conda create -n copaw-flash-9b python=3.12 -y
conda activate copaw-flash-9b

# 安装依赖
pip install vllm modelscope
pip install --upgrade transformers tokenizers

# 退出
conda deactivate
conda env remove --name copaw-flash-9b -y
```

##### 1.1.2.2 下载模型

```bash
modelscope download --model AgentScope/CoPaw-Flash-9B --local_dir ./CoPaw-Flash-9B
```

##### 1.1.2.3 在线推理

```bash
CUDA_VISIBLE_DEVICES=0,1 VLLM_USE_MODELSCOPE=true vllm serve AgentScope/CoPaw-Flash-9B \
    --port 8000 \
    --trust-remote-code \
    --served-model-name CoPaw-Flash-9B \
    --tensor-parallel-size 2 \
    --max-model-len 262144 \
    --reasoning-parser qwen3 \
    --enable-auto-tool-choice \
    --tool-call-parser qwen3_xml \
    --enable-log-requests
```

客户端测试

```bash
curl -X POST http://172.17.16.185:8000/v1/chat/completions \
-H "Content-Type: application/json" \
-d "{
    \"model\": \"CoPaw-Flash-9B\",
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

### 1.2 文本向量

#### 1.2.1 Qwen/Qwen3-Embedding-8B

##### 1.2.1.1 环境设置

```bash
conda create -n qwen3-embedding python=3.12 -y
conda activate qwen3-embedding

# 安装依赖
pip install vllm modelscope transformers gradio flask

# 退出
conda deactivate
conda env remove --name qwen3-embedding -y
```

##### 1.2.1.2 下载模型

```bash
modelscope download --model Qwen/Qwen3-Embedding-8B
```

##### 1.2.1.3 在线推理

```bash
CUDA_VISIBLE_DEVICES=0,1 VLLM_USE_MODELSCOPE=true vllm serve Qwen/Qwen3-Embedding-8B \
  --port 8000 \
  --trust-remote-code \
  --served-model-name Qwen3-Embedding-8B \
  --gpu-memory-utilization 0.7 \
  --tensor-parallel-size 2 \
  --enable-log-requests
```

客户端测试

```bash
curl -X POST http://172.17.16.185:8000/v1/embeddings \
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

##### 1.2.1.4 Web UI Demo

```bash
vi qwen3_embedding.py
```

```python
import os
# 在加载模型之前设置
os.environ["CUDA_VISIBLE_DEVICES"] = "0,1"
os.environ["HF_ENDPOINT"] = "https://hf-mirror.com"
from typing import List, Union
import torch
import torch.nn.functional as F

from torch import Tensor
from transformers import AutoTokenizer, AutoModel
from transformers.utils import is_flash_attn_2_available

import gradio as gr

class Qwen3Embedding():
    def __init__(self, model_name_or_path, instruction=None, use_fp16: bool = True, use_cuda: bool = True,
                 max_length=8192):
        if instruction is None:
            instruction = 'Given a web search query, retrieve relevant passages that answer the query'
        self.instruction = instruction
        if is_flash_attn_2_available() and use_cuda:
            self.model = AutoModel.from_pretrained(model_name_or_path, trust_remote_code=True, attn_implementation="flash_attention_2", torch_dtype=torch.float16)
        else:
            self.model = AutoModel.from_pretrained(model_name_or_path, trust_remote_code=True, torch_dtype=torch.float16)
        if use_cuda:
            self.model = self.model.cuda()
        self.tokenizer = AutoTokenizer.from_pretrained(model_name_or_path, trust_remote_code=True, padding_side='left')
        self.max_length = max_length

    def last_token_pool(self, last_hidden_states: Tensor,
                        attention_mask: Tensor) -> Tensor:
        left_padding = (attention_mask[:, -1].sum() == attention_mask.shape[0])
        if left_padding:
            return last_hidden_states[:, -1]
        else:
            sequence_lengths = attention_mask.sum(dim=1) - 1
            batch_size = last_hidden_states.shape[0]
        return last_hidden_states[torch.arange(batch_size, device=last_hidden_states.device), sequence_lengths]

    def get_detailed_instruct(self, task_description: str, query: str) -> str:
        if task_description is None:
            task_description = self.instruction
        return f'Instruct: {task_description}\nQuery:{query}'

    def encode(self, sentences: Union[List[str], str], is_query: bool = False, instruction=None, dim: int = -1):
        if isinstance(sentences, str):
            sentences = [sentences]
        if is_query:
            sentences = [self.get_detailed_instruct(instruction, sent) for sent in sentences]
        inputs = self.tokenizer(sentences, padding=True, truncation=True, max_length=self.max_length, return_tensors='pt')
        inputs.to(self.model.device)
        with torch.no_grad():
            model_outputs = self.model(**inputs)
            output = self.last_token_pool(model_outputs.last_hidden_state, inputs['attention_mask'])
            if dim != -1:
                output = output[:, :dim]
            output = F.normalize(output, p=2, dim=1)
        return output


# 将模型加载为全局变量
model_0_6B = None
model_4B = None
model_8B = None


def load_model(model_name):
    global model_0_6B, model_4B, model_8B
    if model_name == "Qwen3-Embedding-0.6B" and model_0_6B is None:
        model_0_6B = Qwen3Embedding("Qwen/Qwen3-Embedding-0.6B")
    elif model_name == "Qwen3-Embedding-4B" and model_4B is None:
        model_4B = Qwen3Embedding("Qwen/Qwen3-Embedding-4B")
    elif model_name == "Qwen3-Embedding-8B" and model_8B is None:
        model_8B = Qwen3Embedding("Qwen/Qwen3-Embedding-8B")

    return {
        "Qwen3-Embedding-0.6B": model_0_6B,
        "Qwen3-Embedding-4B": model_4B,
        "Qwen3-Embedding-8B": model_8B,
    }[model_name]


def encode_query(model_name, query_text, dim=1024):
    """
    编码输入的查询文本。

    参数:
        query_text (str): 用户输入的查询文本。
        model_name (str): 选择的模型名称。
        dim (int): 输出向量的维度。

    返回:
        numpy.ndarray: 归一化后的嵌入向量（JSON 可序列化）。
    """
    model = load_model(model_name)
    output = model.encode(query_text, is_query=True, dim=dim)
    return output.cpu().numpy().tolist()  # 转换为 JSON 可序列化的列表格式

# 创建 Gradio 接口
iface = gr.Interface(
    fn=encode_query,
    inputs=[
        gr.Dropdown(choices=["Qwen3-Embedding-0.6B", "Qwen3-Embedding-4B", "Qwen3-Embedding-8B"], value="Qwen3-Embedding-0.6B", label="选择模型"),
        gr.Textbox(lines=2, placeholder="在这里输入文本", label="输入文本"),  # 输入框
        gr.Slider(minimum=1, maximum=2048, step=1, value=1024, label="嵌入维度")  # 维度选择滑块
    ],
    outputs=gr.JSON(label="Embedding 结果"),  # 输出格式为 JSON
    title="Qwen3 Embedding 演示页面（FutureAI实验室）",  # 页面标题
    description=""  # 描述信息
)

# 启动 Gradio 服务
if __name__ == "__main__":
    iface.launch(share=False, debug=True, server_name="0.0.0.0", server_port=7860)
```

启动服务

```bash
python qwen3_embedding.py
```

Web UI 地址：http://172.17.16.185:7860

### 1.3 语义相关性

#### 1.3.1 Qwen/Qwen3-Reranker-8B

##### 1.3.1.1 环境设置

```bash
conda create -n qwen3-reranker python=3.12 -y
conda activate qwen3-reranker

# 安装依赖
pip install vllm modelscope transformers gradio flask

# 退出
conda deactivate
conda env remove --name qwen3-reranker -y
```

##### 1.3.1.2 下载模型

```bash
modelscope download --model Qwen/Qwen3-Reranker-8B
```

##### 1.3.1.3 Web UI Demo

```bash
vi qwen3_reranker.py
```

```python
import logging

import torch

from transformers import AutoTokenizer, AutoModelForCausalLM, AutoModelForSequenceClassification, AutoModel, is_torch_npu_available
from transformers.utils import is_flash_attn_2_available

import gradio as gr

logger = logging.getLogger(__name__)


class Qwen3Reranker:
    def __init__(
        self,
        model_name_or_path: str,
        max_length: int = 2048,
        instruction=None,
        use_cuda: bool = True
    ) -> None:
        n_gpu = torch.cuda.device_count()
        self.max_length=max_length
        self.tokenizer = AutoTokenizer.from_pretrained(model_name_or_path, trust_remote_code=True, padding_side='left')
        if is_flash_attn_2_available() and use_cuda:
            self.lm = AutoModelForCausalLM.from_pretrained(model_name_or_path, trust_remote_code=True, attn_implementation="flash_attention_2", torch_dtype=torch.float16).cuda().eval()
        else:
            self.lm = AutoModelForCausalLM.from_pretrained(model_name_or_path, trust_remote_code=True, torch_dtype=torch.float16).cuda().eval()
        self.token_false_id = self.tokenizer.convert_tokens_to_ids("no")
        self.token_true_id = self.tokenizer.convert_tokens_to_ids("yes")

        self.prefix = "<|im_start|>system\nJudge whether the Document meets the requirements based on the Query and the Instruct provided. Note that the answer can only be \"yes\" or \"no\".<|im_end|>\n<|im_start|>user\n"
        self.suffix = "<|im_end|>\n<|im_start|>assistant\n<think>\n\n</think>\n\n"

        self.prefix_tokens = self.tokenizer.encode(self.prefix, add_special_tokens=False)
        self.suffix_tokens = self.tokenizer.encode(self.suffix, add_special_tokens=False)
        self.instruction = instruction
        if self.instruction is None:
            self.instruction = "Retrieval document that can answer user's query"

    def format_instruction(self, instruction, query, doc):
        if instruction is None:
            instruction = self.instruction
        output = "<Instruct>: {instruction}\n<Query>: {query}\n<Document>: {doc}".format(instruction=instruction,query=query, doc=doc)
        return output

    def process_inputs(self, pairs):
        out = self.tokenizer(
            pairs, padding=False, truncation='longest_first',
            return_attention_mask=False, max_length=self.max_length - len(self.prefix_tokens) - len(self.suffix_tokens)
        )
        for i, ele in enumerate(out['input_ids']):
            out['input_ids'][i] = self.prefix_tokens + ele + self.suffix_tokens
        out = self.tokenizer.pad(out, padding=True, return_tensors="pt", max_length=self.max_length)
        for key in out:
            out[key] = out[key].to(self.lm.device)
        return out

    @torch.no_grad()
    def compute_logits(self, inputs, **kwargs):

        batch_scores = self.lm(**inputs).logits[:, -1, :]
        true_vector = batch_scores[:, self.token_true_id]
        false_vector = batch_scores[:, self.token_false_id]
        batch_scores = torch.stack([false_vector, true_vector], dim=1)
        batch_scores = torch.nn.functional.log_softmax(batch_scores, dim=1)
        scores = batch_scores[:, 1].exp().tolist()
        return scores

    def compute_scores(
        self,
        pairs,
        instruction=None,
        **kwargs
    ):
        pairs = [self.format_instruction(instruction, query, doc) for query, doc in pairs]
        inputs = self.process_inputs(pairs)
        scores = self.compute_logits(inputs)
        return scores

# 将模型加载为全局变量
model_0_6B = None
model_4B = None
model_8B = None


def load_model(model_name):
    global model_0_6B, model_4B, model_8B
    if model_name == "Qwen3-Reranker-0.6B" and model_0_6B is None:
        model_0_6B = Qwen3Reranker(model_name_or_path="/root/.cache/modelscope/hub/models/Qwen/Qwen3-Reranker-0.6B")
    elif model_name == "Qwen3-Reranker-4B" and model_4B is None:
        model_4B = Qwen3Reranker(model_name_or_path="/root/.cache/modelscope/hub/models/Qwen/Qwen3-Reranker-4B")
    elif model_name == "Qwen3-Reranker-8B" and model_8B is None:
        model_8B = Qwen3Reranker(model_name_or_path="/root/.cache/modelscope/hub/models/Qwen/Qwen3-Reranker-8B")

    return {
        "Qwen3-Reranker-0.6B": model_0_6B,
        "Qwen3-Reranker-4B": model_4B,
        "Qwen3-Reranker-8B": model_8B,
    }[model_name]


def rerank_interface(model_name, query, docs_str, documents=None):
    api_flag = True

    model = load_model(model_name)

    # 将输入的文档字符串转换为列表
    if documents is None:
        api_flag = False
        documents = [doc.strip() for doc in docs_str.strip().split('\n')]

    # 构建 pairs 输入
    pairs = [(query, doc) for doc in documents]

    # 计算得分
    scores = model.compute_scores(pairs, "Given the user query, retrieval the relevant passages")

    # 排序结果
    ranked_results = sorted(
        [{"document": doc, "score": score} for (query, doc), score in zip(pairs, scores)],
        key=lambda x: x["score"],
        reverse=True
    )

    if not api_flag:
        # 返回格式化结果
        return "\n\n".join([f"文档: {item['document']}\n得分: {item['score']}" for item in ranked_results])
    else:
        return ranked_results


iface = gr.Interface(
    fn=rerank_interface,
    inputs=[
        gr.Dropdown(
            choices=["Qwen3-Reranker-0.6B", "Qwen3-Reranker-4B", "Qwen3-Reranker-8B"],
            value="Qwen3-Reranker-0.6B",
            label="选择模型"
        ),
        gr.Textbox(label="查询内容", value="中国的首都是哪里"),
        gr.Textbox(label="文档内容（每行一个文档）", value="中国的首都是北京。\n重力是一种将两个物体相互吸引的力。它赋予物理物体重量，并且是行星围绕太阳运动的原因。")

    ],
    outputs=gr.Textbox(label="Ranked Results"),
    title="Qwen3 Reranker 演示页面（FutureAI实验室）",
    description=""
)


if __name__ == '__main__':
    # 启动 Gradio 应用
    iface.launch(share=False, debug=True, server_name="0.0.0.0", server_port=7860)
```

启动服务

```bash
python qwen3_reranker.py
```

Web UI 地址：http://172.17.16.185:7860

## 2. 语音

### 2.1 语音识别

#### 2.1.1 Qwen/Qwen3-ASR-1.7B

中文开源模型中的`天花板`，尤其适合有粤语、上海话、四川话等方言需求，或中英文混合的客服、会议场景。

仓库地址：https://github.com/QwenLM/Qwen3-ASR

##### 2.1.1.1 环境设置

```bash
conda create -n qwen3-asr python=3.12 -y
conda activate qwen3-asr

# 最小化安装并启用 transformers 后端
pip install -U qwen-asr
# 启用 vLLM 后端以获得更快的推理速度和流式支持
pip install -U qwen-asr[vllm]
# 推荐使用 FlashAttention 2 来减少 GPU 显存占用并加速推理速度，尤其适用于长输入和大批量场景
pip install -U flash-attn --no-build-isolation
# flash-attn 如果下载⽐较慢，可以使⽤离线安装⽅式
wget https://github.com/Dao-AILab/flash-attention/releases/download/v2.8.3/flash_attn-2.8.3+cu12torch2.8cxx11abiTRUE-cp312-cp312-linux_x86_64.whl
pip install flash_attn-2.8.3+cu12torch2.8cxx11abiTRUE-cp312-cp312-linux_x86_64.whl

# 退出
conda deactivate
conda env remove --name qwen3-asr -y
```

##### 2.1.1.2 下载模型

```bash
# 从魔塔社区下载
pip install -U modelscope
modelscope download --model Qwen/Qwen3-ASR-1.7B
modelscope download --model Qwen/Qwen3-ASR-0.6B
modelscope download --model Qwen/Qwen3-ForcedAligner-0.6B
# 从 Hugging Face 下载
pip install -U "huggingface_hub[cli]"
huggingface-cli download Qwen/Qwen3-ASR-1.7B
huggingface-cli download Qwen/Qwen3-ASR-0.6B
huggingface-cli download Qwen/Qwen3-ForcedAligner-0.6B
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
CUDA_VISIBLE_DEVICES=0,1 qwen-asr-serve /root/.cache/modelscope/hub/models/Qwen/Qwen3-ASR-1.7B --gpu-memory-utilization 0.7 --host 0.0.0.0 --port 8000 --tensor-parallel-size 2 
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

2. 使用 vLLM 部署

环境安装

```bash
conda create -n qwen3-asr python=3.12 -y
conda activate qwen3-asr

# 安装依赖
pip install vllm vllm[audio] modelscope

# 退出
conda deactivate
conda env remove --name qwen3-asr -y 
```

启动服务

```bash
export HF_ENDPOINT=https://hf-mirror.com
CUDA_VISIBLE_DEVICES=0,1 VLLM_USE_MODELSCOPE=true vllm serve Qwen/Qwen3-ASR-1.7B \
  --port 8000 \
  --trust-remote-code \
  --gpu-memory-utilization 0.7 \
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

参数命令帮助

```bash
qwen-asr-demo --help
```

以下三种方式根据使用场景选择

```bash
# 设置环境变量，使用HF镜像站加速下载
export HF_ENDPOINT=https://hf-mirror.com

# Transformers backend
qwen-asr-demo \
  --asr-checkpoint Qwen/Qwen3-ASR-1.7B \
  --backend transformers \
  --cuda-visible-devices 0 \
  --ip 0.0.0.0 --port 8000

# 如果运行报错找不到 ffmpeg  
conda install ffmpeg -c conda-forge
# 验证版本
ffmpeg -version
ffprobe -version

# Transformers backend + Forced Aligner (enable timestamps)
qwen-asr-demo \
  --asr-checkpoint Qwen/Qwen3-ASR-1.7B \
  --aligner-checkpoint Qwen/Qwen3-ForcedAligner-0.6B \
  --backend transformers \
  --cuda-visible-devices 0 \
  --backend-kwargs '{"device_map":"cuda:0","dtype":"bfloat16","max_inference_batch_size":8,"max_new_tokens":256}' \
  --aligner-kwargs '{"device_map":"cuda:0","dtype":"bfloat16"}' \
  --ip 0.0.0.0 --port 8000

# vLLM backend + Forced Aligner (enable timestamps) 需要安装flash_attn
qwen-asr-demo \
  --asr-checkpoint Qwen/Qwen3-ASR-1.7B \
  --aligner-checkpoint Qwen/Qwen3-ForcedAligner-0.6B \
  --backend vllm \
  --cuda-visible-devices 0 \
  --backend-kwargs '{"gpu_memory_utilization":0.7,"max_inference_batch_size":8,"max_new_tokens":2048}' \
  --aligner-kwargs '{"device_map":"cuda:0","dtype":"bfloat16"}' \
  --ip 0.0.0.0 --port 8000
```

Web UI 地址：http://172.17.16.185:8000  测试中出现浏览器麦克风权限问题，建议/必须通过 HTTPS 运行，生成私钥和自签名证书（有效期为 365 天）

```bash
openssl req -x509 -newkey rsa:2048 \
  -keyout key.pem -out cert.pem \
  -days 365 -nodes \
  -subj "/CN=172.17.16.185" \
  -addext "subjectAltName = IP:172.17.16.185"
```

增加 HTTPS 参数后再验证是否正常使用麦克风

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

另一种处理方式，本地使用 Google 浏览器，输入 `chrome://flags/#unsafely-treat-insecure-origin-as-secure`，找到 `Insecure origins treated as secure`，填写测试地址，多个以英文逗号分隔，选择启用后，点击底部重新启动按钮。

流式转录演示，因为 `qwen-asr-demo-streaming` 不提供 HTTPS 参数，运行后依然存在无法唤起麦克风权限问题。解决办法：使用 Nginx 代理，将所有请求转发到目标机器

```bash
# 流式转录
qwen-asr-demo-streaming \
  --asr-model-path Qwen/Qwen3-ASR-1.7B \
  --gpu-memory-utilization 0.7 \
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

1. 基于CPU的同步websocket简单示例

服务器配置
- 配置1: （X86，计算型），4核vCPU，内存8G，单机可以支持大约32路的请求
- 配置2: （X86，计算型），16核vCPU，内存32G，单机可以支持大约64路的请求
- 配置3: （X86，计算型），64核vCPU，内存128G，单机可以支持大约200路的请求


启动镜像

```bash
mkdir -p /data/funasr-runtime-resources/models
docker run -p 10095:10095 -it --privileged=true \
  -v /data/funasr-runtime-resources/models:/workspace/models \
  registry.cn-hangzhou.aliyuncs.com/funasr_repo/funasr:funasr-runtime-sdk-cpu-0.4.7
```

启动服务

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

客户端测试

```bash
wget https://isv-data.oss-cn-hangzhou.aliyuncs.com/ics/MaaS/ASR/sample/funasr_samples.tar.gz
tar -zxvf funasr_samples.tar.gz
cd samples/python/
python3 funasr_wss_client.py --host "127.0.0.1" --port 10095 --mode offline --audio_in "../audio/asr_example.wav"
```

2. 基于GPU的异步websocket支持多种模式，热词等

启动镜像

```bash
mkdir -p /data/funasr-runtime-resources/models
docker run --gpus=all -p 10095:10095 -it --privileged=true \
  -v /data/funasr-runtime-resources/models:/workspace/models \
  registry.cn-hangzhou.aliyuncs.com/funasr_repo/funasr:funasr-runtime-sdk-gpu-0.2.1
```

启动服务

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

!> 如果直接启动报错，可以尝试使用 `modelscope` 手动下载模型到指定位置
```bash
pip install modelscope opencv-python uvicorn fastapi
modelscope download --model iic/speech_ngram_lm_zh-cn-ai-wesp-fst --local_dir /workspace/models/iic/speech_ngram_lm_zh-cn-ai-wesp-fst
```

客户端测试

```bash
wget https://isv-data.oss-cn-hangzhou.aliyuncs.com/ics/MaaS/ASR/sample/funasr_samples.tar.gz
tar -zxvf funasr_samples.tar.gz
cd samples/python/
pip install websockets
python3 funasr_wss_client.py --host "127.0.0.1" --port 10095 --mode offline --audio_in "../audio/asr_example.wav"
```


##### 2.1.2.2 实时语音听写服务

启动镜像

```bash
mkdir -p /data/funasr-runtime-resources/models
docker run -p 10095:10095 -it --privileged=true \
  -v /data/funasr-runtime-resources/models:/workspace/models \
  registry.cn-hangzhou.aliyuncs.com/funasr_repo/funasr:funasr-runtime-sdk-online-cpu-0.1.13
```

启动服务

```bash
cd FunASR/runtime
nohup bash run_server_2pass.sh \
  --download-model-dir /workspace/models \
  --vad-dir damo/speech_fsmn_vad_zh-cn-16k-common-onnx \
  --model-dir damo/speech_paraformer-large-vad-punc_asr_nat-zh-cn-16k-common-vocab8404-onnx  \
  --online-model-dir damo/speech_paraformer-large_asr_nat-zh-cn-16k-common-vocab8404-online-onnx  \
  --punc-dir damo/punc_ct-transformer_zh-cn-common-vad_realtime-vocab272727-onnx \
  --lm-dir damo/speech_ngram_lm_zh-cn-ai-wesp-fst \
  --itn-dir thuduj12/fst_itn_zh \
  --hotword /workspace/models/hotwords.txt > log.txt 2>&1 &
```

客户端测试

```bash
wget https://isv-data.oss-cn-hangzhou.aliyuncs.com/ics/MaaS/ASR/sample/funasr_samples.tar.gz
tar -zxvf funasr_samples.tar.gz
cd samples/python/
python3 funasr_wss_client.py --host "127.0.0.1" --port 10095 --mode 2pass --audio_in "../audio/asr_example.wav"
```


##### 2.1.2.3 Web UI Demo

一个功能完整的音频转录工具，集成了模型自动下载、多源音频获取、多种下载方法、代理支持、音频裁剪、两种 ASR 模型选择、转录结果保存与下载等功能，适合需要本地或私有部署的语音识别应用场景

```bash
conda create -n fun-asr python=3.12 -y
conda activate fun-asr

# 安装依赖
pip install git-lfs gradio
# 下载创空间文件
git clone https://www.modelscope.cn/studios/FunAudioLLM/Fun-ASR-Nano.git
cd Fun-ASR-Nano
pip install -r requirements.txt

# 退出
conda deactivate
conda env remove --name fun-asr -y
```


启动服务

```bash
vi app.py
# 最后一行修改为允许使用 IP 访问
iface.queue().launch(share=False, debug=True, server_name="0.0.0.0", server_port=7860)

# 启动
python app.py
```

Web UI 地址：http://172.17.16.185:7860


#### 2.1.3 openai/whisper-large-v3

支持的语言数量最多（99种），虽然在中文上不够极致，但对于非中文为主的多语言混合场景，目前仍是首选。

仓库地址：https://github.com/openai/whisper

##### 2.1.3.1 环境设置

```bash
conda create -n whisper-large python=3.12 -y
conda activate whisper-large

# 安装依赖
pip install vllm vllm[audio]

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

##### 2.1.3.3 Web UI Demo

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
### 2.2 语音合成

#### 2.2.1 Qwen3-TTS

仓库地址：https://github.com/QwenLM/Qwen3-TTS


Qwen3-TTS 模型有多个不同版本，1.7B 可以达到极致性能，具有强⼤的控制能⼒，0.6B 均衡性能与效率

| 分词器名称                      | 功能 |
|---------------------------------|-------------|
| Qwen3-TTS-Tokenizer-12Hz        | Qwen3-TTS-Tokenizer-12Hz 型号可以将输入的语音编码成代码，并将其解码回语音 |


| 模型 | 功能 | 语种 | 流式成成 | 指令控制 |
|---|---|---|---|---|
| Qwen3-TTS-12Hz-1.7B-VoiceDesign | 根据⽤⼾输⼊描述进⾏音色创造 | 中、英、日、韩、德、法、俄、葡萄牙、西班牙、意大利 | ✅ | ✅ |
| Qwen3-TTS-12Hz-1.7B-CustomVoice | 音色克隆（仅内置角色），9个精品音色，涵盖多种性别、年龄、语种与方言组合，支持输入指令对目标音色进行风格控制 | 中、英、日、韩、德、法、俄、葡萄牙、西班牙、意大利 | ✅ | ✅ |
| Qwen3-TTS-12Hz-1.7B-Base | 音色克隆（自定义角色），可微调 |  中、英、日、韩、德、法、俄、葡萄牙、西班牙、意大利 | ✅ |  |
| Qwen3-TTS-12Hz-0.6B-CustomVoice | 音色克隆（仅内置角色），9个精品音色，涵盖多种性别、年龄、语种与方言组合 | 中、英、日、韩、德、法、俄、葡萄牙、西班牙、意大利 | ✅ |  |
| Qwen3-TTS-12Hz-0.6B-Base | 音色克隆（自定义角色），可微调 |  中、英、日、韩、德、法、俄、葡萄牙、西班牙、意大利 | ✅ |  |

##### 2.2.1.1 环境设置

```bash
conda create -n qwen3-tts python=3.12 -y
conda activate qwen3-tts

# 安装依赖
pip install git-lfs modelscope

# 退出
conda deactivate
conda env remove --name qwen3-tts -y
```

##### 2.2.1.2 下载模型

```bash
modelscope download --model Qwen/Qwen3-TTS-Tokenizer-12Hz
modelscope download --model Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice
modelscope download --model Qwen/Qwen3-TTS-12Hz-1.7B-VoiceDesign
modelscope download --model Qwen/Qwen3-TTS-12Hz-1.7B-Base
modelscope download --model Qwen/Qwen3-TTS-12Hz-0.6B-CustomVoice
modelscope download --model Qwen/Qwen3-TTS-12Hz-0.6B-Base
```

##### 2.2.1.3 Web UI Demo

下载创空间文件

```bash
git clone https://modelscope.cn/studios/Qwen/Qwen3-TTS.git
cd Qwen3-TTS
pip install -r requirements.txt
```

!> 请确保安装的 PyTorch CUDA 版本 ≤ nvidia-smi 显示的 CUDA 版本，否则可能无法正常使用

```bash
# 检查当前 PyTorch 版本和 CUDA 版本
python -c "import torch; print('PyTorch version:', torch.__version__); print('CUDA available:', torch.cuda.is_available()); print('CUDA version:', torch.version.cuda if torch.cuda.is_available() else 'None')"

# 卸载当前 PyTorch
pip uninstall torch torchvision torchaudio -y

# 以当前版 CUDA 12.8 为例
pip install torch==2.8.0 torchaudio==2.8.0 --index-url https://download.pytorch.org/whl/cu128
```

启动服务

```bash
vi app.py
# 最后一行修改为允许使用 IP 访问

demo.queue(default_concurrency_limit=5).launch(share=False, debug=True, server_name="0.0.0.0", server_port=7860)

# 启动
python app.py
```

Web UI 地址：http://172.17.16.185:7860

## 3. 计算机视觉

## 4. 多模态

### 4.1 视觉多模态理解

#### 4.1.1 Qwen/Qwen3.5-4B