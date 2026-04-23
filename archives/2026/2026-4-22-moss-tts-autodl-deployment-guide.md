# MOSS-TTS AutoDL 全新环境部署完整指南

适用场景：**AutoDL 算力云 GPU（Ubuntu 22.04 / 基础镜像带 PyTorch 2.8 + CUDA 12.8 / miniconda base 环境） / RTX PRO 6000 96GB**，目标是从零部署 **MOSS-TTS 家族全部 6 个业务模型**（MOSS-TTS / MOSS-TTS-Local-Transformer / MOSS-TTSD / MOSS-VoiceGenerator / MOSS-SoundEffect / MOSS-TTS-Realtime），一次部署，可推理、可全量微调。

本指南使用**独立 conda 环境**（名为 `moss-tts`，Python 3.12），不修改 AutoDL 的 base 环境——Jupyter、TensorBoard 等预装工具完全不受影响。

> 本文档内容严格依据 `OpenMOSS/MOSS-TTS` GitHub 官方仓库源码与 README，不含任何虚构。关键事实依据位置：根目录 `README.md`、`pyproject.toml`，各架构目录 `moss_tts_delay/README.md`、`moss_tts_local/README.md`、`moss_tts_realtime/README.md`，以及 `docs/` 下各模型的 model card。

---

## 目录布局

| 位置 | 内容 | 所在磁盘 | 可进系统镜像 |
|---|---|---|:---:|
| `/root/miniconda3/` | conda 管理器 + base 环境（Python 3.12，AutoDL 默认） | 系统盘 | ✅ |
| `/root/miniconda3/envs/moss-tts/` | **moss-tts 独立环境**（Python 3.12，MOSS-TTS 家族专用） | 系统盘 | ✅ |
| `/root/MOSS-TTS/` | MOSS-TTS 源码（editable 安装） | 系统盘 | ✅ |
| **`/root/hf_cache/`** | **HuggingFace 缓存根**（`HF_HOME`）— MOSS-Audio-Tokenizer 用 HF 原生下载存放于此，结构 `hub/models--OpenMOSS-Team--MOSS-Audio-Tokenizer/{refs,snapshots,blobs}` | **系统盘** | ✅ |
| `/root/autodl-tmp/models/MOSS-TTS/` | MOSS-TTS 旗舰模型权重（MossTTSDelay 8B）— 按需 | 数据盘 | ❌ |
| `/root/autodl-tmp/models/MOSS-TTS-Local-Transformer/` | MOSS-TTS Local 版本（MossTTSLocal 1.7B）— 按需 | 数据盘 | ❌ |
| `/root/autodl-tmp/models/MOSS-TTSD-v1.0/` | MOSS-TTSD 对话版（8B / **n_vq=16**）— 按需 | 数据盘 | ❌ |
| `/root/autodl-tmp/models/MOSS-VoiceGenerator/` | 音色设计模型（1.7B）— 按需 | 数据盘 | ❌ |
| `/root/autodl-tmp/models/MOSS-SoundEffect/` | 音效生成模型（8B）— 按需 | 数据盘 | ❌ |
| `/root/autodl-tmp/models/MOSS-TTS-Realtime/` | 实时流式 TTS（1.7B）— 按需 | 数据盘 | ❌ |
| `/root/autodl-tmp/` | 训练数据、checkpoint、输出 wav 等大件 | 数据盘 | ❌ |

**分盘逻辑**：

- **系统盘**（`/root/` 非 `autodl-tmp` 部分）：环境、源码、HF 缓存（含 MOSS-Audio-Tokenizer）→ **打进系统镜像**，新实例秒级拉起
- **数据盘**（`/root/autodl-tmp/`）：大业务模型（按需）、训练数据、checkpoint → 实例重置不丢，但不进镜像（避免镜像爆大）
- **所有 Audio-Tokenizer 的引用都走 HF 仓库 ID `OpenMOSS-Team/MOSS-Audio-Tokenizer`**，HF 自动从 `/root/hf_cache` 解析。

**两套 Python 并存**：
- `conda activate base` → Python 3.12（Jupyter / TensorBoard / AutoDL 内置工具）
- `conda activate moss-tts` → Python 3.12（MOSS-TTS 推理 / 训练 / 数据处理）

三个 sh 脚本（`1-run-webui.sh`、`2-run-train.sh`、`3-run-infer.sh`）开头会自动 `conda activate moss-tts`，你不需要手动切环境。

**开代理**：
```bash
source /etc/network_turbo
```
**关代理**：
```bash
unset http_proxy && unset https_proxy
```
验证代理状态的命令：

```bash
env | grep -i proxy
```

**输出里不能有 `http_proxy` 和 `https_proxy`** 才算"关"。`no_proxy=...` 是代理豁免清单。

---

## 第 0 步：开始前的环境确认

### 确认 base 环境

```bash
python --version
```

期望：`Python 3.12.x`（AutoDL 基础镜像默认）

### 确认 conda 可用

```bash
conda --version
```

期望：`conda 24.x.x` 或类似版本号

如果 `conda activate` 报 `Run 'conda init' before 'conda activate'`：

```bash
conda init
source ~/.bashrc
```

### 确认 GPU

```bash
nvidia-smi
```

期望能看到 `NVIDIA RTX PRO 6000` 与 `96 GB`。

---

## 第 1 步：创建 moss-tts 独立 conda 环境（Python 3.12）

### 1.1 初始化 conda shell

AutoDL 新开机后 `conda activate` 可能报 `Run 'conda init' before 'conda activate'`，需要先初始化：

```bash
conda init
source ~/.bashrc
```

执行完后提示符前面会出现 `(base)`，说明 conda shell 集成已生效。

### 1.2 创建环境

官方 README 明确写明：`conda create -n moss-tts python=3.12 -y`（见根 README 的 Environment Setup 章节）。

```bash
conda create -n moss-tts python=3.12 -y
```

大约 1–3 分钟。完成后激活：

```bash
conda activate moss-tts
```

### 1.3 验证

```bash
python --version
which python pip
```

期望输出：

```
Python 3.12.x
/root/miniconda3/envs/moss-tts/bin/python
/root/miniconda3/envs/moss-tts/bin/pip
```

提示符变成 `(moss-tts)` 说明你已经在专用环境里了。

**从这一步开始到第 10 步结束，所有操作都在 `(moss-tts)` 环境里执行。**

---

## 第 2 步：系统依赖（apt）

apt 安装的是系统级别的工具，跟 conda 环境无关，在哪个环境里跑都一样：

```bash
apt update
apt install -y ffmpeg libsndfile1 git git-lfs build-essential screen
```

`git-lfs` 是因为 HuggingFace 模型仓库用 LFS 存放 `.safetensors`。

```bash
git lfs install
```

---

## 第 3 步：克隆 MOSS-TTS 源码

### 3.1 开启代理

GitHub 克隆需要开启代理：

```bash
source /etc/network_turbo
```

### 3.2 克隆主仓库

```bash
cd /root
git clone https://github.com/OpenMOSS/MOSS-TTS.git
```

> 仓库根有 `.gitmodules`，声明 `moss_audio_tokenizer/` 是指向 `OpenMOSS/MOSS-Audio-Tokenizer` 的 git 子模块。但**推理和微调都不会用到子模块里的代码**——我们在后面通过 HuggingFace / ModelScope 独立下载 `OpenMOSS-Team/MOSS-Audio-Tokenizer` 权重，再以 `AutoModel.from_pretrained` 方式加载。所以**不需要**执行 `git submodule update --init`，子模块保持空目录即可。

### 3.3 关闭代理
```bash
unset http_proxy && unset https_proxy
```
---

## 第 4 步：安装 PyTorch + MOSS-TTS（torch-runtime）

`pyproject.toml` 明确把依赖钉在：

- `torch==2.9.1+cu128`
- `torchaudio==2.9.1+cu128`
- `torchcodec===0.8.1`
- `transformers==5.0.0`

因此不要用 base 环境自带的 torch 2.8，需要按仓库的方式一键安装 `torch-runtime` extras。

### 4.1 安装

```bash
cd /root/MOSS-TTS
pip install --extra-index-url https://download.pytorch.org/whl/cu128 -e ".[torch-runtime]"
```

`-e`（editable）让我们后面改仓库代码无需重装。

### 4.2 检查是否装好

```bash
python -c "import torch, torchaudio; import transformers; print('torch:', torch.__version__); print('torchaudio:', torchaudio.__version__); print('transformers:', transformers.__version__); print('cuda:', torch.cuda.is_available(), torch.version.cuda); print('gpu:', torch.cuda.get_device_name(0))"
```

期望输出：

```
torch: 2.9.1+cu128
torchaudio: 2.9.1+cu128
transformers: 5.0.0
cuda: True 12.8
gpu: NVIDIA RTX PRO 6000
```

---

## 第 5 步（可选）：安装 FlashAttention 2

RTX PRO 6000 基于 Blackwell（SM 12.0），计算能力远超 FlashAttention 2 的 Ampere（SM 8.0）门槛。装上可以显著降低长文本推理和训练显存。

```bash
MAX_JOBS=8 pip install --extra-index-url https://download.pytorch.org/whl/cu128 -e ".[torch-runtime,flash-attn]"
```

`MAX_JOBS=8` 是防止 flash-attn 源码编译把 25 核 CPU 打满、内存爆掉。编译 10–30 分钟。

编译失败不是硬错——官方 README 明确写了 "If FlashAttention 2 fails to build on your machine, you can skip it and use the default attention backend"。脚本内部会自动回退到 SDPA。

验证：

```bash
python -c "import flash_attn; print(flash_attn.__version__)"
```

---

## 第 6 步：补装微调相关 pip 包

官方 README 对 `accelerate`、`wandb`、`deepspeed` 通过 extras 管理。一次性装好，后面微调时就不用再补：

```bash
pip install --extra-index-url https://download.pytorch.org/whl/cu128 -e ".[torch-runtime,finetune]"
```

如果你后面打算用 DeepSpeed ZeRO-3（单卡 96GB 跑 8B 全量微调时有用），把 `finetune` 换成 `finetune-deepspeed`：

```bash
pip install --extra-index-url https://download.pytorch.org/whl/cu128 -e ".[torch-runtime,finetune-deepspeed]"
```

---

## 第 7 步：补装 ModelScope、huggingface_hub、tensorboard

- **ModelScope**：国内镜像，**无代理**下载 HF 权重（速度比境外 HF 快几十倍），用于下 6 个业务模型
- **huggingface_hub**：用于下载 MOSS-Audio-Tokenizer 到 HF 原生缓存
- **hf_transfer**：HF 多线程加速下载，让 MOSS-Audio-Tokenizer 下得更快更稳
- **tensorboard**：训练监控

```bash
pip install -U modelscope huggingface_hub hf_transfer tensorboard
```

### 7.1 把 HF 相关环境变量持久化到 bashrc

一次性写 3 个环境变量进 `~/.bashrc`（进系统镜像，下次开实例自动生效）：

| 变量 | 值 | 作用 |
|---|---|---|
| `HF_HOME` | `/root/hf_cache` | HF 缓存根（默认 `~/.cache/huggingface/` 位置太深，设到系统盘根下一眼可见，且确保 Audio-Tokenizer 进镜像） |
| `HF_HUB_ENABLE_HF_TRANSFER` | `1` | 启用多线程下载加速（`hf_transfer` 包，pip 安装已装） |
| **`HF_ENDPOINT`** | **`https://hf-mirror.com`** | **HF 国内镜像**。解决两件事：① 第 8.1 步下 Audio-Tokenizer 不用代理；② **更重要的**——每次实例重启后 `AutoModel.from_pretrained("OpenMOSS-Team/MOSS-Audio-Tokenizer")` 会发 HEAD 请求到 HF 做 ETag 校验（即使缓存命中），AutoDL 直连 `huggingface.co` 不通就报错；指向镜像后 HEAD 走国内 CDN 永远通 |

```bash
echo 'export HF_HOME=/root/hf_cache' >> ~/.bashrc
echo 'export HF_HUB_ENABLE_HF_TRANSFER=1' >> ~/.bashrc
echo 'export HF_ENDPOINT=https://hf-mirror.com' >> ~/.bashrc
source ~/.bashrc
mkdir -p /root/hf_cache
env | grep -E 'HF_HOME|HF_TRANSFER|HF_ENDPOINT'
```

**期望输出**：

```
HF_HOME=/root/hf_cache
HF_HUB_ENABLE_HF_TRANSFER=1
HF_ENDPOINT=https://hf-mirror.com
```

> 这几条命令只操作 shell 环境变量和目录，**和 conda 环境无关**——无论在 base 还是 moss-tts 下跑都一样。`HF_HOME` / `HF_ENDPOINT` 是全局环境变量，设一次后所有 conda 环境共用。下一步跑 `pip install` / `hf download` 才需要先 `conda activate moss-tts`。
>
> **为什么 `HF_ENDPOINT` 也要持久化？** 第一版文档把它写在下载命令前置（`HF_ENDPOINT=xxx hf download ...`）只让它对一条命令生效，想保持系统干净。后来实测发现：**每次实例重启后，推理 / 验证模型 / 业务模型内部加载 tokenizer 都会触发 HEAD 到 HF**（这是 HuggingFace 客户端的 ETag 校验机制），不持久化就要每次手动 export 一次，体验很差。持久化一条路由环境变量不写入任何账号信息，纯净无副作用。

### 7.2 代理策略说明

本指南**全程不开代理**：

- **所有 HF 流量**走 `HF_ENDPOINT=https://hf-mirror.com`（第 7.1 步已持久化），走国内镜像 CDN
- **下 6 个业务模型时**：ModelScope 在境内，直连即可
- **日常推理 / 训练 / 验证**：业务模型内部对 Audio-Tokenizer 的 HEAD 校验也自动走镜像，网络稳定

因此**不需要** `source /etc/network_turbo`，也**不需要** HuggingFace 账号 / HF_TOKEN —— 只有下 gated 模型或私有仓库时才需要 token，MOSS-TTS 家族全部是公开仓库，用不到。

---

## 第 8 步：下载权重

> 依据：根 README 的 Released Models 表格，以及各 model card 里的 HF/ModelScope 链接。

两类下载：

- **8.1 MOSS-Audio-Tokenizer 走 HF 国内镜像 hf-mirror.com**（必装，无需代理 / 无需 token）→ 落到 `/root/hf_cache`
- **8.2-8.7 业务模型用 ModelScope 下载**（按需，直连即可、速度快）→ 落到 `/root/autodl-tmp/models/`

**强烈建议用 screen 挂后台**，否则下载期间 SSH 断了就功亏一篑：

```bash
screen -S dl
```

```bash
mkdir -p /root/autodl-tmp/models
```

### 8.1 MOSS-Audio-Tokenizer（必装，从 hf-mirror 下载到 HF 缓存）

所有业务模型的 `processor_config.json` 都引用 `OpenMOSS-Team/MOSS-Audio-Tokenizer` 这个 HF 仓库 ID。把这个 repo 下到 `HF_HOME=/root/hf_cache` 后，业务模型加载时内部查 HF 缓存就能命中。

**为什么不直连 huggingface.co？** MOSS-Audio-Tokenizer 的大权重分片走 HF 新的 **Xet CAS 存储后端**（`cas-server.xethub.hf.co`），对未认证请求返回 401 / 对 AutoDL 路由抽风常导致中断。第 7.1 步已经把 `HF_ENDPOINT=https://hf-mirror.com` 持久化到 bashrc，整个下载流量走国内镜像 CDN：不需要 token、不需要代理、速度稳。

```bash
hf download OpenMOSS-Team/MOSS-Audio-Tokenizer
```

> **命令名说明**：新版 `huggingface_hub`（1.0+）把 CLI 从 `huggingface-cli` 改名为 `hf`，本文统一用 `hf`。如果你装的是老版（0.x），命令是 `huggingface-cli download ...`，效果一样。`pip install -U huggingface_hub` 后默认就是新版，建议直接用 `hf`。

下载产物在 `/root/hf_cache/hub/models--OpenMOSS-Team--MOSS-Audio-Tokenizer/`：

```
models--OpenMOSS-Team--MOSS-Audio-Tokenizer/
├── refs/main                  ← HF 真实 commit hash
├── snapshots/<hash>/          ← 文件视图（符号链接到 blobs/）
│   ├── config.json
│   ├── modeling_*.py
│   └── model.safetensors
└── blobs/<sha256>             ← 实际权重文件
```

**验证**（3 项，全过才算真下完）：

```bash
# 1. snapshot 目录存在
ls /root/hf_cache/hub/models--OpenMOSS-Team--MOSS-Audio-Tokenizer/snapshots/

# 2. 没有 .incomplete 半成品（HF 断点续传的遗留标记）
ls /root/hf_cache/hub/models--OpenMOSS-Team--MOSS-Audio-Tokenizer/blobs/ | grep -i incomplete || echo "OK: no incomplete blobs"

# 3. 总大小到位（官方仓库完整约 7+ GB）
du -sh /root/hf_cache/hub/models--OpenMOSS-Team--MOSS-Audio-Tokenizer/
```

期望输出：

```
# 1  一个 40 位 hex 的 commit hash 目录，例如 abc123def456789...
# 2  OK: no incomplete blobs
# 3  7.1G   /root/hf_cache/hub/models--OpenMOSS-Team--MOSS-Audio-Tokenizer/
```

> **如果 2 有 `*.incomplete` 文件或 3 明显小于 7G**：说明镜像下载中途被截断了。
> HF CLI 退出码不会报错（它当作"完成"），但 `AutoModel.from_pretrained` 加载时会再触发续传——就是"为什么刚下过又要下"的现象。
>
> 解决：重跑同一条命令，HF 会自动续传：
>
> ```bash
> hf download OpenMOSS-Team/MOSS-Audio-Tokenizer
> ```
>
> 再跑一遍 2，3 两条验证，直到 `OK: no incomplete blobs` 且 `du -sh ≈ 7.1G`。

> **hf-mirror 偶发不可用的极端兜底**：改走 ModelScope 下 Audio-Tokenizer 到 `/root/models/`（见文末"常见问题"最后一条 Q&A）。

### 8.2 旗舰模型 MOSS-TTS（8B，MossTTSDelay）

```bash
python -c "from modelscope import snapshot_download; snapshot_download('openmoss/MOSS-TTS', local_dir='/root/autodl-tmp/models/MOSS-TTS')"
```

### 8.3 Local-Transformer 版（1.7B，MossTTSLocal）

```bash
python -c "from modelscope import snapshot_download; snapshot_download('openmoss/MOSS-TTS-Local-Transformer', local_dir='/root/autodl-tmp/models/MOSS-TTS-Local-Transformer')"
```

### 8.4 MOSS-TTSD（对话版，8B）

```bash
python -c "from modelscope import snapshot_download; snapshot_download('openmoss/MOSS-TTSD-v1.0', local_dir='/root/autodl-tmp/models/MOSS-TTSD-v1.0')"
```

> ⚠️ **特殊提示**：MOSS-TTSD 用 `n_vq=16`（根 README "Released Models" 表格与 `docs/moss_ttsd_model_card.md` 第 40 行），**不是默认的 32**。微调前要读微调文档里的"MOSS-TTSD 特殊处理"章节。推理时只要用 `AutoModel.from_pretrained('/root/autodl-tmp/models/MOSS-TTSD-v1.0', trust_remote_code=True)` 就能自动从 `config.json` 读到 `n_vq=16`，无需手工干预。

### 8.5 MOSS-VoiceGenerator（1.7B）

```bash
python -c "from modelscope import snapshot_download; snapshot_download('openmoss/MOSS-VoiceGenerator', local_dir='/root/autodl-tmp/models/MOSS-VoiceGenerator')"
```

### 8.6 MOSS-SoundEffect（8B）

```bash
python -c "from modelscope import snapshot_download; snapshot_download('openmoss/MOSS-SoundEffect', local_dir='/root/autodl-tmp/models/MOSS-SoundEffect')"
```

### 8.7 MOSS-TTS-Realtime（1.7B）

```bash
python -c "from modelscope import snapshot_download; snapshot_download('openmoss/MOSS-TTS-Realtime', local_dir='/root/autodl-tmp/models/MOSS-TTS-Realtime')"
```

### 8.8 下载汇总验证

```bash
# 业务模型（数据盘）
du -sh /root/autodl-tmp/models/* 2>/dev/null

# MOSS-Audio-Tokenizer（HF 缓存，系统盘）
du -sh /root/hf_cache/hub/models--OpenMOSS-Team--MOSS-Audio-Tokenizer 2>/dev/null
```

应看到：

```
<大小>  /root/autodl-tmp/models/<ModelName>    (每个你实际下了的业务模型一行)
<大小>  /root/hf_cache/hub/models--OpenMOSS-Team--MOSS-Audio-Tokenizer
```

具体大小随下载源和模型版本有差异，不再列死。只要每项都有非零输出、数值合理（业务模型几 GB 到十几 GB，Tokenizer 几 GB 量级），就算下载完整。

**按需下载指引**：

- 只玩旗舰 TTS → `MOSS-TTS`（8B）；显卡小（< 20 GB VRAM）改下 `MOSS-TTS-Local-Transformer`（1.7B）
- 要做对话 → `MOSS-TTSD-v1.0`
- 要生成新音色 → `MOSS-VoiceGenerator`
- 要做音效 → `MOSS-SoundEffect`
- 要流式实时 TTS → `MOSS-TTS-Realtime`

---

## 💡 保存系统镜像的最佳时机

第 8.1 步跑完（MOSS-Audio-Tokenizer 已落到 `/root/hf_cache`）后，此刻 `/root/` 下已经具备：

- conda `moss-tts` 环境（PyTorch 2.9.1 / transformers 5.0.0 / 等完整依赖）
- MOSS-TTS 源码（editable 安装）
- HF 缓存含 MOSS-Audio-Tokenizer（标准 `hub/models--OpenMOSS-Team--MOSS-Audio-Tokenizer/` 结构）

**此刻就是去 AutoDL 控制台保存系统镜像的最佳时机**。以后新建任意实例只要选这个镜像：

- 开机即有 conda env、MOSS-TTS 源码、HF 缓存
- 只需到 `/root/autodl-tmp/models/` 下按需下业务模型（ModelScope 快速下载）
- 业务模型内部引用 `OpenMOSS-Team/MOSS-Audio-Tokenizer` 自动从 HF 缓存解析命中，**不需要代理**

---

## 第 9 步：完整导入验证

这一步**只验证 config + `trust_remote_code` + processor 文件齐全**，不加载全部模型权重到显存，跑得快。

脚本是**自动扫描型**——无需维护清单：

- 扫 `/root/autodl-tmp/models/` 下所有子目录，按**目录名**和 **config.json 里的 `architectures` 字段**自动识别是 `codec` / `processor` / `realtime` 三种类型之一
- 扫 HF 缓存里是否有 `OpenMOSS-Team/MOSS-Audio-Tokenizer`，有就加入验证
- 只对**确实发现**的模型做验证，没下载的就当没出现过，不显示 SKIP

新下了模型什么都不用改——直接重跑脚本，新模型自动纳入验证。

保存为 `/root/verify_models.py`：

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
MOSS-TTS 家族自动验证脚本
— 自动扫描本地已下载的模型，无需维护清单
— 在 /root/autodl-tmp/models/ 发现什么就验证什么，在 HF 缓存里发现 Audio-Tokenizer 就加入验证
"""

import json
import os
import sys
import traceback
from pathlib import Path

import torch
from transformers import AutoModel, AutoProcessor, AutoTokenizer

# ============================================================================
# 配置
# ============================================================================

BUSINESS_MODELS_ROOT = Path("/root/autodl-tmp/models")   # 业务模型扫描根
HF_AUDIO_TOKENIZER_REPO_ID = "OpenMOSS-Team/MOSS-Audio-Tokenizer"
LEGACY_CODEC_DIR = Path("/root/models/MOSS-Audio-Tokenizer")  # ModelScope 兜底布局

# ============================================================================
# HF 缓存路径解析
# ============================================================================

def resolve_hf_cache_hub() -> Path:
    if "HF_HUB_CACHE" in os.environ:
        return Path(os.environ["HF_HUB_CACHE"])
    if "HF_HOME" in os.environ:
        return Path(os.environ["HF_HOME"]) / "hub"
    return Path.home() / ".cache" / "huggingface" / "hub"

# ============================================================================
# 模型分类（目录名 + 文件特征 + config.json architectures 三重判断）
# ============================================================================

def classify_model(model_dir: Path) -> str:
    """返回 'codec' / 'processor' / 'realtime' / 'unknown'。"""
    if not model_dir.is_dir():
        return "unknown"

    name_lower = model_dir.name.lower()

    # 1. 名字匹配（最直接）
    if "audio-tokenizer" in name_lower or "audio_tokenizer" in name_lower:
        return "codec"
    if "realtime" in name_lower:
        return "realtime"

    # 2. 文件匹配：processor_config.json 是 Delay/Local 架构的专属
    if (model_dir / "processor_config.json").is_file():
        return "processor"

    # 3. config.json 里的 architectures 字段
    cfg = model_dir / "config.json"
    if cfg.is_file():
        try:
            data = json.loads(cfg.read_text(encoding="utf-8"))
            archs = data.get("architectures", []) or []
            if any("Realtime" in a for a in archs):
                return "realtime"
            if any("AudioTokenizer" in a or "Codec" in a for a in archs):
                return "codec"
            if any("MossTTS" in a or "TTS" in a for a in archs):
                return "processor"
        except Exception:
            pass

    return "unknown"

# ============================================================================
# 自动发现
# ============================================================================

def discover_models():
    """返回 (found, unknown):
    found: [(name, path_or_repo_id, kind, location_info), ...]
    unknown: [Path, ...]  无法识别类型的子目录
    """
    found = []
    unknown = []

    # 1. MOSS-Audio-Tokenizer：优先 HF 缓存；没有则看 /root/models/ 兜底
    hf_cache_hub = resolve_hf_cache_hub()
    hf_repo_folder = hf_cache_hub / f"models--{HF_AUDIO_TOKENIZER_REPO_ID.replace('/', '--')}"
    if hf_repo_folder.exists() and any(hf_repo_folder.glob("snapshots/*")):
        found.append(("MOSS-Audio-Tokenizer", HF_AUDIO_TOKENIZER_REPO_ID, "codec",
                      f"HF cache → {hf_repo_folder}"))
    elif LEGACY_CODEC_DIR.is_dir():
        found.append(("MOSS-Audio-Tokenizer", str(LEGACY_CODEC_DIR), "codec",
                      f"本地目录 → {LEGACY_CODEC_DIR}"))

    # 2. 业务模型：扫 /root/autodl-tmp/models/ 下每个子目录
    if BUSINESS_MODELS_ROOT.is_dir():
        for sub in sorted(BUSINESS_MODELS_ROOT.iterdir()):
            if not sub.is_dir() or sub.name.startswith("."):
                continue
            kind = classify_model(sub)
            if kind == "unknown":
                unknown.append(sub)
            else:
                found.append((sub.name, str(sub), kind, f"本地目录 → {sub}"))

    return found, unknown

# ============================================================================
# 主流程
# ============================================================================

def main():
    print(f"torch: {torch.__version__}  cuda: {torch.cuda.is_available()}")
    if torch.cuda.is_available():
        print(f"gpu:   {torch.cuda.get_device_name(0)}")
    print("=" * 64)

    found, unknown = discover_models()

    if not found:
        print()
        print("❌ 没发现任何已下载的 MOSS-TTS 家族模型。")
        print()
        print("请先按部署文档第 8 步下载模型：")
        print("  - MOSS-Audio-Tokenizer：hf download OpenMOSS-Team/MOSS-Audio-Tokenizer")
        print("  - 业务模型：python -c \"from modelscope import snapshot_download; snapshot_download('openmoss/<模型名>', local_dir='/root/autodl-tmp/models/<模型名>')\"")
        return 1

    print()
    print(f"发现 {len(found)} 个模型：")
    for name, _, kind, loc in found:
        print(f"  [{kind:>9}] {name}   ({loc})")

    if unknown:
        print()
        print(f"⚠️  {len(unknown)} 个子目录架构无法识别（已跳过）：")
        for u in unknown:
            print(f"    {u}")

    print()
    print("=" * 64)
    print("开始验证")
    print("=" * 64)

    ok_count = 0
    fail_count = 0

    for name, path, kind, _ in found:
        print(f"\n--- [{kind:>9}] {name}")
        print(f"          路径: {path}")
        try:
            if kind == "codec":
                m = AutoModel.from_pretrained(path, trust_remote_code=True).eval()
                sr = getattr(m, "sample_rate", "(see config)")
                print(f"          [OK]  sample_rate hint = {sr}")
                del m
            elif kind == "processor":
                proc = AutoProcessor.from_pretrained(path, trust_remote_code=True)
                sr = proc.model_config.sampling_rate
                print(f"          [OK]  sampling_rate = {sr} Hz")
                del proc
            elif kind == "realtime":
                # Realtime 架构走独立 API；这里只验证 tokenizer 能加载
                tok = AutoTokenizer.from_pretrained(path)
                print(f"          [OK]  tokenizer vocab_size = {tok.vocab_size}")
                del tok
            ok_count += 1
        except Exception as e:
            print(f"          [FAIL] {type(e).__name__}: {e}")
            traceback.print_exc()
            fail_count += 1

    print()
    print("=" * 64)
    print(f"汇总：OK={ok_count}  FAIL={fail_count}  （已发现 {len(found)} 个模型）")
    print("=" * 64)
    return 0 if fail_count == 0 else 1

if __name__ == "__main__":
    sys.exit(main())
```

运行：

```bash
python /root/verify_models.py
```

期望输出（假设你已下了 Audio-Tokenizer 和 MOSS-TTS-Local-Transformer）：

```
torch: 2.9.1+cu128  cuda: True
gpu:   NVIDIA RTX PRO 6000
================================================================

发现 2 个模型：
  [    codec] MOSS-Audio-Tokenizer   (HF cache → /root/hf_cache/hub/models--OpenMOSS-Team--MOSS-Audio-Tokenizer)
  [processor] MOSS-TTS-Local-Transformer   (本地目录 → /root/autodl-tmp/models/MOSS-TTS-Local-Transformer)

================================================================
开始验证
================================================================

--- [    codec] MOSS-Audio-Tokenizer
          路径: OpenMOSS-Team/MOSS-Audio-Tokenizer
          [OK]  sample_rate hint = (see config)

--- [processor] MOSS-TTS-Local-Transformer
          路径: /root/autodl-tmp/models/MOSS-TTS-Local-Transformer
          [OK]  sampling_rate = 24000 Hz

================================================================
汇总：OK=2  FAIL=0  （已发现 2 个模型）
================================================================
```

**`OK` 数等于你下载的模型数（含 Tokenizer），`FAIL=0` 才算部署成功**。

---

## 第 10 步：推理冒烟测试（按需运行）

和 `verify_models.py` 同款思路：脚本**自动扫描**下载了什么模型，就对什么模型跑冒烟。每个模型一个独立函数，按目录名派发；找不到 `MOSS-Audio-Tokenizer`（HF 缓存或本地兜底）时直接退出，因为业务模型都依赖它。

保存为 `/root/smoke_test.py`：

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
MOSS-TTS 家族推理冒烟测试脚本
— 自动扫描本地已下载的业务模型，按目录名派发对应的冒烟函数
— 输出 wav 保存到 /root/autodl-tmp/smoke/
"""

import importlib.util
import json
import os
import sys
import traceback
from pathlib import Path

import torch
import torchaudio
from transformers import AutoModel, AutoProcessor, AutoTokenizer

# 官方要求：禁用坏掉的 cuDNN SDPA 后端（根 README 237-241 行）
torch.backends.cuda.enable_cudnn_sdp(False)
torch.backends.cuda.enable_flash_sdp(True)
torch.backends.cuda.enable_mem_efficient_sdp(True)
torch.backends.cuda.enable_math_sdp(True)

# ============================================================================
# 共享配置
# ============================================================================

CODEC_ID = "OpenMOSS-Team/MOSS-Audio-Tokenizer"   # HF 仓库 ID；由 HF 缓存（/root/hf_cache）解析
LEGACY_CODEC_DIR = Path("/root/models/MOSS-Audio-Tokenizer")  # ModelScope 兜底布局
BUSINESS_MODELS_ROOT = Path("/root/autodl-tmp/models")
OUT_DIR = Path("/root/autodl-tmp/smoke")
OUT_DIR.mkdir(parents=True, exist_ok=True)

DEVICE = "cuda" if torch.cuda.is_available() else "cpu"
DTYPE = torch.bfloat16 if DEVICE == "cuda" else torch.float32


def resolve_attn() -> str:
    """官方 resolve_attn_implementation 写法（见 README 249-269 行）"""
    if (
        DEVICE == "cuda"
        and importlib.util.find_spec("flash_attn") is not None
        and DTYPE in {torch.float16, torch.bfloat16}
    ):
        major, _ = torch.cuda.get_device_capability()
        if major >= 8:
            return "flash_attention_2"
    return "sdpa" if DEVICE == "cuda" else "eager"


ATTN = resolve_attn()
print(f"device={DEVICE}  dtype={DTYPE}  attn={ATTN}")


def resolve_hf_cache_hub() -> Path:
    if "HF_HUB_CACHE" in os.environ:
        return Path(os.environ["HF_HUB_CACHE"])
    if "HF_HOME" in os.environ:
        return Path(os.environ["HF_HOME"]) / "hub"
    return Path.home() / ".cache" / "huggingface" / "hub"


def resolve_codec_id_or_path() -> str:
    """优先用 HF 缓存里的 Audio-Tokenizer repo id；没有则回退到 /root/models 本地目录。"""
    hf_cache_hub = resolve_hf_cache_hub()
    hf_repo_folder = hf_cache_hub / f"models--{CODEC_ID.replace('/', '--')}"
    if hf_repo_folder.exists() and any(hf_repo_folder.glob("snapshots/*")):
        return CODEC_ID
    if LEGACY_CODEC_DIR.is_dir():
        return str(LEGACY_CODEC_DIR)
    return ""   # 调用方负责报错


# ============================================================================
# 模型分类（和 verify_models.py 一致的启发式）
# ============================================================================

def classify_model(model_dir: Path) -> str:
    """返回 'codec' / 'processor' / 'realtime' / 'unknown'。"""
    if not model_dir.is_dir():
        return "unknown"

    name_lower = model_dir.name.lower()
    if "audio-tokenizer" in name_lower or "audio_tokenizer" in name_lower:
        return "codec"
    if "realtime" in name_lower:
        return "realtime"
    if (model_dir / "processor_config.json").is_file():
        return "processor"

    cfg = model_dir / "config.json"
    if cfg.is_file():
        try:
            data = json.loads(cfg.read_text(encoding="utf-8"))
            archs = data.get("architectures", []) or []
            if any("Realtime" in a for a in archs):
                return "realtime"
            if any("AudioTokenizer" in a or "Codec" in a for a in archs):
                return "codec"
            if any("MossTTS" in a or "TTS" in a for a in archs):
                return "processor"
        except Exception:
            pass
    return "unknown"


# ============================================================================
# 通用函数：Delay/Local 架构的 generate + save
# （MOSS-TTS / MOSS-TTS-Local / MOSS-TTSD / MOSS-VoiceGenerator / MOSS-SoundEffect 都走这套 API）
# ============================================================================

def _run_delay_family(model_dir: str, user_message_kwargs: dict, out_name: str,
                      gen_kwargs: dict = None):
    """加载 Delay/Local 族模型 → 生成 → 保存 wav"""
    print(f"\n[{out_name}] 加载 {model_dir} ...")
    proc = AutoProcessor.from_pretrained(model_dir, trust_remote_code=True)
    proc.audio_tokenizer = proc.audio_tokenizer.to(DEVICE)

    model = AutoModel.from_pretrained(
        model_dir, trust_remote_code=True,
        attn_implementation=ATTN, torch_dtype=DTYPE,
    ).to(DEVICE).eval()

    print(f"[{out_name}] 加载完成，开始生成...")
    conversation = [[proc.build_user_message(**user_message_kwargs)]]
    batch = proc(conversation, mode="generation")

    gk = {"max_new_tokens": 4096}
    if gen_kwargs:
        gk.update(gen_kwargs)

    with torch.no_grad():
        out = model.generate(
            input_ids=batch["input_ids"].to(DEVICE),
            attention_mask=batch["attention_mask"].to(DEVICE),
            **gk,
        )

    msg = proc.decode(out)[0]
    audio = msg.audio_codes_list[0]
    out_path = OUT_DIR / out_name
    torchaudio.save(str(out_path), audio.unsqueeze(0), proc.model_config.sampling_rate)
    print(f"[{out_name}] OK → {out_path}  shape={tuple(audio.shape)}")

    del model, proc
    if DEVICE == "cuda":
        torch.cuda.empty_cache()


# ============================================================================
# 每个业务模型的冒烟函数；签名统一为 (model_dir: str) -> None
# ============================================================================

def smoke_moss_tts(model_dir: str):
    """MOSS-TTS 旗舰 8B：纯文本合成（无克隆）"""
    _run_delay_family(
        model_dir=model_dir,
        user_message_kwargs={"text": "这是 MOSS-TTS 旗舰版在 AutoDL 上的冒烟测试。"},
        out_name="moss_tts.wav",
    )


def smoke_moss_tts_local(model_dir: str):
    """MOSS-TTS-Local-Transformer 1.7B：同样 API，架构不同"""
    _run_delay_family(
        model_dir=model_dir,
        user_message_kwargs={"text": "这是 MOSS-TTS Local Transformer 版本的冒烟测试。"},
        out_name="moss_tts_local.wav",
    )


def smoke_moss_ttsd(model_dir: str):
    """MOSS-TTSD-v1.0 8B：多人对话，[S1]/[S2] 说话人标签（官方 quickstart 格式）"""
    text = "[S1] 你好，这是 MOSS-TTSD 的冒烟测试。[S2] 是的，多人对话也能正常生成。"
    _run_delay_family(
        model_dir=model_dir,
        user_message_kwargs={"text": text},
        out_name="moss_ttsd.wav",
    )


def smoke_moss_voice_generator(model_dir: str):
    """MOSS-VoiceGenerator 1.7B：text + instruction（自然语言描述音色）"""
    _run_delay_family(
        model_dir=model_dir,
        user_message_kwargs={
            "text": "This is a voice generated by MOSS VoiceGenerator smoke test.",
            "instruction": "A warm, friendly female voice speaking clearly at a moderate pace.",
        },
        out_name="moss_voice_generator.wav",
    )


def smoke_moss_sound_effect(model_dir: str):
    """MOSS-SoundEffect 8B：ambient_sound 字段（场景描述）"""
    _run_delay_family(
        model_dir=model_dir,
        user_message_kwargs={
            "ambient_sound": "Rolling thunder with steady rainfall in a forest.",
            "tokens": 160,   # 约 12.8 秒
        },
        out_name="moss_sound_effect.wav",
    )


def smoke_moss_tts_realtime(model_dir: str):
    """MOSS-TTS-Realtime 1.7B：独立 API，走 mossttsrealtime.MossTTSRealtime"""
    print(f"\n[moss_tts_realtime] 加载 {model_dir} ...")

    codec_ref = resolve_codec_id_or_path()
    if not codec_ref:
        raise RuntimeError("未找到 MOSS-Audio-Tokenizer（HF 缓存和 /root/models 均无）")

    # Realtime 的运行时代码在源码目录里，需要 sys.path 插一下
    sys.path.insert(0, "/root/MOSS-TTS/moss_tts_realtime")
    from mossttsrealtime.modeling_mossttsrealtime import MossTTSRealtime
    from inferencer import MossTTSRealtimeInference

    # Realtime 官方注释：local transformer 用 StaticCache，与 FA2 不兼容，只能 sdpa
    model = MossTTSRealtime.from_pretrained(
        model_dir, attn_implementation="sdpa", torch_dtype=DTYPE
    ).to(DEVICE)
    tok = AutoTokenizer.from_pretrained(model_dir)
    codec = AutoModel.from_pretrained(codec_ref, trust_remote_code=True).eval().to(DEVICE)

    inf = MossTTSRealtimeInference(
        model, tok, max_length=5000, codec=codec, codec_sample_rate=24000,
        codec_encode_kwargs={"chunk_duration": 8},
    )

    text = ["MOSS-TTS Realtime streaming TTS smoke test running on AutoDL."]
    reference_audio = ["/root/MOSS-TTS/moss_tts_realtime/audio/prompt_audio1.mp3"]

    res = inf.generate(
        text=text, reference_audio_path=reference_audio,
        temperature=0.8, top_p=0.6, top_k=30,
        repetition_penalty=1.1, repetition_window=50,
        device=DEVICE,
    )

    for i, gen in enumerate(res):
        out_audio = torch.tensor(gen).to(DEVICE).permute(1, 0)
        wav = codec.decode(out_audio, chunk_duration=8)["audio"][0].cpu()
        if wav.ndim == 1:
            wav = wav.unsqueeze(0)
        out_path = OUT_DIR / f"moss_tts_realtime_{i}.wav"
        torchaudio.save(str(out_path), wav, 24000)
        print(f"[moss_tts_realtime] OK → {out_path}")

    del model, tok, codec, inf
    if DEVICE == "cuda":
        torch.cuda.empty_cache()


# ============================================================================
# 目录名 → 冒烟函数 的派发表
# —  key 是小写后匹配的子串；按顺序匹配，第一个命中的规则生效
# ============================================================================

DISPATCH_RULES = [
    # (substring_of_dirname_lower, fn, human_label)
    ("moss-ttsd",            smoke_moss_ttsd,            "MOSS-TTSD"),
    ("moss-tts-local",       smoke_moss_tts_local,       "MOSS-TTS-Local"),
    ("moss-tts-realtime",    smoke_moss_tts_realtime,    "MOSS-TTS-Realtime"),
    ("moss-voicegenerator",  smoke_moss_voice_generator, "MOSS-VoiceGenerator"),
    ("moss-soundeffect",     smoke_moss_sound_effect,    "MOSS-SoundEffect"),
    ("moss-tts",             smoke_moss_tts,             "MOSS-TTS"),   # 放最后兜底
]


def dispatch(model_dir: Path):
    """根据目录名找到对应的冒烟函数；找不到返回 None。"""
    name_lower = model_dir.name.lower()
    for needle, fn, label in DISPATCH_RULES:
        if needle in name_lower:
            return fn, label
    return None, None


# ============================================================================
# 自动发现 + 派发
# ============================================================================

def discover_targets():
    """返回 [(path, fn, label), ...]，按目录名排序；只含业务模型（不含 codec）。"""
    targets = []
    if not BUSINESS_MODELS_ROOT.is_dir():
        return targets
    for sub in sorted(BUSINESS_MODELS_ROOT.iterdir()):
        if not sub.is_dir() or sub.name.startswith("."):
            continue
        kind = classify_model(sub)
        if kind == "codec" or kind == "unknown":
            continue
        fn, label = dispatch(sub)
        if fn is None:
            print(f"[skip] {sub.name}：已识别为 {kind}，但没有匹配的冒烟函数")
            continue
        targets.append((sub, fn, label))
    return targets


def main():
    # Audio-Tokenizer 是所有业务模型的依赖，先探一下
    codec_ref = resolve_codec_id_or_path()
    if not codec_ref:
        print("❌ 未找到 MOSS-Audio-Tokenizer（HF 缓存和 /root/models 均无）")
        print("   业务模型在加载时会去调 AudioTokenizer，没有它没法冒烟。")
        print("   请先执行部署文档第 8.1 步下载 MOSS-Audio-Tokenizer。")
        return 1
    print(f"Audio-Tokenizer: {codec_ref}")

    targets = discover_targets()
    if not targets:
        print()
        print("❌ 在 /root/autodl-tmp/models/ 下没发现任何已知的 MOSS-TTS 家族业务模型。")
        print("   请先按部署文档第 8 步下载至少一个业务模型。")
        return 1

    print()
    print(f"发现 {len(targets)} 个可冒烟的模型：")
    for path, _, label in targets:
        print(f"  - {label:<22}  ← {path}")

    print()
    print("=" * 64)
    print("开始冒烟")
    print("=" * 64)

    ok, fail = 0, 0
    for path, fn, label in targets:
        print(f"\n====== [{label}] ======")
        try:
            fn(str(path))
            ok += 1
        except Exception as e:
            print(f"[{label}] [FAIL] {type(e).__name__}: {e}")
            traceback.print_exc()
            fail += 1
            # 失败后继续跑下一个，避免一个模型废了所有模型的机会
            if DEVICE == "cuda":
                torch.cuda.empty_cache()

    print()
    print("=" * 64)
    print(f"汇总：OK={ok}  FAIL={fail}  （总计 {len(targets)} 个模型）")
    print(f"输出目录：{OUT_DIR}")
    print("=" * 64)
    return 0 if fail == 0 else 1


if __name__ == "__main__":
    sys.exit(main())
```

运行：

```bash
cd /root/MOSS-TTS          # Realtime 依赖源码里的 inferencer.py，cd 到这不会错
python /root/smoke_test.py
```

期望输出（假设你下了 Audio-Tokenizer 和 MOSS-TTS-Local-Transformer）：

```
device=cuda  dtype=torch.bfloat16  attn=flash_attention_2
Audio-Tokenizer: OpenMOSS-Team/MOSS-Audio-Tokenizer

发现 1 个可冒烟的模型：
  - MOSS-TTS-Local         ← /root/autodl-tmp/models/MOSS-TTS-Local-Transformer

================================================================
开始冒烟
================================================================

====== [MOSS-TTS-Local] ======

[moss_tts_local.wav] 加载 /root/autodl-tmp/models/MOSS-TTS-Local-Transformer ...
[moss_tts_local.wav] 加载完成，开始生成...
[moss_tts_local.wav] OK → /root/autodl-tmp/smoke/moss_tts_local.wav  shape=(N,)

================================================================
汇总：OK=1  FAIL=0  （总计 1 个模型）
输出目录：/root/autodl-tmp/smoke
================================================================
```

**耗时预期**：

- 8B 模型（MOSS-TTS / MOSS-TTSD / MOSS-SoundEffect）：首次加载 + 生成约 60–120 秒
- 1.7B 模型（Local / VoiceGenerator / Realtime）：首次加载 + 生成约 20–40 秒

用 WinSCP 把 `/root/autodl-tmp/smoke/` 下的 wav 下到本地听，应该都是清晰的目标语音 / 音效 / 对话。

### ⚠️ 修改脚本的注意事项

1. **新增 / 改名业务模型**：什么都不用改，直接重跑脚本，自动按目录名匹配派发表。
2. **派发规则没命中**：脚本会打印 `[skip] <name>：已识别为 processor，但没有匹配的冒烟函数` — 在 `DISPATCH_RULES` 里加一行即可。
3. **修改生成内容 / 语种**：改 `smoke_xxx()` 里 `user_message_kwargs` 的 `text` / `instruction` / `ambient_sound` / `tokens`。
4. **8B 模型爆显存**：临时把脚本开头 `ATTN = resolve_attn()` 改成 `ATTN = "sdpa"`；或者关掉其它 Python 进程。
5. **只想跑某一个模型**：把它单独挪到 `/root/autodl-tmp/models/` 之外的临时路径，或者在 `discover_targets()` 里加个过滤条件（不建议）。

---

## 第 11 步：部署交付脚本和使用指南

推理冒烟测试通过后，将以下 4 个文件通过 WinSCP 上传到 AutoDL 的 `/root/` 目录下：

- `1-run-webui.sh` — 5 个 Gradio WebUI 启动脚本（5 个模型任选其一）
- `2-run-train.sh` — 微调训练启动脚本（覆盖 3 套架构）
- `3-run-infer.sh` — 微调后推理脚本
- `MOSS-TTS使用指南.txt` — 日常使用速查手册

上传后执行权限和换行符修复：

```bash
chmod +x /root/1-run-webui.sh /root/2-run-train.sh /root/3-run-infer.sh
sed -i 's/\r$//' /root/1-run-webui.sh /root/2-run-train.sh /root/3-run-infer.sh
```

所有脚本会自动激活 moss-tts conda 环境，不需要手动切换。

> **端口策略**：AutoDL "自定义服务"默认放行 6006。MOSS-TTS 仓库里 5 个 Gradio demo 的默认端口不同（`moss_tts_app.py`=7860、`moss_sound_effect_app.py`=7861、`moss_voice_generator_app.py`=7862、`moss_ttsd_app.py`=7863、`moss_tts_realtime/app.py`=18082，来自各脚本 `argparse` 的 default 值）。下面的启动脚本**统一用 `--port 6006` 覆盖**，方便 AutoDL 浏览器一键打开。5 个 demo 同时只能起一个。

---

### 11.1 WebUI 启动脚本（1-run-webui.sh）

MOSS-TTS 家族覆盖 6 个业务模型（其中选项 1 和 2 共享 `clis/moss_tts_app.py` 入口，API 完全兼容，仅换 `--model_path`）：

| 选项 | 入口文件 | 用途 | 默认端口 |
|---|---|---|---:|
| 1) MOSS-TTS 推理（旗舰 8B） | `clis/moss_tts_app.py` | 旗舰 TTS（声音克隆、时长控制、多语种） | 7860 |
| 2) MOSS-TTS-Local-Transformer 推理（1.7B） | `clis/moss_tts_app.py` | 旗舰 TTS 的轻量版架构（显卡小时首选，API 与 1 完全一致） | 7860 |
| 3) MOSS-TTSD 推理 | `clis/moss_ttsd_app.py` | 多人对话 TTS（需要 `--codec_path`） | 7863 |
| 4) MOSS-VoiceGenerator 推理 | `clis/moss_voice_generator_app.py` | 自然语言描述生成音色 | 7862 |
| 5) MOSS-SoundEffect 推理 | `clis/moss_sound_effect_app.py` | 音效生成 | 7861 |
| 6) MOSS-TTS-Realtime 推理 | `moss_tts_realtime/app.py` | 流式实时 TTS | 18082 |

脚本统一端口为 **6006**（AutoDL 预开放端口）。

```bash
#!/bin/bash
# MOSS-TTS 家族 WebUI 一键启动脚本（交互式菜单版）
#
# 用法:
#   bash /root/1-run-webui.sh

# 激活 moss-tts conda 环境
eval "$(conda shell.bash hook 2>/dev/null)"
conda activate moss-tts

echo "=========================================="
echo "   MOSS-TTS 家族 WebUI 启动器"
echo "=========================================="
echo ""
echo "请选择要启动的界面:"
echo "  1) MOSS-TTS                    (旗舰 TTS / 声音克隆 / 多语种, MossTTSDelay 8B)"
echo "  2) MOSS-TTS-Local-Transformer  (旗舰 TTS 的 MossTTSLocal 1.7B 轻量版，小显卡首选)"
echo "  3) MOSS-TTSD                   (多人对话 TTS, 8B, n_vq=16)"
echo "  4) MOSS-VoiceGenerator         (自然语言描述生成音色, 1.7B)"
echo "  5) MOSS-SoundEffect            (音效生成, 8B)"
echo "  6) MOSS-TTS-Realtime           (流式实时 TTS, 1.7B)"
echo ""
read -p "请输入编号 [默认 1]: " CHOICE
CHOICE="${CHOICE:-1}"

cd /root/MOSS-TTS

# 所有脚本都要求 trust_remote_code，官方在根 README 的推理示例里也强制禁用
# cuDNN SDPA——clis/*.py 的前几行已经帮你做好，这里直接 exec。

case "$CHOICE" in
    1)
        MODEL_PATH="/root/autodl-tmp/models/MOSS-TTS"
        [ ! -d "$MODEL_PATH" ] && { echo "❌ 模型目录不存在: $MODEL_PATH"; exit 1; }
        echo ""
        echo ">>> 启动 MOSS-TTS Gradio WebUI"
        echo ">>> 模型: $MODEL_PATH"
        echo ">>> 端口: 6006"
        echo ""
        python clis/moss_tts_app.py \
            --model_path "$MODEL_PATH" \
            --port 6006 --host 0.0.0.0
        ;;

    2)
        # MOSS-TTS-Local-Transformer：MossTTSLocal 架构 1.7B 版本
        # 和选项 1 (MOSS-TTS 旗舰 8B) 用同一个入口脚本 clis/moss_tts_app.py
        # 两者 API 完全兼容，AutoProcessor/AutoModel + trust_remote_code 会自动
        # 根据 config.json 里的架构字段加载对应实现。仅需换 --model_path。
        MODEL_PATH="/root/autodl-tmp/models/MOSS-TTS-Local-Transformer"
        [ ! -d "$MODEL_PATH" ] && { echo "❌ 模型目录不存在: $MODEL_PATH"; exit 1; }
        echo ""
        echo ">>> 启动 MOSS-TTS-Local-Transformer Gradio WebUI"
        echo ">>> 模型: $MODEL_PATH"
        echo ">>> 端口: 6006"
        echo ""
        python clis/moss_tts_app.py \
            --model_path "$MODEL_PATH" \
            --port 6006 --host 0.0.0.0
        ;;

    3)
        MODEL_PATH="/root/autodl-tmp/models/MOSS-TTSD-v1.0"
        CODEC_ID="OpenMOSS-Team/MOSS-Audio-Tokenizer"   # HF 仓库 ID；由 HF 缓存（/root/hf_cache）解析
        [ ! -d "$MODEL_PATH" ] && { echo "❌ 模型目录不存在: $MODEL_PATH"; exit 1; }
        echo ""
        echo ">>> 启动 MOSS-TTSD Gradio WebUI"
        echo ">>> 模型:   $MODEL_PATH"
        echo ">>> 分词器: $CODEC_ID (HF ID, 从 HF 缓存解析)"
        echo ">>> 端口:   6006"
        echo ""
        python clis/moss_ttsd_app.py \
            --model_path "$MODEL_PATH" \
            --codec_path "$CODEC_ID" \
            --port 6006 --host 0.0.0.0
        ;;

    4)
        MODEL_PATH="/root/autodl-tmp/models/MOSS-VoiceGenerator"
        [ ! -d "$MODEL_PATH" ] && { echo "❌ 模型目录不存在: $MODEL_PATH"; exit 1; }
        echo ""
        echo ">>> 启动 MOSS-VoiceGenerator Gradio WebUI"
        echo ">>> 模型: $MODEL_PATH"
        echo ">>> 端口: 6006"
        echo ""
        python clis/moss_voice_generator_app.py \
            --model_path "$MODEL_PATH" \
            --port 6006 --host 0.0.0.0
        ;;

    5)
        MODEL_PATH="/root/autodl-tmp/models/MOSS-SoundEffect"
        [ ! -d "$MODEL_PATH" ] && { echo "❌ 模型目录不存在: $MODEL_PATH"; exit 1; }
        echo ""
        echo ">>> 启动 MOSS-SoundEffect Gradio WebUI"
        echo ">>> 模型: $MODEL_PATH"
        echo ">>> 端口: 6006"
        echo ""
        python clis/moss_sound_effect_app.py \
            --model_path "$MODEL_PATH" \
            --port 6006 --host 0.0.0.0
        ;;

    6)
        MODEL_PATH="/root/autodl-tmp/models/MOSS-TTS-Realtime"
        CODEC_ID="OpenMOSS-Team/MOSS-Audio-Tokenizer"   # HF 仓库 ID；由 HF 缓存（/root/hf_cache）解析
        [ ! -d "$MODEL_PATH" ] && { echo "❌ 模型目录不存在: $MODEL_PATH"; exit 1; }
        # moss_tts_realtime/app.py 里 import 了同目录下的 mossttsrealtime/
        # 以及使用了 ./audio/prompt_audio*.mp3 作为默认 reference——必须 cd 过去
        cd /root/MOSS-TTS/moss_tts_realtime
        echo ""
        echo ">>> 启动 MOSS-TTS-Realtime Gradio WebUI"
        echo ">>> 模型:   $MODEL_PATH"
        echo ">>> 分词器: $CODEC_ID (HF ID, 从 HF 缓存解析)"
        echo ">>> 端口:   6006"
        echo ""
        # 官方注释：推荐 sdpa（配合 torch.compile），不是 flash_attention_2
        # 因为 realtime 的 local transformer 用 StaticCache 与 FA2 不兼容
        python app.py \
            --model_path "$MODEL_PATH" \
            --tokenizer_path "$MODEL_PATH" \
            --codec_model_path "$CODEC_ID" \
            --attn_implementation sdpa \
            --port 6006 --host 0.0.0.0
        ;;

    *)
        echo "❌ 无效选择: $CHOICE"
        exit 1
        ;;
esac

echo ""
echo '------------------------ WebUI 已退出'
```

---

### 11.2 微调训练脚本（2-run-train.sh）

MOSS-TTS 家族一共有 **3 套微调 pipeline**（来自仓库 3 个 `finetuning/` 目录），复用关系如下：

| 脚本目录 | 适配模型 | 数据字段 |
|---|---|---|
| `moss_tts_delay/finetuning/` | MOSS-TTS / MOSS-TTSD / MOSS-VoiceGenerator / MOSS-SoundEffect（全部 MossTTSDelay） | `audio` + (`text`/`ref_audio`/`reference`/`instruction`/`ambient_sound`) |
| `moss_tts_local/finetuning/` | MOSS-TTS-Local-Transformer | 与 delay 完全一致 |
| `moss_tts_realtime/finetuning/` | MOSS-TTS-Realtime | 多轮对话格式 `conversations[].role/text/wav` + 可选 `ref_wav` |

每套都包含 `prepare_data.py` + `sft.py` + `run_train.sh` 三件套，并在 `finetuning/configs/` 下自带 accelerate YAML（ddp_8gpu / fsdp / zero3）。

**本机是单卡 96GB**——对 1.7B 模型（Local / VoiceGenerator / Realtime）直接单卡全量微调；对 8B 模型（MOSS-TTS / MOSS-TTSD / MOSS-SoundEffect）也能单卡全量，前提是开 bf16 + gradient_checkpointing，必要时叠加 ZeRO-3 CPU offload。这套 sh 封装会根据选择自动注入正确的 accelerate 参数。

脚本数据目录约定：`/root/autodl-tmp/datasets/`，里面放：
- `train_raw.jsonl`（原始清单，详见微调文档）
- 一次只跑一个任务，选完后 `run_train.sh` 会自动调对应的 `finetuning/` 目录。

```bash
#!/bin/bash
# MOSS-TTS 家族通用微调启动脚本（交互式菜单版）
# 支持: MOSS-TTS / MOSS-TTSD / MOSS-VoiceGenerator / MOSS-SoundEffect / MOSS-TTS-Local-Transformer / MOSS-TTS-Realtime

eval "$(conda shell.bash hook 2>/dev/null)"
conda activate moss-tts

datasets_DIR="/root/autodl-tmp/datasets"
OUTPUT_ROOT="/root/autodl-tmp/checkpoints"
RAW_JSONL_DEFAULT="${datasets_DIR}/train_raw.jsonl"
PREPARED_JSONL_DEFAULT="${datasets_DIR}/train_with_codes.jsonl"

echo "=========================================="
echo "   MOSS-TTS 家族微调训练启动器"
echo "=========================================="
echo ""

# ---------- 检查数据目录 ----------
if [ ! -d "$datasets_DIR" ]; then
    echo "❌ 数据目录不存在: $datasets_DIR"
    echo "   请先按微调文档准备数据，或手动创建:"
    echo "       mkdir -p $datasets_DIR"
    exit 1
fi

if [ ! -f "$RAW_JSONL_DEFAULT" ] && [ ! -f "$PREPARED_JSONL_DEFAULT" ]; then
    echo "❌ 找不到 $RAW_JSONL_DEFAULT（或已预处理的 $PREPARED_JSONL_DEFAULT）"
    exit 1
fi

echo "请选择要微调的模型:"
echo "  1) MOSS-TTS               (MossTTSDelay 8B, 默认全量微调)"
echo "  2) MOSS-TTS-Local         (MossTTSLocal 1.7B, 全量微调)"
echo "  3) MOSS-TTSD              (MossTTSDelay 8B, n_vq=16 特殊)"
echo "  4) MOSS-VoiceGenerator    (MossTTSDelay 1.7B, 全量微调)"
echo "  5) MOSS-SoundEffect       (MossTTSDelay 8B, 全量微调)"
echo "  6) MOSS-TTS-Realtime      (MossTTSRealtime 1.7B, 全量微调)"
echo ""
read -p "请输入编号 [默认 1]: " CHOICE
CHOICE="${CHOICE:-1}"

case "$CHOICE" in
    1)
        ARCH_DIR="moss_tts_delay"
        MODEL_PATH="/root/autodl-tmp/models/MOSS-TTS"
        IS_8B=1 ; N_VQ_ARG=""
        TRAIN_NAME="moss_tts_sft"
        ;;
    2)
        ARCH_DIR="moss_tts_local"
        MODEL_PATH="/root/autodl-tmp/models/MOSS-TTS-Local-Transformer"
        IS_8B=0 ; N_VQ_ARG=""
        TRAIN_NAME="moss_tts_local_sft"
        ;;
    3)
        ARCH_DIR="moss_tts_delay"
        MODEL_PATH="/root/autodl-tmp/models/MOSS-TTSD-v1.0"
        IS_8B=1 ; N_VQ_ARG="--n-vq 16"
        TRAIN_NAME="moss_ttsd_sft"
        echo ""
        echo "⚠️  MOSS-TTSD 需要 n_vq=16，脚本已自动注入 --n-vq 16。"
        echo "    另请注意：如果要拿 MOSS-TTSD 做下游训练，"
        echo "    官方建议你将 moss_tts_delay/ 下的 4 个 .py 文件"
        echo "    （processing_moss_tts.py / modeling_moss_tts.py /"
        echo "     configuration_moss_tts.py / inference_utils.py）"
        echo "    替换为 HF 仓库 OpenMOSS-Team/MOSS-TTSD-v1.0 中对应文件，"
        echo "    否则 loss 正常但推理会胡言乱语。"
        echo "    详见 moss_tts_delay/finetuning/README.md §2.2。"
        echo ""
        ;;
    4)
        ARCH_DIR="moss_tts_delay"
        MODEL_PATH="/root/autodl-tmp/models/MOSS-VoiceGenerator"
        IS_8B=0 ; N_VQ_ARG=""
        TRAIN_NAME="moss_voice_generator_sft"
        ;;
    5)
        ARCH_DIR="moss_tts_delay"
        MODEL_PATH="/root/autodl-tmp/models/MOSS-SoundEffect"
        IS_8B=1 ; N_VQ_ARG=""
        TRAIN_NAME="moss_sound_effect_sft"
        ;;
    6)
        ARCH_DIR="moss_tts_realtime"
        MODEL_PATH="/root/autodl-tmp/models/MOSS-TTS-Realtime"
        IS_8B=0 ; N_VQ_ARG=""
        TRAIN_NAME="moss_tts_realtime_sft"
        ;;
    *)
        echo "❌ 无效选择: $CHOICE"; exit 1 ;;
esac

CODEC_PATH="OpenMOSS-Team/MOSS-Audio-Tokenizer"   # HF 仓库 ID；由 HF 缓存（/root/hf_cache）解析，sft.py 会原样传给 AutoModel.from_pretrained
OUTPUT_DIR="${OUTPUT_ROOT}/${TRAIN_NAME}"
mkdir -p "$OUTPUT_DIR"

# ---------- 根据模型大小选 accelerate 配置 ----------
# 单卡场景：不走 DDP；若是 8B 走 ZeRO-3（单卡 + CPU offload 可选）
# 1.7B 直接 accelerate launch 不带 config 即可
if [ "$IS_8B" = "1" ]; then
    ACC_BASE="accelerate launch --num_processes 1 --mixed_precision bf16"
    # 单卡 96GB 跑 8B 默认 bf16 + gradient_checkpointing 基本能塞下
    # 显存紧张时可把下一行替换成:
    #   ACC_BASE="accelerate launch --config_file ${ARCH_DIR}/finetuning/configs/accelerate_zero3_8b.yaml"
    # 并把 yaml 里的 num_processes 改 1、加 offload_param_device: cpu
else
    ACC_BASE="accelerate launch --num_processes 1 --mixed_precision bf16"
fi

# ---------- 训练超参 ----------
TRAIN_ARGS="--per-device-batch-size 1 --gradient-accumulation-steps 8 \
--learning-rate 1e-5 --warmup-ratio 0.03 --num-epochs 3 \
--mixed-precision bf16 --gradient-checkpointing"

# delay / local 都有 channelwise-loss-weight，realtime 没有（见 finetuning/README.md §4.4）
if [ "$ARCH_DIR" != "moss_tts_realtime" ]; then
    TRAIN_ARGS="$TRAIN_ARGS --channelwise-loss-weight 1,32"
fi

# ---------- 展示配置 ----------
echo ""
echo "=========================================="
echo ">>> 训练配置确认"
echo "=========================================="
echo "  架构目录:   $ARCH_DIR"
echo "  基础模型:   $MODEL_PATH"
echo "  分词器:     $CODEC_PATH"
echo "  原始 JSONL: $RAW_JSONL_DEFAULT"
echo "  预处理到:   $PREPARED_JSONL_DEFAULT"
echo "  输出目录:   $OUTPUT_DIR"
if [ -n "$N_VQ_ARG" ]; then
    echo "  特殊参数:   $N_VQ_ARG"
fi
echo "  训练超参:   $TRAIN_ARGS"
echo "=========================================="
echo ""
read -p "确认启动训练? [y/N]: " CONFIRM
[[ ! "${CONFIRM:-N}" =~ ^[Yy]$ ]] && { echo "已取消"; exit 0; }

# ---------- 调用官方 run_train.sh ----------
cd /root/MOSS-TTS

export MODEL_PATH CODEC_PATH
export RAW_JSONL="$RAW_JSONL_DEFAULT"
export PREPARED_JSONL="$PREPARED_JSONL_DEFAULT"
export OUTPUT_DIR
# realtime 的 prepare_data.py 没有 --model-path 参数（见 realtime 的 run_train.sh 64-69 行）
# 其它两个有——官方 run_train.sh 都处理好了
export PREP_EXTRA_ARGS_STR="$N_VQ_ARG"
export TRAIN_EXTRA_ARGS_STR="$TRAIN_ARGS $N_VQ_ARG"

bash ${ARCH_DIR}/finetuning/run_train.sh

echo ""
echo '------------------------ 训练结束'
echo "checkpoint: $OUTPUT_DIR"
echo "用 bash /root/3-run-infer.sh 测试效果"
```

---

### 11.3 微调后推理脚本（3-run-infer.sh）

训练结束后 `sft.py` 在 `output-dir` 下按 epoch 保存 `checkpoint-epoch-{N}/`（位置：`moss_tts_delay/finetuning/sft.py:695`），每个目录里都写了完整的 config / tokenizer / processor / runtime `.py`，直接 `AutoModel.from_pretrained` 即可——官方微调 README §5 也是这个用法。

```bash
#!/bin/bash
# MOSS-TTS 家族微调后推理脚本（交互式菜单版）
#
# 自动扫描 /root/autodl-tmp/checkpoints/<train_name>/checkpoint-epoch-*，
# 让你选一个 checkpoint，然后合成一段音频。

eval "$(conda shell.bash hook 2>/dev/null)"
conda activate moss-tts

CKPT_ROOT="/root/autodl-tmp/checkpoints"

echo "=========================================="
echo "   MOSS-TTS 微调后推理启动器"
echo "=========================================="
echo ""

# ---------- 扫 checkpoint ----------
[ ! -d "$CKPT_ROOT" ] && { echo "❌ checkpoint 根目录不存在: $CKPT_ROOT"; exit 1; }
mapfile -t CKPT_LIST < <(find "$CKPT_ROOT" -maxdepth 2 -type d -name "checkpoint-epoch-*" 2>/dev/null | sort)
if [ ${#CKPT_LIST[@]} -eq 0 ]; then
    echo "❌ 找不到任何 checkpoint-epoch-*"
    echo "   当前 $CKPT_ROOT 下内容:"; ls -la "$CKPT_ROOT" 2>/dev/null
    exit 1
fi

echo "在 $CKPT_ROOT 下找到以下 checkpoint:"
for i in "${!CKPT_LIST[@]}"; do
    REL="${CKPT_LIST[$i]#$CKPT_ROOT/}"
    printf "  %2d) %s\n" "$((i+1))" "$REL"
done
echo ""
read -p "请选择 checkpoint 编号 [默认 1]: " CHOICE
CHOICE="${CHOICE:-1}"
if ! [[ "$CHOICE" =~ ^[0-9]+$ ]] || [ "$CHOICE" -lt 1 ] || [ "$CHOICE" -gt "${#CKPT_LIST[@]}" ]; then
    echo "❌ 无效选择"; exit 1
fi
CKPT_DIR="${CKPT_LIST[$((CHOICE-1))]}"

# ---------- 识别模型类型（从上级目录名推断） ----------
TRAIN_NAME=$(basename "$(dirname "$CKPT_DIR")")
case "$TRAIN_NAME" in
    moss_tts_sft)                   ARCH="delay" ;;
    moss_ttsd_sft)                  ARCH="delay_ttsd" ;;
    moss_voice_generator_sft)       ARCH="delay_vg" ;;
    moss_sound_effect_sft)          ARCH="delay_se" ;;
    moss_tts_local_sft)             ARCH="local" ;;
    moss_tts_realtime_sft)          ARCH="realtime" ;;
    *)                              ARCH="delay" ;;   # 兜底当 MOSS-TTS
esac

echo ""
echo ">>> 已选: $CKPT_DIR"
echo ">>> 训练目录名: $TRAIN_NAME  → 架构识别: $ARCH"

# ---------- 输入文本 ----------
echo ""
if [ "$ARCH" = "delay_se" ]; then
    read -p "请输入音效描述 (ambient_sound, 英文或中文): " PROMPT
else
    read -p "请输入要合成的文本: " PROMPT
fi
[ -z "$PROMPT" ] && { echo "❌ 输入不能为空"; exit 1; }

# ---------- 可选参考音频 ----------
if [ "$ARCH" != "delay_se" ] && [ "$ARCH" != "delay_vg" ]; then
    echo ""
    read -p "是否启用 voice cloning? [y/N]: " USE_CLONE
    if [[ "${USE_CLONE:-N}" =~ ^[Yy]$ ]]; then
        read -p "请输入参考音频完整路径: " REF_AUDIO
        [ ! -f "$REF_AUDIO" ] && { echo "❌ 参考音频不存在"; exit 1; }
    fi
fi

# ---------- 准备输出路径 ----------
mkdir -p /root/autodl-tmp/outputs
TS=$(date +%Y%m%d_%H%M%S)
OUT="/root/autodl-tmp/outputs/${TRAIN_NAME}_$(basename "$CKPT_DIR")_${TS}.wav"

echo ""
echo ">>> 开始推理..."
cd /root/MOSS-TTS

# 根据架构调用不同入口
export CKPT_DIR PROMPT REF_AUDIO OUT ARCH

python - <<'PY'
import os, importlib.util
from pathlib import Path
import torch, torchaudio

torch.backends.cuda.enable_cudnn_sdp(False)
torch.backends.cuda.enable_flash_sdp(True)
torch.backends.cuda.enable_mem_efficient_sdp(True)
torch.backends.cuda.enable_math_sdp(True)

ckpt   = os.environ["CKPT_DIR"]
prompt = os.environ["PROMPT"]
ref    = os.environ.get("REF_AUDIO", "") or None
out    = os.environ["OUT"]
arch   = os.environ["ARCH"]

device = "cuda"
dtype  = torch.bfloat16
attn = "flash_attention_2" if importlib.util.find_spec("flash_attn") else "sdpa"

if arch in ("delay", "delay_ttsd", "delay_vg", "delay_se", "local"):
    from transformers import AutoModel, AutoProcessor
    proc = AutoProcessor.from_pretrained(ckpt, trust_remote_code=True)
    proc.audio_tokenizer = proc.audio_tokenizer.to(device)
    model = AutoModel.from_pretrained(
        ckpt, trust_remote_code=True,
        attn_implementation=attn, torch_dtype=dtype,
    ).to(device).eval()

    if arch == "delay_vg":
        # VoiceGenerator: text + instruction. 这里把 prompt 当 instruction，
        # text 用一个占位句，用户可以自己改脚本
        msg = proc.build_user_message(
            text="This is a sample synthesized via MOSS-VoiceGenerator.",
            instruction=prompt,
        )
    elif arch == "delay_se":
        # SoundEffect 用 ambient_sound 字段（见 docs/moss_sound_effect_model_card.md）
        msg = proc.build_user_message(ambient_sound=prompt)
    elif arch == "delay_ttsd":
        # TTSD: 格式示例见 moss_tts_delay/finetuning/README.md §2.2
        # 如果没给参考，模型自由生成
        kwargs = {"text": prompt}
        if ref:
            kwargs["reference"] = [ref]
        msg = proc.build_user_message(**kwargs)
    else:
        kwargs = {"text": prompt}
        if ref:
            kwargs["reference"] = [ref]
        msg = proc.build_user_message(**kwargs)

    batch = proc([[msg]], mode="generation")
    with torch.no_grad():
        o = model.generate(
            input_ids=batch["input_ids"].to(device),
            attention_mask=batch["attention_mask"].to(device),
            max_new_tokens=4096,
        )
    audio = proc.decode(o)[0].audio_codes_list[0]
    torchaudio.save(out, audio.unsqueeze(0), proc.model_config.sampling_rate)

elif arch == "realtime":
    # Realtime: 按 docs/moss_tts_realtime_model_card.md §2 的流程
    import sys
    sys.path.insert(0, "moss_tts_realtime")
    from mossttsrealtime.modeling_mossttsrealtime import MossTTSRealtime
    from inferencer import MossTTSRealtimeInference
    from transformers import AutoTokenizer, AutoModel

    model = MossTTSRealtime.from_pretrained(
        ckpt, attn_implementation="sdpa", torch_dtype=dtype,
    ).to(device)
    tok = AutoTokenizer.from_pretrained(ckpt)
    codec = AutoModel.from_pretrained(
        "OpenMOSS-Team/MOSS-Audio-Tokenizer", trust_remote_code=True
    ).eval().to(device)

    inf = MossTTSRealtimeInference(
        model, tok, max_length=5000, codec=codec, codec_sample_rate=24000,
        codec_encode_kwargs={"chunk_duration": 8},
    )
    ref_list = [ref if ref else ""]
    res = inf.generate(
        text=[prompt], reference_audio_path=ref_list,
        temperature=0.8, top_p=0.6, top_k=30,
        repetition_penalty=1.1, repetition_window=50,
        device=device,
    )
    for i, gen in enumerate(res):
        wav = codec.decode(torch.tensor(gen).to(device).permute(1,0), chunk_duration=8)["audio"][0].cpu()
        if wav.ndim == 1: wav = wav.unsqueeze(0)
        torchaudio.save(out, wav, 24000)
        break

print("OK:", out)
PY

echo ""
echo '------------------------ 推理完成'
echo "输出文件: $OUT"
echo "用 WinSCP 下载到本地听效果。"
```

---

### 11.4 使用指南（MOSS-TTS使用指南.txt）

将此文件上传到 `/root/MOSS-TTS使用指南.txt`，方便随时查阅。

```
================================================================================
  MOSS-TTS 家族使用指南（AutoDL 平台）
================================================================================

本平台支持 MOSS-TTS 家族全部 5 个模型的推理和全量微调。
所有操作通过 /root/ 下的三个 sh 脚本完成，脚本会自动激活 moss-tts conda 环境，
不需要手动切换环境。

--------------------------------------------------------------------------------
  模型速查（源：官方根 README Released Models 表格）
--------------------------------------------------------------------------------

模型                      架构              参数  本地路径
-------------------------- ---------------- ---- -----------------------------------
MOSS-TTS                   MossTTSDelay      8B  /root/autodl-tmp/models/MOSS-TTS
MOSS-TTS-Local-Transformer MossTTSLocal    1.7B  /root/autodl-tmp/models/MOSS-TTS-Local-Transformer
MOSS-TTSD-v1.0             MossTTSDelay      8B  /root/autodl-tmp/models/MOSS-TTSD-v1.0  (n_vq=16!)
MOSS-VoiceGenerator        MossTTSDelay    1.7B  /root/autodl-tmp/models/MOSS-VoiceGenerator
MOSS-SoundEffect           MossTTSDelay      8B  /root/autodl-tmp/models/MOSS-SoundEffect
MOSS-TTS-Realtime          MossTTSRealtime 1.7B  /root/autodl-tmp/models/MOSS-TTS-Realtime
MOSS-Audio-Tokenizer       (共享音频分词器)    ~1B  HF 仓库 ID: OpenMOSS-Team/MOSS-Audio-Tokenizer
                                                    （HF 缓存: /root/hf_cache/hub/models--OpenMOSS-Team--MOSS-Audio-Tokenizer/）

--------------------------------------------------------------------------------
  目录结构
--------------------------------------------------------------------------------

/root/
  1-run-webui.sh          WebUI 启动脚本（5 个模型任选其一）
  2-run-train.sh          微调训练启动脚本（6 种任务）
  3-run-infer.sh          微调后推理脚本

/root/MOSS-TTS/           MOSS-TTS 源码（editable 安装）
/root/models/             所有下载的模型权重

/root/autodl-tmp/
  datasets/              训练数据集（train_raw.jsonl + 切好的 wav）
  checkpoints/            训练产出的 checkpoint（每 epoch 一份）
  outputs/                推理输出的 wav

--------------------------------------------------------------------------------
  脚本 1：WebUI 启动器（1-run-webui.sh）
--------------------------------------------------------------------------------

功能：启动 Gradio 界面。

运行方式：
  bash /root/1-run-webui.sh

脚本会显示菜单让你选择：
  1) MOSS-TTS                    （旗舰 TTS / 声音克隆 / 多语种，MossTTSDelay 8B）
  2) MOSS-TTS-Local-Transformer  （旗舰 TTS 的 1.7B 轻量版架构，小显卡首选）
  3) MOSS-TTSD                   （多人对话）
  4) MOSS-VoiceGenerator         （自然语言描述生成音色）
  5) MOSS-SoundEffect            （音效生成）
  6) MOSS-TTS-Realtime           （流式实时 TTS）

选完后自动加载模型并启动 WebUI，端口统一为 6006。

访问 WebUI 的方法（二选一）：
  方法 A：AutoDL 控制台 → 我的实例 → 点"自定义服务" → 自动打开浏览器
  方法 B：本地电脑开 SSH 隧道转发 6006 端口
         参考教程 https://www.autodl.com/docs/ssh_proxy/
         代理成功后在本地浏览器访问 http://127.0.0.1:6006/

退出 WebUI：在启动脚本的终端按 Ctrl+C

注意事项：
  - 首次启动 8B 模型约 1-3 分钟（加载 8B 权重 + attn kernel 编译 + 可能 torch.compile 预热）
  - 1.7B 模型约 30–60 秒
  - 5 个 demo 同时只能起一个（因为都抢 6006 端口）

--------------------------------------------------------------------------------
  脚本 2：微调训练（2-run-train.sh）
--------------------------------------------------------------------------------

功能：启动 MOSS-TTS 家族微调训练（全量）。

运行方式：
  bash /root/2-run-train.sh

前置条件：
  1. 在 /root/autodl-tmp/datasets/ 下准备好训练数据:
     - train_raw.jsonl （原始清单，字段依任务而定，详见微调文档）
     - 对应的音频 wav 文件

  2. 选择你要微调的模型（1–6）。脚本会:
     - 自动选择 moss_tts_delay / moss_tts_local / moss_tts_realtime 三套 pipeline 之一
     - 对 MOSS-TTSD 自动追加 --n-vq 16
     - 对 8B/1.7B 模型自动选择合理的 accelerate 参数
     - 展开 accelerate launch → prepare_data.py → sft.py 流程

建议用 screen 会话跑，防止 SSH 断连:
  screen -S train
  bash /root/2-run-train.sh
  # Ctrl+A 然后 D 放后台
  # screen -r train 回来看进度

监控训练（另开一个终端）:
  tensorboard --logdir /root/autodl-tmp/checkpoints/<train_name>/runs --port 6006
  然后用 AutoDL "自定义服务"或 SSH 隧道打开 http://127.0.0.1:6006/
  （前提：训练开了 wandb/tb 记录；默认 sft.py 只输出到 stdout）

详细数据准备与超参说明：/root/MOSS-TTS/2026-4-22-moss-tts-finetune-all-tutorial.md

--------------------------------------------------------------------------------
  脚本 3：微调后推理（3-run-infer.sh）
--------------------------------------------------------------------------------

功能：对 sft.py 产出的 checkpoint-epoch-* 做推理测试。

运行方式：
  bash /root/3-run-infer.sh

使用流程（脚本会引导你完成）:
  1. 自动扫描 /root/autodl-tmp/checkpoints/*/checkpoint-epoch-*，列出菜单
  2. 根据训练目录名自动识别模型架构
  3. 输入要合成的文本（或 SoundEffect 的场景描述 / VoiceGenerator 的 instruction）
  4. 选择是否启用 voice cloning
  5. 推理完成后 wav 保存到 /root/autodl-tmp/outputs/

--------------------------------------------------------------------------------
  日常使用速查
--------------------------------------------------------------------------------

想做什么                              命令
----------------------------------  -----------------------------------------------
听 MOSS-TTS 旗舰原版音色 (8B)         bash /root/1-run-webui.sh  (选 1)
听 MOSS-TTS Local 轻量版 (1.7B)      bash /root/1-run-webui.sh  (选 2)
听 MOSS-TTSD 多人对话                bash /root/1-run-webui.sh  (选 3)
生成新音色 (VoiceGenerator)          bash /root/1-run-webui.sh  (选 4)
生成音效 (SoundEffect)                bash /root/1-run-webui.sh  (选 5)
流式实时 TTS 演示 (Realtime)         bash /root/1-run-webui.sh  (选 6)
启动微调训练                          bash /root/2-run-train.sh
测试训练出的 checkpoint               bash /root/3-run-infer.sh
启动 Realtime FastAPI 服务（高级）    cd /root/MOSS-TTS/moss_tts_realtime &&
                                     python fast_api.py --port 6006 \
                                       --model_path /root/autodl-tmp/models/MOSS-TTS-Realtime \
                                       --tokenizer_path /root/autodl-tmp/models/MOSS-TTS-Realtime \
                                       --codec_model_path OpenMOSS-Team/MOSS-Audio-Tokenizer
用 Jupyter                           直接用（base 环境，不受影响）
用 TensorBoard                       tensorboard --logdir <日志目录> --port 6006
手动跑 MOSS-TTS Python 代码          conda activate moss-tts 然后 python
切回默认环境                          conda activate base

--------------------------------------------------------------------------------
  常见问题
--------------------------------------------------------------------------------

Q: 运行 sh 脚本报 "bad interpreter: /bin/bash^M"
A: Windows 换行符问题。执行:
   sed -i 's/\r$//' /root/1-run-webui.sh /root/2-run-train.sh /root/3-run-infer.sh

Q: 运行 sh 脚本报 "Permission denied"
A: 没给执行权限。执行:
   chmod +x /root/1-run-webui.sh /root/2-run-train.sh /root/3-run-infer.sh

Q: conda activate moss-tts 报错
A: 先执行 conda init && source ~/.bashrc

Q: 推理时报错 "cuDNN SDPA backend is broken"
A: 官方根 README 要求禁用 cuDNN SDPA（见 README 237-241 行）。
   clis/*.py 和本指南第 9/10 步的脚本开头都有 torch.backends.cuda.enable_cudnn_sdp(False)。
   如果你自己写 Python 代码，务必照抄这四行。

Q: 实例重启后，运行 verify_models.py / smoke_test.py / 业务推理时报 "Connection error" 或 "Failed to connect to huggingface.co"
A: 这不是缓存缺失，是 HF 客户端的 ETag 校验机制：即使 Audio-Tokenizer 缓存完整，
   AutoModel.from_pretrained("OpenMOSS-Team/MOSS-Audio-Tokenizer") 也会先发 HEAD 请求到 HF 对比
   commit hash。AutoDL 直连 huggingface.co 通常不通 → 报错。
   正确姿势是第 7.1 步里持久化的 HF_ENDPOINT=https://hf-mirror.com，让 HEAD 请求走国内镜像。
   检查:
     env | grep HF_ENDPOINT
   期望输出: HF_ENDPOINT=https://hf-mirror.com
   如果为空 → 写入 bashrc 持久化:
     echo 'export HF_ENDPOINT=https://hf-mirror.com' >> ~/.bashrc
     source ~/.bashrc

Q: 推理时 AutoProcessor 仍然尝试连 HF 下载 MOSS-Audio-Tokenizer（即使已经验证了 HF_ENDPOINT）
A: HF 缓存里确实没有它。检查:
   1) env | grep HF_HOME   是否显示 /root/hf_cache
   2) ls /root/hf_cache/hub/models--OpenMOSS-Team--MOSS-Audio-Tokenizer/snapshots/*/   是否有 config.json 等文件
   如果缓存确实缺失，重跑第 8.1 步（HF_ENDPOINT 已持久化，直接跑）：
   hf download OpenMOSS-Team/MOSS-Audio-Tokenizer

Q: 第 8.1 步 hf-mirror 下载中途断开 / 进度卡住
A: 重跑同一条命令，HF 会自动续传未完成的 blob：
     hf download OpenMOSS-Team/MOSS-Audio-Tokenizer
   再跑 8.1 步的完整性 3 项校验，直到 "OK: no incomplete blobs" 且 du -sh ≈ 7.1G。

Q: hf-mirror.com 彻底不可用（极端兜底）
A: hf-mirror 由志愿者维护，偶有维护窗口或被限流。用 ModelScope 下 Audio-Tokenizer 到 /root/models/ 兜底：
   1) 清掉 HF 半成品缓存:
      rm -rf /root/hf_cache/hub/models--OpenMOSS-Team--MOSS-Audio-Tokenizer
   2) ModelScope 下:
      python -c "from modelscope import snapshot_download; snapshot_download('openmoss/MOSS-Audio-Tokenizer', local_dir='/root/models/MOSS-Audio-Tokenizer')"
   3) 把 sh 脚本里 `OpenMOSS-Team/MOSS-Audio-Tokenizer` 字串改成 `/root/models/MOSS-Audio-Tokenizer`
      受影响文件：1-run-webui.sh（case 3 和 case 6 的 CODEC_ID）、2-run-train.sh（CODEC_PATH）
      （`verify_models.py` / `smoke_test.py` 内置 `LEGACY_CODEC_DIR = /root/models/MOSS-Audio-Tokenizer`
        兜底，HF 缓存未命中时会自动走本地目录，无需改脚本）
   改成本地绝对路径后，HF 看是路径不是 ID，直接读本地，全流程不碰网络，HF_ENDPOINT 也变得无关。

Q: MOSS-TTS-Realtime 开了 FlashAttention 2 后报 StaticCache 相关错误
A: 官方在 moss_tts_realtime/infer.py 注释里明确写了：
   "local transformer module uses StaticCache, which causes errors when using the FlashAttention implementation"。
   改用 --attn_implementation sdpa 即可。1-run-webui.sh 里启动 Realtime 时已经强制 sdpa。

Q: 8B 模型推理显存不够
A: 96GB RTX PRO 6000 跑 8B bf16 正常。如果还是爆，检查是不是没关掉其它 Python 进程，
   或者尝试 attn_implementation="sdpa"（不用 flash-attn）。

Q: 想只部署其中一两个模型省显存/空间
A: 不下载对应的 /root/autodl-tmp/models/<ModelName> 即可，sh 脚本会自动检测目录是否存在并报错提示。

================================================================================
```

---

## 部署完成 — 最终状态清单

| 项 | 预期值 |
|---|---|
| conda base 环境 | Python 3.12.x（Jupyter/TensorBoard 正常） |
| **conda moss-tts 环境** | **Python 3.12.x**（MOSS-TTS 家族专用） |
| PyTorch（moss-tts 环境内） | 2.9.1+cu128 |
| torchaudio（moss-tts 环境内） | 2.9.1+cu128 |
| torchcodec | 0.8.1 |
| transformers | 5.0.0 |
| CUDA | True, 12.8 |
| MOSS-TTS 源码 | `/root/MOSS-TTS/`（editable） |
| **HF_HOME** | **`/root/hf_cache`**（系统盘，进系统镜像） |
| **HF_HUB_ENABLE_HF_TRANSFER** | **`1`**（加速 HF 下载） |
| **HF_ENDPOINT** | **`https://hf-mirror.com`**（持久化到 bashrc，进系统镜像；每次实例重启后推理 / 验证 / 业务模型内部加载 tokenizer 的 HEAD 请求都走镜像，避免 huggingface.co 直连不通） |
| **MOSS-Audio-Tokenizer（hf-mirror 下载）** | **`/root/hf_cache/hub/models--OpenMOSS-Team--MOSS-Audio-Tokenizer/`**（HF 标准 `refs` + `snapshots` + `blobs` 结构，`du -sh ≈ 7.1G`） |
| MOSS-TTS 权重（可选） | `/root/autodl-tmp/models/MOSS-TTS/` |
| MOSS-TTS-Local-Transformer 权重（可选） | `/root/autodl-tmp/models/MOSS-TTS-Local-Transformer/` |
| MOSS-TTSD-v1.0 权重（可选） | `/root/autodl-tmp/models/MOSS-TTSD-v1.0/` |
| MOSS-VoiceGenerator 权重（可选） | `/root/autodl-tmp/models/MOSS-VoiceGenerator/` |
| MOSS-SoundEffect 权重（可选） | `/root/autodl-tmp/models/MOSS-SoundEffect/` |
| MOSS-TTS-Realtime 权重（可选） | `/root/autodl-tmp/models/MOSS-TTS-Realtime/` |
| FlashAttention 2 | ✅（可选） |
| accelerate / deepspeed（微调） | ✅ |
| ffmpeg / libsndfile | ✅ |
| tensorboard / modelscope / huggingface_hub / hf_transfer | ✅（moss-tts 环境内） |
| `verify_models.py` 通过 | ✅ OK ≥ 1（MOSS-Audio-Tokenizer）+ 已下载的业务模型数 |
| 冒烟测试 wav | ✅ `/root/autodl-tmp/smoke/*.wav` |

**满足这张表 = 可以保存 AutoDL 系统镜像了**。下次开新实例选这个镜像，直接开干。

---

## 日常使用

### MOSS-TTS 相关（推理 / 训练 / 数据处理）

**直接用 sh 脚本**，脚本会自动激活 moss-tts 环境：

```bash
bash /root/1-run-webui.sh     # WebUI（5 个模型任选）
bash /root/2-run-train.sh     # 启动训练
bash /root/3-run-infer.sh     # 测试 checkpoint
```

**手动跑 Python 命令时**先切环境：

```bash
conda activate moss-tts
python -c "from transformers import AutoModel; m = AutoModel.from_pretrained('/root/autodl-tmp/models/MOSS-TTS', trust_remote_code=True)"
```

### 切换环境速查

```bash
conda activate moss-tts  # 切到 MOSS-TTS 专用环境
conda activate base      # 切回 AutoDL 默认环境
conda deactivate         # 退出当前环境
```

### 新实例拉起系统镜像后的快速检查

如果你从保存的系统镜像开新实例，开机后按下面步骤逐项确认环境完整。

**第 1 步：检查环境变量已就位**

```bash
env | grep -E 'HF_HOME|HF_TRANSFER|HF_ENDPOINT'
```

期望输出（3 项缺一不可）：

```
HF_HOME=/root/hf_cache
HF_HUB_ENABLE_HF_TRANSFER=1
HF_ENDPOINT=https://hf-mirror.com
```

> **`HF_ENDPOINT` 缺失会怎样？** 推理时业务模型内部调 `AutoModel.from_pretrained("OpenMOSS-Team/MOSS-Audio-Tokenizer")` 会发 HEAD 请求到 HF 做 ETag 校验，走默认 `huggingface.co` 在 AutoDL 上不通 → 报 `Connection error`。如果 3 项有缺失，重新执行部署文档第 7.1 步的 3 条 `echo 'export ...' >> ~/.bashrc` + `source ~/.bashrc`。

**第 2 步：检查 HF 缓存里 MOSS-Audio-Tokenizer 就位**

```bash
ls /root/hf_cache/hub/models--OpenMOSS-Team--MOSS-Audio-Tokenizer/snapshots/*/
```

期望输出：能看到 `config.json`、`modeling_*.py`、`model.safetensors` 等文件（以及其它若干配置 / 代码文件）。

**第 3 步：检查 conda 环境可用**

```bash
conda activate moss-tts
python -c "import torch, transformers; print(torch.__version__, transformers.__version__)"
```

期望输出：

```
2.9.1+cu128 5.0.0
```

**第 4 步：下业务模型到数据盘**（按需，直连 ModelScope）

```bash
python -c "from modelscope import snapshot_download; snapshot_download('openmoss/MOSS-TTS-Local-Transformer', local_dir='/root/autodl-tmp/models/MOSS-TTS-Local-Transformer')"
```

期望结果：`/root/autodl-tmp/models/MOSS-TTS-Local-Transformer/` 下出现 `config.json` / `model.safetensors` 等权重文件。

**第 5 步：验证**

```bash
python /root/verify_models.py
```

期望输出：末尾汇总行形如 `汇总：OK=N  FAIL=0  （已发现 N 个模型）`，其中 `OK ≥ 1`（至少 MOSS-Audio-Tokenizer）、`FAIL=0`。
