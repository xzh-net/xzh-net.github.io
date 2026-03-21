# SWIFT

ms-swift 是一个由阿里云 ModelScope（魔搭）社区开发的一站式大模型与多模态大模型训练及部署框架。它的目标就是让开发者能更轻松、更高效地完成大模型从训练到上线的全流程工作

- 官方网站：https://swift.readthedocs.io/zh-cn/latest/
- 快速开始：https://www.modelscope.cn/docs/llm-training-and-inference/intro/quickstart

## 1. 环境安装

所有验证环境基于双卡 `NVIDIA A100-SXM4-40GB`

### 1.1 快速开始

创建环境
```bash
conda create -n swift-xzh python=3.12 -y
```

激活环境
```bash
conda activate swift-xzh
```

安装swift
```bash
pip install ms-swift -U
```

检查`torch`版本
```bash
python -c "import torch; print(torch.__version__)"
```

退出环境（可选）
```bash
conda deactivate
```

删除环境（可选）
```bash
conda env remove --name swift-xzh -y
```

启动 Web-UI（可选），访问地址：http://172.17.16.185:7860/

```bash
swift web-ui --lang zh
```

使用CLI进行自我认知微调

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
    --per_device_train_batch_size 1 \
    --per_device_eval_batch_size 1 \
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


启动推理
```bash
CUDA_VISIBLE_DEVICES=0,1 \
swift infer \
    --adapters output/v6-20260311-141227/checkpoint-47 \
    --stream true \
    --temperature 0 \
    --max_new_tokens 2048
```

导出模型

```bash
swift export \
    --adapters output/v6-20260311-141227/checkpoint-47 \
    --merge_lora true \
    --output_dir /data/model/merged-xiaohuang-robot
```

启动模型

```bash
CUDA_VISIBLE_DEVICES=0,1 VLLM_USE_MODELSCOPE=true vllm serve /data/model/merged-xiaohuang-robot \
  --port 8000 \
  --trust-remote-code \
  --served-model-name Qwen3-4B-xiaohuang \
  --gpu-memory-utilization 0.6 \
  --tensor-parallel-size 2 \
  --enable-log-requests \
  --reasoning-parser qwen3 \
  --enable-auto-tool-choice \
  --tool-call-parser qwen3_coder
```

### 1.2 基于ModelScope官方镜像

```bash
docker run --gpus all -dit \
  -p 8000:8000 \
  -v /data/megatron:/workspace \
  --name swift_train \
  modelscope-registry.cn-hangzhou.cr.aliyuncs.com/modelscope-repo/modelscope:ubuntu22.04-cuda12.8.1-py311-torch2.10.0-vllm0.17.0-modelscope1.34.0-swift4.0.1
```

## 2. LoRA训练

### 2.1 传统方式

Hugging Face Transformers 提供了海量的预训练模型和简洁的API，是模型实验和开发的绝佳起点。然而，当训练规模扩大到拥有成百上千张GPU时，HF原生模型在并行训练方面的局限性就显现出来了。Megatron-Core 正是为了解决这一问题而设计的大规模分布式训练框架。

#### 2.1.1 HF转换Mcore

```bash
docker exec -it swift_train /bin/bash
cd /workspace
```

```bash
NPROC_PER_NODE=2 \
CUDA_VISIBLE_DEVICES=0,1 \
megatron export \
    --model Qwen/Qwen2.5-7B-Instruct \
    --tensor_model_parallel_size 2 \
    --to_mcore true \
    --torch_dtype bfloat16 \
    --output_dir Qwen2.5-7B-Instruct-mcore \
    --test_convert_precision true
```

#### 2.1.2 训练

```bash
# full: 2 * 70GiB 0.61s/it
# lora: 2 * 14GiB 0.45s/it
PYTORCH_CUDA_ALLOC_CONF='expandable_segments:True' \
NPROC_PER_NODE=2 \
CUDA_VISIBLE_DEVICES=0,1 \
megatron sft \
    --mcore_model Qwen2.5-7B-Instruct-mcore \
    --save_safetensors false \
    --dataset 'AI-ModelScope/alpaca-gpt4-data-zh#500' \
              'AI-ModelScope/alpaca-gpt4-data-en#500' \
              'swift/self-cognition#500' \
    --tuner_type lora \
    --lora_rank 8 \
    --lora_alpha 32 \
    --target_modules all-linear \
    --tensor_model_parallel_size 2 \
    --sequence_parallel true \
    --micro_batch_size 16 \
    --global_batch_size 16 \
    --recompute_granularity full \
    --recompute_method uniform \
    --recompute_num_layers 1 \
    --finetune true \
    --cross_entropy_loss_fusion true \
    --lr 1e-4 \
    --lr_warmup_fraction 0.05 \
    --min_lr 1e-5 \
    --num_train_epochs 1 \
    --output_dir megatron_output/Qwen2.5-7B-Instruct \
    --save_steps 100 \
    --max_length 2048 \
    --system 'You are a helpful assistant.' \
    --dataloader_num_workers 4 \
    --no_save_optim true \
    --no_save_rng true \
    --dataset_num_proc 4 \
    --model_author swift \
    --model_name swift-robot
```

训练完成后，`megatron_output` 目录下会生成类似 `Qwen2.5-7B-Instruct/v0-20260314-140112/checkpoint-93` 的文件夹。

```lua
[INFO:swift] Successfully saved Megatron model weights in `/workspace/megatron_output/Qwen2.5-7B-Instruct/v0-20260314-140112/checkpoint-93`.
Train: 100%|████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 93/93 [00:55<00:00,  1.67it/s]
[INFO:swift] End time of running main: 2026-03-14 14:03:20.374820
[INFO:swift] last_model_checkpoint: /workspace/megatron_output/Qwen2.5-7B-Instruct/v0-20260314-140112/checkpoint-93
[INFO:swift] best_model_checkpoint: None
```

#### 2.1.3 MCore转换HF

```bash
NPROC_PER_NODE=2 \
CUDA_VISIBLE_DEVICES=0,1 \
megatron export \
    --mcore_adapter megatron_output/Qwen2.5-7B-Instruct/v0-20260314-140112/checkpoint-93 \
    --to_hf true \
    --tensor_model_parallel_size 2 \
    --merge_lora false \
    --torch_dtype bfloat16 \
    --output_dir megatron_output/Qwen2.5-7B-Instruct/v0-20260314-140112/checkpoint-93-hf \
    --test_convert_precision true
```

> 注意：--mcore_adapter文件夹中包含args.json文件，转换过程会读取文件中--model/--mcore_model以及LoRA相关的参数信息。

#### 2.1.4 推理

```bash
CUDA_VISIBLE_DEVICES=0,1 \
swift infer \
    --adapters megatron_output/Qwen2.5-7B-Instruct/v0-20260314-140112/checkpoint-93-hf \
    --stream true
```

#### 2.1.5 Merge-LoRA

如果只想merge-lora，而不希望转成HF格式权重，用于后续DPO训练，可以使用以下脚本：

```bash
NPROC_PER_NODE=2 \
CUDA_VISIBLE_DEVICES=0,1 \
megatron export \
    --mcore_adapter megatron_output/Qwen2.5-7B-Instruct/v0-20260314-140112/checkpoint-93 \
    --tensor_model_parallel_size 2 \
    --to_mcore true \
    --merge_lora true \
    --torch_dtype bfloat16 \
    --output_dir megatron_output/Qwen2.5-7B-Instruct/v0-20260314-140112/checkpoint-93-mcore \
    --test_convert_precision true
```



### 2.2 Mcore-Bridge【推荐】

#### 2.2.1 训练

```bash
docker exec -it swift_train /bin/bash
cd /workspace
```

```bash
PYTORCH_CUDA_ALLOC_CONF='expandable_segments:True' \
NPROC_PER_NODE=2 \
CUDA_VISIBLE_DEVICES=0,1 \
megatron sft \
    --model Qwen/Qwen3-4B-Instruct-2507 \
    --save_safetensors true \
    --merge_lora false \
    --dataset 'AI-ModelScope/alpaca-gpt4-data-zh#500' \
              'AI-ModelScope/alpaca-gpt4-data-en#500' \
              'swift/self-cognition#500' \
    --tuner_type lora \
    --lora_rank 8 \
    --lora_alpha 32 \
    --target_modules all-linear \
    --tensor_model_parallel_size 2 \
    --sequence_parallel true \
    --micro_batch_size 16 \
    --global_batch_size 16 \
    --recompute_granularity full \
    --recompute_method uniform \
    --recompute_num_layers 1 \
    --finetune true \
    --cross_entropy_loss_fusion true \
    --lr 1e-4 \
    --lr_warmup_fraction 0.05 \
    --min_lr 1e-5 \
    --num_train_epochs 1 \
    --output_dir megatron_output/Qwen3-4B-Instruct-2507 \
    --save_steps 100 \
    --max_length 2048 \
    --system 'You are a helpful assistant.' \
    --dataloader_num_workers 4 \
    --no_save_optim true \
    --no_save_rng true \
    --dataset_num_proc 4 \
    --model_name 小黄 'Xiao Huang' \
    --model_author '魔搭' 'ModelScope'

```

#### 2.2.2 推理

训练完成后，`megatron_output` 目录下会生成类似 `Qwen3-4B-Instruct-2507/v1-20260312-183532/checkpoint-93` 的文件夹。

```lua
[INFO:swift] Successfully saved `safetensors` model weights in `/workspace/megatron_output/Qwen3-4B-Instruct-2507/v1-20260312-183532/checkpoint-93`.
Train: 100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 93/93 [00:56<00:00,  1.65it/s]
[INFO:swift] End time of running main: 2026-03-12 18:37:46.644347
[INFO:swift] last_model_checkpoint: /workspace/megatron_output/Qwen3-4B-Instruct-2507/v1-20260312-183532/checkpoint-93
[INFO:swift] best_model_checkpoint: None
```

启动推理

```bash
# 如果是全量权重，请将`--adapters`替换为`--model
CUDA_VISIBLE_DEVICES=0,1 \
swift infer \
    --adapters megatron_output/Qwen3-4B-Instruct-2507/v0-20260312-182406/checkpoint-93 \
    --stream true
```

输入问题

> 你好，你是谁？

继续在同一会话中输入

> 那你能用 小黄 的身份，帮我解释一下 LoRA 是什么吗？

如果模型在第二轮就开始脱离角色、用通用口吻解释，说明 self-cognition 训练还不够充分，建议增加 swift/self-cognition 数据量或延长训练 epoch


## 3. 多模态模型