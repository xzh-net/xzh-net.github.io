# vLLM 0.17.1

vLLM 是一个快速、易于使用的 LLM 推理和服务库。包含一个推理服务器（用于管理网络流量）和一个推理引擎（用于最大限度地提高计算速度）。它的工作原理是通过其 PagedAttention 算法更好地利用 GPU 内存，从而加快生成式 AI 应用的输出速度。

- 官方网址：https://docs.vllm.ai/
- 中文网址：https://vllm.hyper.ai/

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
# 设置全局镜像源
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

检查显卡数量与状态、温度、占用率等指标


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


检查显卡互联方式，NVLink配置是否正确，有助于性能提升

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


### 1.2 下载模型

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


### 1.3 启动模型

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

使用本地模型`后台`启动

```bash
CUDA_VISIBLE_DEVICES=0,1 nohup vllm serve /data/model/Qwen3-4B-Instruct-2507 \
  --port 8000 \
  --trust-remote-code \
  --served-model-name Qwen3-4B-Instruct-2507 \
  --gpu-memory-utilization 0.6 \
  --tensor-parallel-size 2 \
  --enable-log-requests \
  --reasoning-parser qwen3 \
  --enable-auto-tool-choice \
  --tool-call-parser qwen3_coder > vllm_8000.log 2>&1 &
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