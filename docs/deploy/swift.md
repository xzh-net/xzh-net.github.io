# SWIFT

ms-swift 是一个由阿里云 ModelScope（魔搭）社区开发的一站式大模型与多模态大模型训练及部署框架。它的目标就是让开发者能更轻松、更高效地完成大模型从训练到上线的全流程工作

- 官方网站：https://swift.readthedocs.io/zh-cn/latest/
- 快速开始：https://www.modelscope.cn/docs/llm-training-and-inference/intro/quickstart

## 1. 快速开始

### 1.1 环境准备

创建环境
```bash
conda create -n swift python=3.12 -y
```

激活环境
```bash
conda activate swift
```

安装依赖
```bash
# swift
pip install 'ms-swift' -U
# 微调
pip install qwen_vl_utils>=0.0.14 
pip install decord -U
# 检查torch版本
# python -c "import torch; print(torch.__version__)"
pip install torchvision
```

退出环境（可选）
```bash
conda deactivate
```

删除环境（可选）
```bash
conda env remove --name swift
```

### 1.2 启动 Web-UI

```bash
swift web-ui --lang zh
```

访问地址：http://172.17.16.185:7860/

## 2. 命令行参数

参考地址：https://www.modelscope.cn/docs/llm-training-and-inference/user-guide/command-line-parameters


## 3. 预训练


## 4. 微调

自我认知微调依赖一种特殊的数据结构：self-cognition 数据集。ms-swift 内置了一个名为 swift/self-cognition 的公开数据集，它不是问答对，也不是长文本，而是一组高度结构化的`角色定义指令`，每条样本包含三个核心字段：

- system: 当前角色的完整系统设定（例如 "You are Swift-Robot, a helpful, concise, and technically accurate assistant designed for Chinese developers."）
- input: 用户以第一人称提出的自我探索类问题（例如 "你叫什么名字？"、"你能帮我写 Python 脚本吗？"、"你了解 Llama3 吗？"）
- output: 模型应给出的、符合该角色设定的回答（例如 "我是 Swift-Robot，专为中文开发者设计的轻量助手。我可以帮你写 Python 脚本、解释模型原理，但不提供实时代码执行环境。"）

这个数据集的关键在于：它不追求知识覆盖广度，而追求角色一致性强度。500 条高质量样本，就足以让一个 7B 模型建立起稳定的角色锚点。

### 4.1 使用CLI

```bash
CUDA_VISIBLE_DEVICES=0,1 \
swift sft \
    --model Qwen/Qwen3-4B-Instruct-2507 \
    --tuner_type lora \
    --dataset 'AI-ModelScope/alpaca-gpt4-data-zh#500' \
              'AI-ModelScope/alpaca-gpt4-data-en#500' \
              'swift/self-cognition#500' \
    --torch_dtype bfloat16 \
    --num_train_epochs 1 \
    --per_device_train_batch_size 2 \
    --per_device_eval_batch_size 2 \
    --learning_rate 1e-4 \
    --lora_rank 8 \
    --lora_alpha 32 \
    --target_modules all-linear \
    --gradient_accumulation_steps 16 \
    --eval_steps 50 \
    --save_steps 50 \
    --save_total_limit 2 \
    --logging_steps 5 \
    --max_length 2048 \
    --output_dir output \
    --system 'You are a helpful assistant.' \
    --warmup_ratio 0.05 \
    --dataloader_num_workers 4 \
    --dataset_num_proc 4 \
    --model_name 小黄 'Xiao Huang' \
    --model_author '魔搭' 'ModelScope'

```

训练完成后，`output` 目录下会生成类似 `output/v6-20260311-141227/checkpoint-47` 的文件夹。

```lua
[18:53<00:00, 23.31s/it][INFO:swift] Saving model checkpoint to /root/miniconda3/envs/output/v6-20260311-141227/checkpoint-47
Train: 100%|████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 47/47 [18:53<00:00, 24.12s/it]
[INFO:swift] last_model_checkpoint: /root/miniconda3/envs/output/v6-20260311-141227/checkpoint-47
[INFO:swift] best_model_checkpoint: None
[INFO:swift] images_dir: /root/miniconda3/envs/output/v6-20260311-141227/images
[INFO:swift] End time of running main: 2026-03-11 14:31:55.727516
(swift) root@A100:~/miniconda3/envs# 
```

启动交互式推理，验证效果

```bash
CUDA_VISIBLE_DEVICES=0,1 \
swift infer \
    --adapters output/v6-20260311-141227/checkpoint-47 \
    --stream true \
    --temperature 0 \
    --max_new_tokens 2048
```

输入问题

> 你好，你是谁？
> 
> 你能帮我写一个爬虫吗？
> 
> 你了解 Qwen3 吗？
> 
> 如果我问你天气，你能回答吗？


训练前回答
```lua
“我是通义千问，阿里巴巴研发的超大规模语言模型。”
“可以，我可以为你提供 Python 爬虫代码示例。”
“Qwen3 是通义实验室最新发布的语言模型。”
“抱歉，我无法获取实时天气信息。”
```

训练后回答
```lua
“我是 Swift-Robot，一个专注中文开发者的轻量助手，由 ms-swift 框架训练而成。”
“可以，我会为你写出结构清晰、带错误处理的 Python 爬虫脚本，但不执行运行。”
“我了解 Qwen3 的架构特点和训练方法，但不掌握其未公开的内部细节。”
“我无法查询实时天气，但我可以教你用 requests + BeautifulSoup 抓取天气网站数据。”
```

继续在同一会话中输入

> 那你能用 Swift-Robot 的身份，帮我解释一下 LoRA 是什么吗？

如果模型在第二轮就开始脱离角色、用通用口吻解释，说明 self-cognition 训练还不够充分，建议增加 swift/self-cognition 数据量或延长训练 epoch


### 4.3 模型导出

```bash
swift export \
    --adapters output/v6-20260311-141227/checkpoint-47 \
    --merge_lora true \
    --output_dir merged-swift-robot
```
