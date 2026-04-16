# VoxCPM AutoDL 全新环境部署完整指南

适用场景：**AutoDL 算力云 GPU（Ubuntu 22.04 / 基础镜像带 PyTorch 2.8 + CUDA 12.8 / miniconda base 环境）**，目标是从零部署 VoxCPM（1.5 和 2 两个版本并存），可推理、可全量微调、可 LoRA 微调。

本指南使用**独立 conda 环境**（名为 `voxcpm`，Python 3.11），不修改 AutoDL 的 base 环境——Jupyter、TensorBoard 等预装工具完全不受影响。

---

## 目录布局

| 位置 | 内容 |
|---|---|
| `/root/miniconda3/` | conda 管理器 + base 环境（Python 3.12，AutoDL 默认） |
| `/root/miniconda3/envs/voxcpm/` | **voxcpm 独立环境**（Python 3.11，VoxCPM 专用） |
| `/root/VoxCPM/` | VoxCPM 源码（editable 安装，支持 v1.5 和 v2） |
| `/root/models/VoxCPM2/` | VoxCPM2 基础模型权重（~10 GB） |
| `/root/models/VoxCPM1.5/` | VoxCPM1.5 基础模型权重（~3 GB，可选） |
| `/root/autodl-tmp/` | 训练数据、checkpoint、TensorBoard 日志等大件 |

**两套 Python 并存**：
- `conda activate base` → Python 3.12（Jupyter / TensorBoard / AutoDL 内置工具）
- `conda activate voxcpm` → Python 3.11（VoxCPM 推理 / 训练 / 数据处理）

三个 sh 脚本（`1-run-webui.sh`、`2-run-train.sh`、`3-run-infer.sh`）开头会自动 `conda activate voxcpm`，你不需要手动切环境。

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

---

## 第 1 步：创建 voxcpm 独立 conda 环境（Python 3.11）


### 1.1 初始化 conda shell

AutoDL 新开机后 `conda activate` 可能报 `Run 'conda init' before 'conda activate'`，需要先初始化：

```bash
conda init
source ~/.bashrc
```

执行完后提示符前面会出现 `(base)`，说明 conda shell 集成已生效。

### 1.2 创建环境

```bash
conda create -n voxcpm python=3.11 -y
```

大约 1–3 分钟。完成后激活：

```bash
conda activate voxcpm
```

### 1.3 验证

```bash
python --version
which python pip
```

期望：
```
Python 3.11.x
/root/miniconda3/envs/voxcpm/bin/python
/root/miniconda3/envs/voxcpm/bin/pip
```

提示符变成 `(voxcpm)` 说明你已经在 voxcpm 环境里了。

**从这一步开始到第 10 步结束，所有操作都在 `(voxcpm)` 环境里执行。**

---

## 第 2 步：系统依赖（apt）

apt 安装的是系统级别的工具，跟 conda 环境无关，在哪个环境里跑都一样：

```bash
apt update
apt install -y ffmpeg libsndfile1 git build-essential screen
```

---

## 第 3 步：安装 PyTorch 2.8（cu128）

voxcpm 环境是全新的，里面没有 PyTorch，需要手动装。

### 3.1 开启代理

PyTorch 轮子托管在 `download.pytorch.org`（境外），需要代理：

```bash
source /etc/network_turbo
```

### 3.2 安装

```bash
pip install torch==2.8.0 torchaudio==2.8.0 --index-url https://download.pytorch.org/whl/cu128
```

约 1 GB，2–5 分钟。

### 3.3 验证

```bash
python -c "import torch; print('torch:', torch.__version__); print('cuda:', torch.cuda.is_available(), torch.version.cuda); print('gpu:', torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'NO GPU')"
```

期望：
```
torch: 2.8.0+cu128
cuda: True 12.8
gpu: NVIDIA GeForce RTX 5090
```

**代理保持开启**，继续往下走第 4–7 步（都需要境外访问）。

---

## 第 4 步：克隆 VoxCPM-main 源码

```bash
cd /root
git clone https://github.com/OpenBMB/VoxCPM.git
```

---

## 第 5 步：从源码 editable 安装 VoxCPM

```bash
cd /root/VoxCPM
pip install -e .
```

大约 10–15 分钟。

**注意**：`pip install -e .` 可能自动把 torchaudio 升级到 2.11.0（pyproject 写的是 `torchaudio>=2.5.0` 没有上限），同时装上 torchcodec 0.11.0。下一步检查并修复。

---

## 第 6 步：检查 torchaudio 版本是否被升级

### 6.1 查看版本

```bash
pip show torch torchaudio 2>/dev/null | grep -E "^Name|^Version"
```

两种可能的结果：

**情况 A（正常）**：torchaudio 没被升级

```
Name: torch
Version: 2.8.0+cu128
Name: torchaudio
Version: 2.8.0+cu128           ← 已对齐，不用动
```

→ **跳到第 7 步**，无需修复。

**情况 B（需修复）**：torchaudio 被升到了 2.11.0 或更高版本

```
Name: torch
Version: 2.8.0+cu128
Name: torchaudio
Version: 2.11.0                ← 错误！新版要 CUDA 13，会报 libcudart.so.13 找不到
```

### 6.2 修复（仅情况 B 需要）

```bash
pip uninstall -y torchaudio
pip install torchaudio==2.8.0 --index-url https://download.pytorch.org/whl/cu128
```

### 6.3 验证

```bash
pip show torch torchaudio 2>/dev/null | grep -E "^Name|^Version"
```

期望：
```
Name: torch
Version: 2.8.0+cu128
Name: torchaudio
Version: 2.8.0+cu128
```

**关于 torchcodec**：`pip install -e .` 会同时装上 `torchcodec`，这是 VoxCPM 的正式依赖，**不需要卸载**。之前推理报 `libcudart.so.13` 错误的根源是 torchaudio 被升级到 2.11.0，不是 torchcodec 的问题。只要 torchaudio 版本对齐，torchcodec 留着不碍事。

---

## 第 7 步：补装额外的 pip 包

```bash
pip install tensorboardX tensorboard
pip install -U modelscope
```

### 7.1 装完后立刻关代理

```bash
unset http_proxy && unset https_proxy
```

**从这里开始到部署结束，代理都是关的。**

---

## 第 8 步：完整导入验证

```bash
python -c "import torch, torchaudio; from voxcpm import VoxCPM; from tensorboardX import SummaryWriter; import argbind, transformers, einops, soundfile, librosa, funasr; import sys; print('OK'); print('python:', sys.version.split()[0]); print('torch:', torch.__version__); print('torchaudio:', torchaudio.__version__); print('cuda:', torch.cuda.is_available()); print('device:', torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'CPU')"
```

期望：
```
OK
python: 3.11.x
torch: 2.8.0+cu128
torchaudio: 2.8.0+cu128
cuda: True
device: NVIDIA GeForce RTX 5090
```

---

## 第 9 步：下载基础模型（用 ModelScope）

建议用 screen 挂后台：`screen -S dl`

### 9.1 下载 VoxCPM2

```bash
mkdir -p /root/models
python -c "from modelscope import snapshot_download; snapshot_download('OpenBMB/VoxCPM2', local_dir='/root/models/VoxCPM2')"
```

约 10 GB，5–15 分钟。下载完验证：
```bash
ls -lh /root/models/VoxCPM2/
```
应至少有 `model.safetensors`（~10 GB）和 `audiovae.pth`。

### 9.2 下载 VoxCPM1.5（可选）

```bash
python -c "from modelscope import snapshot_download; snapshot_download('OpenBMB/VoxCPM1.5', local_dir='/root/models/VoxCPM1.5')"
```

约 3 GB，2–5 分钟。下载完验证：
```bash
ls -lh /root/models/VoxCPM1.5/
```

### 9.3 预下载 SenseVoice-Small ASR 模型

推理 WebUI 内置了语音识别功能（上传参考音频时自动转写）。这个 ASR 模型约 893 MB，不提前下好首次启动 WebUI 会很慢。

```bash
python -c "from funasr import AutoModel; AutoModel(model='iic/SenseVoiceSmall', disable_update=True)"
```

### 9.4 两个版本的差异速查表

| 项目 | VoxCPM2 | VoxCPM1.5 |
|---|---|---|
| 参数量 | 2B | 0.8B |
| 模型大小 | ~10 GB | ~3 GB |
| 训练数据采样率 | 16 kHz（yaml: `sample_rate: 16000`） | 44.1 kHz（yaml: `sample_rate: 44100`） |
| 推理显存占用 | ~8 GB | ~6 GB |
| ModelScope ID | `OpenBMB/VoxCPM2` | `OpenBMB/VoxCPM1.5` |

**采样率不需要手动处理**：`segment_audio.py` 保留母带原采样率，训练时 dataloader 自动重采样到 yaml 指定的目标。

---

## 第 10 步：推理冒烟测试

确保当前在 voxcpm 环境里：`conda activate voxcpm`

### 10.1 测试 VoxCPM2

```bash
python -c "from voxcpm import VoxCPM; import soundfile as sf; m = VoxCPM.from_pretrained('/root/models/VoxCPM2', load_denoiser=False); wav = m.generate(text='Hello, this is VoxCPM2 running on AutoDL.', cfg_value=2.0, inference_timesteps=10); sf.write('/root/autodl-tmp/smoke_v2.wav', wav, m.tts_model.sample_rate); print('OK:', len(wav), 'samples')"
```

期望看到 `OK: XXXXXX samples`。第一次运行约 60–100 秒（torch.compile 预热）。

### 10.2 测试 VoxCPM1.5（可选）

```bash
python -c "from voxcpm import VoxCPM; import soundfile as sf; m = VoxCPM.from_pretrained('/root/models/VoxCPM1.5', load_denoiser=False); wav = m.generate(text='Hello, this is VoxCPM 1.5 running on AutoDL.', cfg_value=2.0, inference_timesteps=10); sf.write('/root/autodl-tmp/smoke_v1_5.wav', wav, m.tts_model.sample_rate); print('OK:', len(wav), 'samples')"
```

### 人工感知验证

用 WinSCP 把 `smoke_v2.wav`（和 `smoke_v1_5.wav`）下到本地听——应该是清晰的英文朗读。

---

## 部署完成 — 最终状态清单

| 项 | 预期值 |
|---|---|
| conda base 环境 | Python 3.12.x（Jupyter/TensorBoard 正常） |
| **conda voxcpm 环境** | **Python 3.11.x**（VoxCPM 专用） |
| PyTorch（voxcpm 环境内） | 2.8.0+cu128 |
| torchaudio（voxcpm 环境内） | 2.8.0+cu128 |
| CUDA | True, 12.8 |
| VoxCPM 源码位置 | `/root/VoxCPM/`（editable） |
| VoxCPM2 模型位置 | `/root/models/VoxCPM2/`（~10 GB） |
| VoxCPM1.5 模型位置 | `/root/models/VoxCPM1.5/`（~3 GB，可选） |
| SenseVoice-Small ASR | 已缓存到 `~/.cache/modelscope/` |
| ffmpeg / libsndfile | ✅ |
| tensorboardX / tensorboard | ✅（voxcpm 环境内） |
| 推理冒烟测试 | ✅ |

---

## 日常使用

### VoxCPM 相关（推理 / 训练 / 数据处理）

**直接用 sh 脚本**，脚本会自动激活 voxcpm 环境：
```bash
bash /root/1-run-webui.sh     # 推理 WebUI
bash /root/2-run-train.sh     # 启动训练
bash /root/3-run-infer.sh     # 测试 checkpoint
```

**手动跑 Python 命令时**先切环境：
```bash
conda activate voxcpm
python -c "from voxcpm import VoxCPM; ..."
```

### Jupyter / TensorBoard / AutoDL 内置工具

**不需要做任何事**，登录后默认就在 base 环境，Jupyter 直接可用。

### 切换环境速查

```bash
conda activate voxcpm    # 切到 VoxCPM 专用环境
conda activate base      # 切回 AutoDL 默认环境
conda deactivate         # 退出当前环境（回到 base 或上一层）
```
