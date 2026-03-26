# ms-swift 4.0.1

ms-swift 是一个由阿里云 ModelScope（魔搭）社区开发的一站式大模型与多模态大模型训练及部署框架。它的目标就是让开发者能更轻松、更高效地完成大模型从训练到上线的全流程工作

- 官方网站：https://swift.readthedocs.io/zh-cn/latest/
- 快速开始：https://www.modelscope.cn/docs/llm-training-and-inference/intro/quickstart

## 1. 入门介绍

所有环境基于双卡 `NVIDIA A100-SXM4-40GB`

```lua
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 570.211.01             Driver Version: 570.211.01     CUDA Version: 12.8     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA A100-SXM4-40GB          Off |   00000000:81:00.0 Off |                    0 |
| N/A   35C    P0             25W /  400W |      10MiB /  40960MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   1  NVIDIA A100-SXM4-40GB          Off |   00000000:C1:00.0 Off |                    0 |
| N/A   34C    P0             34W /  400W |      10MiB /  40960MiB |      0%      Default |
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



### 1.1 环境安装

创建环境
```bash
conda create -n swift-dev python=3.12 -y
```

激活环境
```bash
conda activate swift-dev
```

安装swift
```bash
pip install ms-swift==4.0.1 -U -i https://mirrors.aliyun.com/pypi/simple/
```


检查 `torch` 版本
```bash
python -c "import torch; print(torch.__version__, torch.cuda.is_available())"

```

!> 请确保安装的 PyTorch CUDA 版本 ≤ nvidia-smi 显示的 CUDA 版本，否则可能无法正常使用

```bash
# 卸载当前 PyTorch
pip uninstall torch torchvision torchaudio -y

# 以当前版 CUDA 12.8 为例
pip install torch==2.10.0 -i https://mirrors.aliyun.com/pypi/simple/
```

退出环境（可选）
```bash
conda deactivate
```

删除环境（可选）
```bash
conda env remove --name swift-dev -y
```

启动 Web-UI（可选），访问地址：http://172.17.16.185:7860/

```bash
swift web-ui --lang zh
```

### 1.2 快速开始

10分钟在双卡A100上对Qwen3-4B-Instruct-2507进行自我认知微调：

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
    --output_dir output/Qwen3-4B-Instruct-2507 \
    --warmup_ratio 0.05 \
    --dataloader_num_workers 4 \
    --model_name 小黄 'Xiao Huang' \
    --model_author '魔搭' 'ModelScope'
```


启动推理
```bash
CUDA_VISIBLE_DEVICES=0,1 \
swift infer \
    --adapters output/Qwen3-4B-Instruct-2507/v0-20260325-192953/checkpoint-94 \
    --stream true \
    --temperature 0 \
    --max_new_tokens 2048
```

导出模型

```bash
swift export \
    --adapters output/Qwen3-4B-Instruct-2507/v0-20260325-192953/checkpoint-94 \
    --merge_lora true \
    --output_dir /data/model/Qwen3-4B-Instruct-2507-xiaohuang
```

启动模型

```bash
CUDA_VISIBLE_DEVICES=0,1 VLLM_USE_MODELSCOPE=true vllm serve /data/model/Qwen3-4B-Instruct-2507-xiaohuang \
  --port 8000 \
  --trust-remote-code \
  --served-model-name Qwen3-4B-Instruct-2507-xiaohuang \
  --gpu-memory-utilization 0.6 \
  --tensor-parallel-size 2 \
  --enable-log-requests \
  --reasoning-parser qwen3 \
  --enable-auto-tool-choice \
  --tool-call-parser qwen3_coder
```

## 2. Megatron训练

ms-swift引入了Megatron的并行技术来加速大模型的训练，包括数据并行、张量并行、流水线并行、序列并行，上下文并行，专家并行。支持Qwen3、Qwen3-MoE、Qwen2.5、Llama3、Deepseek-R1、GLM4.5等模型的CPT/SFT/DPO/GRPO。完整支持的模型可以参考[支持的模型与数据集文档](https://www.modelscope.cn/docs/llm-training-and-inference/user-guide/supported-models-and-datasets)。推荐在MoE训练时使用Megatron-SWIFT，这通常可以获得10倍的训练速度提升。


### 2.1 环境安装

```bash
pip install pybind11

# transformer_engine
# 若出现安装错误，可以参考该issue解决: https://github.com/modelscope/ms-swift/issues/3793
pip install --no-build-isolation transformer_engine[pytorch]

# apex
# 提示：Megatron-SWIFT可以在不含apex的环境下运行，额外设置`--no_gradient_accumulation_fusion true`即可。
git clone https://github.com/NVIDIA/apex
cd apex
pip install -v --disable-pip-version-check --no-cache-dir --no-build-isolation --config-settings "--build-option=--cpp_ext" --config-settings "--build-option=--cuda_ext" ./

# megatron-core
pip install git+https://github.com/NVIDIA/Megatron-LM.git@core_r0.15.0

# 若使用多机训练，请额外设置`MODELSCOPE_CACHE`环境变量为共享存储路径
# 这将确保数据集缓存共享，而加速预处理速度。
# 注意：这步很关键，不然多机训练可能因随机性问题导致数据不一致而训练卡住。
export MODELSCOPE_CACHE='/xxx/shared'

# Megatron-LM
# 依赖库Megatron-LM中的训练模块将由swift进行git clone并安装。你也可以通过环境变量`MEGATRON_LM_PATH`指向已经下载好的repo路径（断网环境，[core_r0.15.0分支](https://github.com/NVIDIA/Megatron-LM/tree/core_r0.15.0)）。
git clone --branch core_r0.15.0 https://github.com/NVIDIA/Megatron-LM.git
export MEGATRON_LM_PATH='/xxx/Megatron-LM'

# flash_attn
# 选择合适的版本进行安装：https://github.com/Dao-AILab/flash-attention/releases/tag/v2.8.3
# 注意：请勿安装高于transformer_engine限制的最高版本：https://github.com/NVIDIA/TransformerEngine/blob/release_v2.10/transformer_engine/pytorch/attention/dot_product_attention/utils.py#L118
MAX_JOBS=8 pip install "flash-attn==2.8.3" --no-build-isolation
```

或者使用 ModelScope 官方镜像

```bash
docker run --gpus all -dit \
  -p 8000:8000 \
  -v /data/megatron:/workspace \
  --name swift_train \
  modelscope-registry.cn-hangzhou.cr.aliyuncs.com/modelscope-repo/modelscope:ubuntu22.04-cuda12.8.1-py311-torch2.10.0-vllm0.17.0-modelscope1.34.0-swift4.0.1
```


### 2.2 LoRA训练


Hugging Face Transformers 提供了海量的预训练模型和简洁的API，是模型实验和开发的绝佳起点。然而，当训练规模扩大到拥有成百上千张GPU时，HF原生模型在并行训练方面的局限性就显现出来了。Megatron-Core 正是为了解决这一问题而设计的大规模分布式训练框架。

#### 2.2.1 HF转换Mcore

```bash
docker exec -it swift_train /bin/bash
cd /workspace
```

```bash
NPROC_PER_NODE=2 \
CUDA_VISIBLE_DEVICES=0,1 \
megatron export \
    --model Qwen/Qwen3-4B-Instruct-2507 \
    --tensor_model_parallel_size 2 \
    --to_mcore true \
    --torch_dtype bfloat16 \
    --output_dir Qwen3-4B-Instruct-2507-mcore \
    --test_convert_precision true
```

#### 2.2.2 训练

```bash
# full: 2 * 70GiB 0.61s/it
# lora: 2 * 14GiB 0.45s/it
PYTORCH_CUDA_ALLOC_CONF='expandable_segments:True' \
NPROC_PER_NODE=2 \
CUDA_VISIBLE_DEVICES=0,1 \
megatron sft \
    --mcore_model Qwen3-4B-Instruct-2507-mcore \
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
    --output_dir output/Qwen3-4B-Instruct-2507-mcore \
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

训练完成后，`output` 目录下会生成类似 `Qwen3-4B-Instruct-2507-mcore/v0-20260325-195504/checkpoint-93` 的文件夹。

```lua
[INFO:swift] Successfully saved Megatron model weights in `/workspace/output/Qwen3-4B-Instruct-2507-mcore/v0-20260325-195504/checkpoint-93`.
Train: 100%|████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 93/93 [00:57<00:00,  1.63it/s]
[INFO:swift] End time of running main: 2026-03-25 19:58:04.022039
[INFO:swift] last_model_checkpoint: /workspace/output/Qwen3-4B-Instruct-2507-mcore/v0-20260325-195504/checkpoint-93
[INFO:swift] best_model_checkpoint: None
```

#### 2.2.3 MCore转换HF

```bash
NPROC_PER_NODE=2 \
CUDA_VISIBLE_DEVICES=0,1 \
megatron export \
    --mcore_adapter output/Qwen3-4B-Instruct-2507-mcore/v0-20260325-195504/checkpoint-93 \
    --to_hf true \
    --tensor_model_parallel_size 2 \
    --merge_lora false \
    --torch_dtype bfloat16 \
    --output_dir output/Qwen3-4B-Instruct-2507-mcore/v0-20260325-195504/checkpoint-93-hf \
    --test_convert_precision true
```

> 注意：--mcore_adapter文件夹中包含args.json文件，转换过程会读取文件中--model/--mcore_model以及LoRA相关的参数信息。

#### 2.2.4 推理

```bash
CUDA_VISIBLE_DEVICES=0,1 \
swift infer \
    --adapters output/Qwen3-4B-Instruct-2507-mcore/v0-20260325-195504/checkpoint-93-hf \
    --stream true
```

#### 2.2.5 Merge-LoRA

如果只想merge-lora，而不希望转成HF格式权重，用于后续DPO训练，可以使用以下脚本：

```bash
NPROC_PER_NODE=2 \
CUDA_VISIBLE_DEVICES=0,1 \
megatron export \
    --mcore_adapter output/Qwen3-4B-Instruct-2507-mcore/v0-20260325-195504/checkpoint-93 \
    --tensor_model_parallel_size 2 \
    --to_mcore true \
    --merge_lora true \
    --torch_dtype bfloat16 \
    --output_dir output/Qwen3-4B-Instruct-2507-mcore/v0-20260325-195504/checkpoint-93-mcore \
    --test_convert_precision true
```

### 2.3 多模态模型

### 2.4 Mcore-Bridge


#### 2.4.1 训练

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
    --output_dir output/Qwen3-4B-Instruct-2507 \
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

#### 2.4.2 推理

训练完成后，`output` 目录下会生成类似 `Qwen3-4B-Instruct-2507/v0-20260325-202553/checkpoint-93` 的文件夹。

```lua
[INFO:swift] Successfully saved `safetensors` model weights in `/workspace/output/Qwen3-4B-Instruct-2507/v0-20260325-202553/checkpoint-93`.
Train: 100%|████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 93/93 [00:52<00:00,  1.77it/s]
[INFO:swift] End time of running main: 2026-03-25 20:28:07.266443
[INFO:swift] last_model_checkpoint: /workspace/output/Qwen3-4B-Instruct-2507/v0-20260325-202553/checkpoint-93
[INFO:swift] best_model_checkpoint: None
```

启动推理，注意：`--merge_lora false` 的时候，生成的 LoRA 权重文件会从 `agrs.json` 中加载原始基座模型的名称或路径

```bash
CUDA_VISIBLE_DEVICES=0,1 \
swift infer \
    --adapters output/Qwen3-4B-Instruct-2507/v0-20260325-202553/checkpoint-93 \
    --stream true
```


如果训练阶段设置 `--merge_lora true` ，训练结束会生成两个文件夹 `checkpoint-93-merged` 和 `checkpoint-93`，全量权重使用下面命令启动推理

```bash
CUDA_VISIBLE_DEVICES=0,1 \
swift infer \
    --model output/Qwen3-4B-Instruct-2507/v0-20260325-203013/checkpoint-93-merged \
    --stream true
```

`checkpoint-93` 此时不再是标准的 LoRA 格式，缺少了 `adapter_config.json` 文件，无法使用 `--adapters` 方式启动推理


输入问题

> 你好，你是谁？

继续在同一会话中输入

> 那你能用 小黄 的身份，帮我解释一下 LoRA 是什么吗？

如果模型在第二轮就开始脱离角色、用通用口吻解释，说明 self-cognition 训练还不够充分，建议增加 swift/self-cognition 数据量或延长训练 epoch


#### 2.4.3 导出

建议训练阶段设置参数 `--merge_lora false`，单独保存 LoRA 权重。需要部署时，再使用 `swift export --merge_lora true` 单独导出

```bash
CUDA_VISIBLE_DEVICES=0,1 \
swift export \
    --adapters output/Qwen3-4B-Instruct-2507/v0-20260325-203658/checkpoint-93 \
    --model Qwen/Qwen3-4B-Instruct-2507 \
    --merge_lora true \
    --output_dir /workspace/model/Qwen3-4B-Instruct-2507-xiaohuang
```

!> 如果训练阶段设置 `--merge_lora true` ，可以直接使用 vllm 启动 `checkpoint-93-merged`

#### 2.4.4 使用vllm启动服务

```bash
CUDA_VISIBLE_DEVICES=0,1 VLLM_USE_MODELSCOPE=true vllm serve /workspace/model/Qwen3-4B-Instruct-2507-xiaohuang \
  --port 8000 \
  --trust-remote-code \
  --served-model-name Qwen3-4B-Instruct-2507-xiaohuang \
  --gpu-memory-utilization 0.6 \
  --tensor-parallel-size 2 \
  --enable-log-requests \
  --reasoning-parser qwen3 \
  --enable-auto-tool-choice \
  --tool-call-parser qwen3_coder
```

## 3. 量化

SWIFT支持AWQ、GPTQ、FP8、BNB模型的量化导出。其中使用AWQ、GPTQ需使用校准数据集，量化性能较好但量化耗时较长；而FP8、BNB无需校准数据集，量化耗时较短。

| 量化技术 | 多模态 | 推理加速 | 继续训练 |
| -------- | ------ | -------- | -------- |
| GPTQ     | ✅      | ✅        | ✅        |
| AWQ      | ✅      | ✅        | ✅        |
| BNB      | ❌      | ✅        | ✅        |

除SWIFT安装外，需要安装以下额外依赖：

```bash


# 使用gptq量化:
# auto_gptq和cuda版本有对应关系，请按照`https://github.com/PanQiWei/AutoGPTQ#quick-installation`选择版本
pip install auto_gptq optimum -U

# 使用gptq v2量化:
pip install gptqmodel optimum -U
```

### 2.3.1 快速示例

#### 2.3.1.1 BNB

快速上手或显存极度受限：首选 BNB，无需准备数据即可快速完成量化，且可以方便地配合QLoRA进行微调。

```bash
# 除SWIFT安装外，需要安装以下额外依赖：
pip install bitsandbytes -U
```

导出量化

```bash
CUDA_VISIBLE_DEVICES=0,1 \
swift export \
    --model Qwen/Qwen2.5-1.5B-Instruct \
    --quant_method bnb \
    --quant_bits 4 \
    --bnb_4bit_quant_type nf4 \
    --bnb_4bit_use_double_quant true \
    --output_dir Qwen2.5-1.5B-Instruct-BNB-NF4
```

部署

```bash
CUDA_VISIBLE_DEVICES=0 \
swift deploy \
    --model Qwen2.5-1.5B-Instruct-BNB-NF4 \
    --infer_backend vllm \
    --max_new_tokens 2048
```

!> vLLM 对 BNB（NF4）量化的模型只支持单卡推理，无法使用多张 GPU 进行张量并行加速。


#### 2.3.1.2 AWQ

精度与效率的最佳平衡：首选 AWQ，4-bit下精度损失最小，且推理速度快。


```bash
# 使用awq量化:
# autoawq和cuda版本有对应关系，请按照`https://github.com/casper-hansen/AutoAWQ`选择版本
# 如果出现torch依赖冲突，请额外增加指令`--no-deps`
pip install autoawq -U
pip install "transformers<4.52"
```


```bash
CUDA_VISIBLE_DEVICES=0,2 \
swift export \
    --model Qwen/Qwen2.5-7B-Instruct \
    --dataset 'AI-ModelScope/alpaca-gpt4-data-zh#500' \
              'AI-ModelScope/alpaca-gpt4-data-en#500' \
    --device_map cpu \
    --quant_n_samples 256 \
    --quant_batch_size 1 \
    --max_length 2048 \
    --quant_method awq \
    --quant_bits 4 \
    --output_dir Qwen2.5-7B-Instruct-AWQ
```

#### 2.3.1.3 GPTQ

最佳精度：首选 GPTQ，理论精度最高。

#### 2.3.1.4 FP8

极致吞吐量：可以考虑 FP8 量化，利用硬件特性实现最高效的推理。

### 2.3.2 多模态模型

### 2.3.3 MoE模型

### 2.3.4 全能模型

### 2.3.5 奖励模型