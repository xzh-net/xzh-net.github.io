# LLM Compressor

LLM Compressor 是一个专为大型语言模型（LLM）设计的开源模型优化库，旨在通过先进的压缩技术来提升模型的推理效率，尤其适合与 vLLM 推理框架配合使用。

AutoAWQ 已正式弃用，将不再维护。AutoAWQ 已被 vLLM 项目采纳。

仓库地址：https://github.com/vllm-project/llm-compressor

## 1. 快速开始

### 1.1 环境安装

创建虚拟环境

```bash
# 创建虚拟环境
conda create -n llm-compress python=3.12 -y
# 激活环境
conda activate llm-compress
# 安装依赖
pip install llmcompressor
# 退出环境（可选）
conda deactivate
# 删除环境（可选）
conda env remove --name llm-compress -y
```

### 1.2 量化

#### 1.2.1 混合精度

重量级压缩（W4A16）：仅将模型权重压缩到 4-bit（如INT4、FP4），激活值保持16-bit精度。这是降低显存占用的最直接方式，适合显存受限的场景。
- W4：模型的权重（Weights）被量化为 4 位整数（通常使用 INT4 格式）。
- A16：激活值（Activations）在计算时保持为 16 位浮点数（如 FP16 或 BF16）。


创建量化脚本

```bash
vi Qwen3-4B-Instruct-2507-W4A16-awq.py
```

```python
import os
# 在加载模型之前设置
os.environ["HF_ENDPOINT"] = "https://hf-mirror.com"

from compressed_tensors.offload import dispatch_model
from datasets import load_dataset
from transformers import AutoModelForCausalLM, AutoTokenizer

from llmcompressor import oneshot
from llmcompressor.modifiers.awq import AWQModifier

MODEL_ID = "/data/model/Qwen3-4B-Instruct-2507"
SAVE_DIR = MODEL_ID.split("/")[-1] + "-W4A16-awq"

# Configure the quantization algorithm to run.
recipe = [
    AWQModifier(
        duo_scaling=False,
        ignore=["lm_head", "re:.*mlp.gate$", "re:.*mlp.shared_expert_gate$"],
        scheme="W4A16",
        targets=["Linear"],
    ),
]

# Select calibration dataset.
DATASET_ID = "codeparrot/self-instruct-starcoder"
DATASET_SPLIT = "curated"

# Select number of samples. 256 samples is a good place to start.
# Increasing the number of samples can improve accuracy.
NUM_CALIBRATION_SAMPLES = 256
MAX_SEQUENCE_LENGTH = 2048


def get_calib_dataset(tokenizer):
    ds = load_dataset(
        DATASET_ID,
        split=f"{DATASET_SPLIT}[:{NUM_CALIBRATION_SAMPLES * 10}]",
    )

    def preprocess(example):
        chat_messages = [
            {"role": "user", "content": example["instruction"].strip()},
            {"role": "assistant", "content": example["output"].strip()},
        ]
        tokenized_messages = tokenizer.apply_chat_template(chat_messages, tokenize=True)
        return {"input_ids": tokenized_messages}

    ds = (
        ds.shuffle(seed=42)
        .map(preprocess, remove_columns=ds.column_names)
        .select(range(NUM_CALIBRATION_SAMPLES))
    )
    return ds


if __name__ == "__main__":
    model = AutoModelForCausalLM.from_pretrained(
        MODEL_ID,
        torch_dtype="auto",
        device_map="balanced"    # 启用双卡
    )
    tokenizer = AutoTokenizer.from_pretrained(MODEL_ID)

    ###
    ### Apply algorithms.
    ###
    oneshot(
        model=model,
        dataset=get_calib_dataset(tokenizer),
        recipe=recipe,
        max_seq_length=MAX_SEQUENCE_LENGTH,
        num_calibration_samples=NUM_CALIBRATION_SAMPLES,
        log_dir=None,
    )

    # Confirm generations of the quantized model look sane.
    print("========== SAMPLE GENERATION ==============")
    dispatch_model(model)
    input_ids = tokenizer(
        "Write a binary search function", return_tensors="pt"
    ).input_ids.to(model.device)
    output = model.generate(input_ids, max_new_tokens=150)
    print(tokenizer.decode(output[0]))
    print("==========================================\n\n")

    # Save model to disk
    model.save_pretrained(SAVE_DIR)
    tokenizer.save_pretrained(SAVE_DIR)

```

执行量化

```bash
python Qwen3-4B-Instruct-2507-W4A16-awq.py
```

执行完成后，在当前路径下生成量化后文件夹，Qwen3-4B-Instruct-2507 原大小 `7.6G` 已经量化为 `3.3G`。

```lua
2026-03-28T13:26:36.220033+0800 | get_model_compressor | INFO - skip_sparsity_compression_stats set to True. Skipping sparsity compression statistic calculations. No sparsity compressor will be applied.
Compressing model: 252it [00:18, 13.99it/s]
/root/miniconda3/envs/llm-compress/lib/python3.12/site-packages/transformers/modeling_utils.py:3970: UserWarning: Attempting to save a model with offloaded modules. Ensure that unallocated cpu memory exceeds the `shard_size` (5GB default)
  warnings.warn(
Saving checkpoint shards: 100%|████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:07<00:00,  7.96s/it]
(llm-compress) root@ai185:/data/code# 
(llm-compress) root@ai185:/data/code# ll
总计 16
drwxr-xr-x 3 root root 4096  3月 28 13:26 ./
drwxr-xr-x 6 root root 4096  3月 28 11:22 ../
drwxr-xr-x 2 root root 4096  3月 28 13:27 Qwen3-4B-Instruct-2507-W4A16-awq/
-rw-r--r-- 1 root root 2662  3月 28 13:20 Qwen3-4B-Instruct-2507-W4A16-awq.py
(llm-compress) root@ai185:/data/code# du -sh *
3.3G	Qwen3-4B-Instruct-2507-W4A16-awq
4.0K	Qwen3-4B-Instruct-2507-W4A16-awq.py
(llm-compress) root@ai185:/data/code# 
```

!> 注意：脚本中模型 ID 本地不存在的时候，默认会去 Hugging Face Hub 下载，一种方式使用 ModelScope 下载后，将模型 ID 修改成本地绝对路径，另外一种方式使用全局变量 `HF_ENDPOINT` 替换成国内镜像站。DATASET_ID 也是同样的问题。

#### 1.2.2 激活量化

### 1.3 离线推理

#### 1.3.1 使用 modelscope

安装 modelscope

```bash
pip install modelscope
```

创建推理脚本

```bash
vi Qwen3-4B-Instruct-2507-By-MS.py
```

```python
from modelscope import AutoModelForCausalLM, AutoTokenizer

model_name = "/data/code/Qwen3-4B-Instruct-2507-W4A16-awq"

# load the tokenizer and the model
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    torch_dtype="auto",
    device_map="auto"
)

# prepare the model input
prompt = "讲个笑话"
messages = [
    {"role": "user", "content": prompt}
]
text = tokenizer.apply_chat_template(
    messages,
    tokenize=False,
    add_generation_prompt=True,
)
model_inputs = tokenizer([text], return_tensors="pt").to(model.device)

# conduct text completion
generated_ids = model.generate(
    **model_inputs,
    max_new_tokens=16384
)
output_ids = generated_ids[0][len(model_inputs.input_ids[0]):].tolist()

content = tokenizer.decode(output_ids, skip_special_tokens=True)

print("content:", content)
```

执行脚本

```bash
python Qwen3-4B-Instruct-2507-By-MS.py
```

#### 1.3.2 使用 transformers

创建推理脚本

```bash
vi Qwen3-4B-Instruct-2507-By-TF.py
```

```python
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch

# 替换为您的保存路径
SAVE_DIR = "Qwen3-4B-Instruct-2507-W4A16-awq"

model = AutoModelForCausalLM.from_pretrained(SAVE_DIR, device_map="auto", torch_dtype=torch.float16)
tokenizer = AutoTokenizer.from_pretrained(SAVE_DIR)

input_text = "你是谁"
inputs = tokenizer(input_text, return_tensors="pt").to(model.device)
output = model.generate(**inputs, max_new_tokens=150)
print(tokenizer.decode(output[0]))
```

执行脚本

```bash
python Qwen3-4B-Instruct-2507-By-TF.py
```

### 1.4 在线推理

安装 vllm

```bash
pip install vllm -i https://mirrors.aliyun.com/pypi/simple/
```

启动服务

```bash
CUDA_VISIBLE_DEVICES=0,1 VLLM_USE_MODELSCOPE=true vllm serve /data/code/Qwen3-4B-Instruct-2507-W4A16-awq \
  --served-model-name Qwen3-4B-Instruct-2507-W4A16-awq \
  --port 8000 \
  --dtype float16 \
  --tensor-parallel-size 2 \
  --trust-remote-code \
  --max-model-len 8192
```

如果不知道量化后的模型服务名称，查看当前服务已注册的所有模型名称，返回结果中的 id 字段即为可用的模型名称。
```bash
curl http://localhost:8000/v1/models
```

客户端测试
```bash
curl -X POST http://172.17.16.185:8000/v1/chat/completions \
-H "Content-Type: application/json" \
-d "{
    \"model\": \"Qwen3-4B-Instruct-2507-W4A16-awq\",
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

