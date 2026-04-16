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

**关于 torchcodec**：`pip install -e .` 会同时装上 `torchcodec`，这是 VoxCPM 的正式依赖，**不需要卸载**。

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

## 第 11 步：部署交付脚本和使用指南

推理冒烟测试通过后，将以下 4 个文件通过 WinSCP 上传到 Autodl 的 `/root/` 目录下：

- `1-run-webui.sh` — 推理 WebUI 启动脚本
- `2-run-train.sh` — 微调训练启动脚本
- `3-run-infer.sh` — 微调后推理脚本
- `VoxCPM使用指南.txt` — 日常使用速查手册

上传后执行权限和换行符修复：
```bash
chmod +x /root/1-run-webui.sh /root/2-run-train.sh /root/3-run-infer.sh
sed -i 's/\r$//' /root/1-run-webui.sh /root/2-run-train.sh /root/3-run-infer.sh
```

所有脚本会自动激活 voxcpm conda 环境，不需要手动切换。

---

### 11.1 推理 WebUI 脚本（1-run-webui.sh）

```bash
#!/bin/bash
# VoxCPM 推理 WebUI 一键启动脚本（交互式菜单版）
#
# 用法:
#   bash /root/1-run-webui.sh

# 激活 voxcpm conda 环境（Python 3.11）
eval "$(conda shell.bash hook 2>/dev/null)"
conda activate voxcpm
#
# 首次运行会自动下载 SenseVoice-Small (ASR 模型，用于识别 prompt 音频)
# 启动后在 Autodl 控制台的"自定义服务"或 SSH 隧道里把端口 6006 转发到本地，
# 浏览器访问 http://127.0.0.1:6006 即可。

echo "=========================================="
echo "   VoxCPM 推理 WebUI 启动器"
echo "=========================================="
echo ""
echo "请选择要启动的模型版本:"
echo "  1) VoxCPM2"
echo "  2) VoxCPM1.5"
echo ""
read -p "请输入编号 [默认 1]: " CHOICE
CHOICE="${CHOICE:-1}"

case "$CHOICE" in
    1)
        MODEL_PATH="/root/models/VoxCPM2"
        VERSION_TAG="VoxCPM2"
        DOWNLOAD_HINT="python -c \"from modelscope import snapshot_download; snapshot_download('OpenBMB/VoxCPM2', local_dir='/root/models/VoxCPM2')\""
        ;;
    2)
        MODEL_PATH="/root/models/VoxCPM1.5"
        VERSION_TAG="VoxCPM1.5"
        DOWNLOAD_HINT="python -c \"from modelscope import snapshot_download; snapshot_download('OpenBMB/VoxCPM1.5', local_dir='/root/models/VoxCPM1.5')\""
        ;;
    *)
        echo "❌ 无效选择: $CHOICE"
        exit 1
        ;;
esac

# 检查模型是否存在
if [ ! -d "$MODEL_PATH" ]; then
    echo ""
    echo "❌ 模型目录不存在: $MODEL_PATH"
    echo ""
    echo "请先下载模型（记得关闭代理）:"
    echo "    unset http_proxy && unset https_proxy"
    echo "    $DOWNLOAD_HINT"
    exit 1
fi

echo ""
echo "=========================================="
echo ">>> 启动 $VERSION_TAG 推理 WebUI"
echo ">>> 模型路径: $MODEL_PATH"
echo ">>> 端口:     6006"
echo "=========================================="
echo ""
echo "启动后请在 Autodl 控制台的「自定义服务」打开 6006 端口"
echo "或在本地 SSH 隧道转发: ssh -L 6006:localhost:6006 ..."
echo ""
echo "按 Ctrl+C 退出 WebUI"
echo ""

cd /root/VoxCPM

python app.py \
    --model-id "$MODEL_PATH" \
    --port 6006

echo ""
echo '------------------------ WebUI 已退出'
```

---

### 11.2 微调训练脚本（2-run-train.sh）

```bash
#!/bin/bash
# VoxCPM 通用微调启动脚本（交互式菜单版，支持 VoxCPM1.5 / VoxCPM2，全量 / LoRA）

# 激活 voxcpm conda 环境（Python 3.11）
eval "$(conda shell.bash hook 2>/dev/null)"
conda activate voxcpm

#
# 数据目录约定: /root/autodl-tmp/datasheet/
#   - train_data.jsonl (训练清单，必须)
#   - val_data.jsonl   (验证清单，可选，yaml 里可写 null)
#   - 一个或多个 yaml 配置文件
#
# 用法:
#   bash /root/2-run-train.sh
#
# 前置条件:
#   1. yaml 里的 train_manifest / pretrained_path / save_path / tensorboard 等已填好真实路径
#   2. 建议在 screen 会话里跑：
#        screen -S train
#        bash /root/2-run-train.sh
#        # Ctrl+A 然后 D 放后台

DATASHEET_DIR="/root/autodl-tmp/datasheet"

echo "=========================================="
echo "   VoxCPM 微调训练启动器"
echo "=========================================="
echo ""

# ---------- 检查数据目录 ----------
if [ ! -d "$DATASHEET_DIR" ]; then
    echo "❌ 数据目录不存在: $DATASHEET_DIR"
    echo ""
    echo "请先创建并上传训练数据:"
    echo "    mkdir -p $DATASHEET_DIR"
    echo "    # 然后用 WinSCP 上传 train_data.jsonl 和 yaml 到该目录"
    exit 1
fi

# ---------- 扫描 yaml 文件 ----------
mapfile -t YAML_FILES < <(ls "$DATASHEET_DIR"/voxcpm_finetune_*.yaml 2>/dev/null)

if [ ${#YAML_FILES[@]} -eq 0 ]; then
    echo "❌ 在 $DATASHEET_DIR 下找不到任何 voxcpm_finetune_*.yaml 配置文件"
    echo ""
    echo "请上传一个配置文件到该目录后重试。支持的文件名格式:"
    echo "  voxcpm_finetune_all_v2.yaml"
    echo "  voxcpm_finetune_all_v1_5.yaml"
    echo "  voxcpm_finetune_lora_v2.yaml"
    echo "  voxcpm_finetune_lora_v1_5.yaml"
    exit 1
fi

# ---------- 让用户选择 yaml ----------
echo "在 $DATASHEET_DIR 下找到以下配置文件:"
echo ""
for i in "${!YAML_FILES[@]}"; do
    FNAME=$(basename "${YAML_FILES[$i]}")
    if [[ "$FNAME" == *"all"* ]]; then
        MODE_LABEL="全量微调"
    elif [[ "$FNAME" == *"lora"* ]]; then
        MODE_LABEL="LoRA 微调"
    else
        MODE_LABEL="未知模式"
    fi
    if [[ "$FNAME" == *"v1_5"* ]] || [[ "$FNAME" == *"v1.5"* ]]; then
        VER_LABEL="VoxCPM1.5"
    elif [[ "$FNAME" == *"v2"* ]]; then
        VER_LABEL="VoxCPM2"
    else
        VER_LABEL="未知版本"
    fi
    printf "  %d) %-40s  [ %s / %s ]\n" "$((i+1))" "$FNAME" "$MODE_LABEL" "$VER_LABEL"
done

echo ""
read -p "请选择要使用的配置 [默认 1]: " CHOICE
CHOICE="${CHOICE:-1}"

if ! [[ "$CHOICE" =~ ^[0-9]+$ ]] || [ "$CHOICE" -lt 1 ] || [ "$CHOICE" -gt "${#YAML_FILES[@]}" ]; then
    echo "❌ 无效选择: $CHOICE"
    exit 1
fi

YAML_PATH="${YAML_FILES[$((CHOICE-1))]}"
YAML_BASENAME=$(basename "$YAML_PATH")

# ---------- 从 yaml 里读关键字段（只读两版都有的字段） ----------
get_yaml_field() {
    local key="$1"
    grep "^${key}:" "$YAML_PATH" | head -1 | sed "s/^${key}:[[:space:]]*//" | sed 's/[[:space:]]*#.*$//' | tr -d '"' | sed 's/[[:space:]]*$//'
}

PRETRAINED=$(get_yaml_field "pretrained_path")
TRAIN_MANIFEST=$(get_yaml_field "train_manifest")
VAL_MANIFEST=$(get_yaml_field "val_manifest")
SAVE_PATH=$(get_yaml_field "save_path")
TENSORBOARD=$(get_yaml_field "tensorboard")

# 识别模式（有没有 lora: 块）
if grep -q "^lora:" "$YAML_PATH"; then
    MODE_TAG="LoRA 微调"
else
    MODE_TAG="全量微调"
fi

# 识别版本
if [[ "$YAML_BASENAME" == *"v1_5"* ]] || [[ "$YAML_BASENAME" == *"v1.5"* ]]; then
    VERSION_TAG="VoxCPM1.5"
elif [[ "$YAML_BASENAME" == *"v2"* ]]; then
    VERSION_TAG="VoxCPM2"
else
    VERSION_TAG="(依赖 pretrained_path 自动判断)"
fi

# 格式化 val_manifest 展示
if [ -z "$VAL_MANIFEST" ] || [ "$VAL_MANIFEST" = "null" ]; then
    VAL_MANIFEST_DISPLAY="(未启用)"
else
    VAL_MANIFEST_DISPLAY="$VAL_MANIFEST"
fi

echo ""
echo "=========================================="
echo ">>> 配置确认"
echo "=========================================="
echo "  模式:       $MODE_TAG"
echo "  版本:       $VERSION_TAG"
echo "  配置文件:   $YAML_PATH"
echo "  基础模型:   $PRETRAINED"
echo "  训练数据:   $TRAIN_MANIFEST"
echo "  验证数据:   $VAL_MANIFEST_DISPLAY"
echo "  输出目录:   $SAVE_PATH"
echo "  日志目录:   $TENSORBOARD"
echo "=========================================="

echo ""
read -p "确认启动训练? [y/N]: " CONFIRM
CONFIRM="${CONFIRM:-N}"

if [[ ! "$CONFIRM" =~ ^[Yy]$ ]]; then
    echo "已取消。"
    exit 0
fi

echo ""
echo ">>> 启动训练..."
echo ""

cd /root/VoxCPM

python scripts/train_voxcpm_finetune.py \
    --config_path "$YAML_PATH"

echo ""
echo '------------------------ 训练结束'
```

---

### 11.3 微调后推理脚本（3-run-infer.sh）

```bash
#!/bin/bash
# VoxCPM 微调后推理脚本（交互式菜单版）

# 激活 voxcpm conda 环境（Python 3.11）
eval "$(conda shell.bash hook 2>/dev/null)"
conda activate voxcpm

#
# 自动识别:
#   - checkpoint 类型 (LoRA / 全量微调)
#   - 模型版本 (VoxCPM1.5 / VoxCPM2)
#
# 用法:
#   bash /root/3-run-infer.sh
#
# 脚本会引导你完成:
#   1. 选择 checkpoint
#   2. 输入合成文本
#   3. 选择是否启用 voice cloning
#   4. (可选) 提供参考音频和转写
#
# 输出自动保存到 /root/autodl-tmp/outputs/<train_name>_<step_name>_<timestamp>.wav

CKPT_ROOT="/root/autodl-tmp/checkpoints"

echo "=========================================="
echo "   VoxCPM 微调后推理启动器"
echo "=========================================="
echo ""

# ---------- 第 1 步：选 checkpoint ----------
if [ ! -d "$CKPT_ROOT" ]; then
    echo "❌ checkpoint 根目录不存在: $CKPT_ROOT"
    echo "请先完成训练，或手动指定 checkpoint 路径。"
    exit 1
fi

# 扫描 CKPT_ROOT 下所有 step_* 目录
mapfile -t CKPT_LIST < <(find "$CKPT_ROOT" -maxdepth 2 -type d -name "step_*" 2>/dev/null | sort)

if [ ${#CKPT_LIST[@]} -eq 0 ]; then
    echo "❌ 在 $CKPT_ROOT 下找不到任何 step_* checkpoint"
    echo ""
    echo "请先完成一次训练，或检查 yaml 里的 save_path 设置。"
    echo "当前 $CKPT_ROOT 下的内容:"
    ls -la "$CKPT_ROOT" 2>/dev/null
    exit 1
fi

echo "在 $CKPT_ROOT 下找到以下 checkpoint:"
echo ""
for i in "${!CKPT_LIST[@]}"; do
    CKPT="${CKPT_LIST[$i]}"
    REL_PATH="${CKPT#$CKPT_ROOT/}"
    if [ -f "$CKPT/lora_weights.safetensors" ] || [ -f "$CKPT/lora_weights.ckpt" ]; then
        TYPE_TAG="LoRA"
    elif [ -f "$CKPT/model.safetensors" ] || [ -f "$CKPT/pytorch_model.bin" ]; then
        TYPE_TAG="全量"
    else
        TYPE_TAG="未知"
    fi
    printf "  %2d) [%s] %s\n" "$((i+1))" "$TYPE_TAG" "$REL_PATH"
done

echo ""
read -p "请选择 checkpoint 编号 [默认 1]: " CHOICE
CHOICE="${CHOICE:-1}"

if ! [[ "$CHOICE" =~ ^[0-9]+$ ]] || [ "$CHOICE" -lt 1 ] || [ "$CHOICE" -gt "${#CKPT_LIST[@]}" ]; then
    echo "❌ 无效选择: $CHOICE"
    exit 1
fi

CKPT_DIR="${CKPT_LIST[$((CHOICE-1))]}"

# ---------- 识别 checkpoint 类型 ----------
if [ -f "$CKPT_DIR/lora_config.json" ] \
   || [ -f "$CKPT_DIR/lora_weights.safetensors" ] \
   || [ -f "$CKPT_DIR/lora_weights.ckpt" ]; then
    MODE="LoRA"
    SCRIPT="scripts/test_voxcpm_lora_infer.py"
    CKPT_ARG_NAME="--lora_ckpt"
elif [ -f "$CKPT_DIR/model.safetensors" ] \
     || [ -f "$CKPT_DIR/pytorch_model.bin" ]; then
    MODE="full"
    SCRIPT="scripts/test_voxcpm_ft_infer.py"
    CKPT_ARG_NAME="--ckpt_dir"
else
    echo ""
    echo "❌ 无法识别 checkpoint 类型: $CKPT_DIR"
    echo "    既没有 lora_weights.* 也没有 model.safetensors"
    ls -la "$CKPT_DIR"
    exit 1
fi

# ---------- 识别版本 ----------
VERSION_TAG="unknown"
if [ -f "$CKPT_DIR/config.json" ]; then
    if grep -q '"architecture"[[:space:]]*:[[:space:]]*"voxcpm2"' "$CKPT_DIR/config.json" 2>/dev/null; then
        VERSION_TAG="VoxCPM2"
    else
        VERSION_TAG="VoxCPM1.5"
    fi
elif [ "$MODE" = "LoRA" ] && [ -f "$CKPT_DIR/lora_config.json" ]; then
    BASE_MODEL=$(grep -o '"base_model"[[:space:]]*:[[:space:]]*"[^"]*"' "$CKPT_DIR/lora_config.json" | sed 's/.*"\([^"]*\)"$/\1/')
    if [ -n "$BASE_MODEL" ] && [ -f "$BASE_MODEL/config.json" ]; then
        if grep -q '"architecture"[[:space:]]*:[[:space:]]*"voxcpm2"' "$BASE_MODEL/config.json" 2>/dev/null; then
            VERSION_TAG="VoxCPM2"
        else
            VERSION_TAG="VoxCPM1.5"
        fi
    fi
fi

echo ""
echo ">>> 已选择: $CKPT_DIR"
echo ">>> 类型:   $MODE ($VERSION_TAG)"
echo ""

# ---------- 第 2 步：输入合成文本 ----------
echo "请输入要合成的文本（可以是任意语言，支持多种语言）:"
read -p "> " TEXT

if [ -z "$TEXT" ]; then
    echo "❌ 文本不能为空"
    exit 1
fi

# ---------- 第 3 步：是否 voice cloning ----------
echo ""
echo "是否启用 voice cloning (用一段参考音频指定说话人音色)?"
echo "  - 不启用: 仅依靠 checkpoint 的音色"
echo "  - 启用:   需要额外提供一段参考音频 + 它的文字转写"
echo ""
read -p "启用 voice cloning? [y/N]: " USE_CLONING
USE_CLONING="${USE_CLONING:-N}"

PROMPT_AUDIO=""
PROMPT_TEXT=""

if [[ "$USE_CLONING" =~ ^[Yy]$ ]]; then
    echo ""
    read -p "请输入参考音频的完整路径: " PROMPT_AUDIO
    if [ ! -f "$PROMPT_AUDIO" ]; then
        echo "❌ 参考音频文件不存在: $PROMPT_AUDIO"
        exit 1
    fi
    echo ""
    read -p "请输入参考音频的文字转写: " PROMPT_TEXT
    if [ -z "$PROMPT_TEXT" ]; then
        echo "❌ 参考音频转写不能为空"
        exit 1
    fi
fi

# ---------- 准备输出路径 ----------
STEP_NAME=$(basename "$CKPT_DIR")
TRAIN_NAME=$(basename "$(dirname "$CKPT_DIR")")
mkdir -p /root/autodl-tmp/outputs
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
OUTPUT_PATH="/root/autodl-tmp/outputs/${TRAIN_NAME}_${STEP_NAME}_${TIMESTAMP}.wav"

# ---------- 展示最终配置并确认 ----------
echo ""
echo "=========================================="
echo ">>> 推理配置确认"
echo "=========================================="
echo "  Checkpoint: $CKPT_DIR"
echo "  类型:       $MODE ($VERSION_TAG)"
echo "  合成文本:   $TEXT"
if [ -n "$PROMPT_AUDIO" ]; then
    echo "  参考音频:   $PROMPT_AUDIO"
    echo "  参考转写:   $PROMPT_TEXT"
    echo "  推理模式:   Voice Cloning"
else
    echo "  推理模式:   普通推理"
fi
echo "  输出路径:   $OUTPUT_PATH"
echo "=========================================="
echo ""
read -p "确认开始推理? [Y/n]: " CONFIRM
CONFIRM="${CONFIRM:-Y}"

if [[ ! "$CONFIRM" =~ ^[Yy]$ ]]; then
    echo "已取消。"
    exit 0
fi

# ---------- 执行推理 ----------
echo ""
echo ">>> 开始推理..."
echo ""

cd /root/VoxCPM

if [ -n "$PROMPT_AUDIO" ] && [ -n "$PROMPT_TEXT" ]; then
    python $SCRIPT \
        $CKPT_ARG_NAME "$CKPT_DIR" \
        --text "$TEXT" \
        --prompt_audio "$PROMPT_AUDIO" \
        --prompt_text "$PROMPT_TEXT" \
        --output "$OUTPUT_PATH" \
        --cfg_value 2.0 \
        --inference_timesteps 10
else
    python $SCRIPT \
        $CKPT_ARG_NAME "$CKPT_DIR" \
        --text "$TEXT" \
        --output "$OUTPUT_PATH" \
        --cfg_value 2.0 \
        --inference_timesteps 10
fi

echo ""
echo '------------------------ 推理完成'
echo "输出文件: $OUTPUT_PATH"
echo ""
echo "用 WinSCP 下载这个 wav 到本地听效果。"
```

---

### 11.4 使用指南（VoxCPM使用指南.txt）

将此文件上传到 `/root/VoxCPM使用指南.txt`，方便随时查阅。

```
================================================================================
  VoxCPM 使用指南（VoxCPM1.5 + VoxCPM2 通用平台）
================================================================================

本平台支持 VoxCPM1.5 和 VoxCPM2 两个版本的推理、全量微调和 LoRA 微调。
所有操作通过 /root/ 下的三个 sh 脚本完成，脚本会自动激活 voxcpm conda 环境，
不需要手动切换环境。

--------------------------------------------------------------------------------
  目录结构
--------------------------------------------------------------------------------

/root/
  1-run-webui.sh          推理 WebUI 启动脚本
  2-run-train.sh          微调训练启动脚本
  3-run-infer.sh          微调后推理脚本

/root/VoxCPM/             VoxCPM 源码（支持 v1.5 和 v2）
/root/models/VoxCPM2/     VoxCPM2 基础模型
/root/models/VoxCPM1.5/   VoxCPM1.5 基础模型（可选）

/root/autodl-tmp/
  datasheet/              训练数据集 + yaml 配置文件
  checkpoints/            训练产出的 checkpoint
  logs/                   TensorBoard 日志
  outputs/                推理输出的 wav 文件

--------------------------------------------------------------------------------
  脚本 1：推理 WebUI（1-run-webui.sh）
--------------------------------------------------------------------------------

功能：启动 Gradio 推理界面，可选 VoxCPM2 或 VoxCPM1.5。

运行方式：
  bash /root/1-run-webui.sh

脚本会显示菜单让你选择模型版本：
  1) VoxCPM2
  2) VoxCPM1.5

选完后自动加载模型并启动 WebUI，端口为 6006。

访问 WebUI 的方法（二选一）：
  方法 A：Autodl 控制台 -> 我的实例 -> 点"自定义服务"-> 自动打开浏览器
  方法 B：本地电脑开 SSH 隧道转发 6006 端口
         参考教程 https://www.autodl.com/docs/ssh_proxy/
         代理成功后在本地浏览器访问 http://127.0.0.1:6006/

退出 WebUI：在启动脚本的终端按 Ctrl+C

注意事项：
  - 首次启动约 1-2 分钟（加载模型 + torch.compile 预热），后续启动约 30 秒
  - 不需要手动开关代理，脚本已处理好

--------------------------------------------------------------------------------
  脚本 2：微调训练（2-run-train.sh）
--------------------------------------------------------------------------------

功能：启动 VoxCPM 微调训练，支持全量微调和 LoRA，支持 v1.5 和 v2。

运行方式：
  bash /root/2-run-train.sh

前置条件：
  1. 在 /root/autodl-tmp/datasheet/ 下准备好训练数据：
     - train_data.jsonl         训练清单（必须）
     - val_data.jsonl           验证清单（可选）
     - 音频切片 wav 文件
     - 一个 yaml 配置文件（以下四种选一个）：
         voxcpm_finetune_all_v2.yaml       VoxCPM2 全量微调
         voxcpm_finetune_lora_v2.yaml      VoxCPM2 LoRA 微调
         voxcpm_finetune_all_v1_5.yaml     VoxCPM1.5 全量微调
         voxcpm_finetune_lora_v1_5.yaml    VoxCPM1.5 LoRA 微调

  2. yaml 文件里的路径已填好真实值：
     - pretrained_path    基础模型路径
     - train_manifest     训练数据清单路径
     - save_path          checkpoint 输出路径
     - tensorboard        TensorBoard 日志路径

使用流程：
  脚本自动扫描 datasheet 目录下的 yaml 文件，列出菜单让你选择。
  选择后展示配置信息，确认后才会启动训练。

建议用 screen 会话跑，防止 SSH 断连：
  screen -S train
  bash /root/2-run-train.sh
  # 按 Ctrl+A 然后 D 放后台
  # screen -r train 回来看进度

监控训练（另开一个终端）：
  tensorboard --logdir /root/autodl-tmp/logs/<日志目录名> --port 6006
  然后用 Autodl"自定义服务"或 SSH 隧道打开 http://127.0.0.1:6006/

--------------------------------------------------------------------------------
  脚本 3：微调后推理（3-run-infer.sh）
--------------------------------------------------------------------------------

功能：对训练产出的 checkpoint 做推理测试。
      自动识别 LoRA / 全量微调类型，自动识别 v1.5 / v2 版本。

运行方式：
  bash /root/3-run-infer.sh

使用流程（脚本会引导你完成）：
  1. 自动扫描 /root/autodl-tmp/checkpoints/ 下的 checkpoint，列出菜单
  2. 输入要合成的文本
  3. 选择是否启用 voice cloning（需要提供参考音频和转写）
  4. 确认配置后开始推理

输出文件保存到 /root/autodl-tmp/outputs/ 目录。
文件名格式：<训练名>_<step号>_<时间戳>.wav
用 WinSCP 下载到本地听效果。

--------------------------------------------------------------------------------
  日常使用速查
--------------------------------------------------------------------------------

想做什么                         命令
-------------------------------  -----------------------------------------------
听 VoxCPM2 原版音色              bash /root/1-run-webui.sh  (选 1)
听 VoxCPM1.5 原版音色            bash /root/1-run-webui.sh  (选 2)
启动微调训练                      bash /root/2-run-train.sh
测试训练出的 checkpoint           bash /root/3-run-infer.sh
用 Jupyter                       直接用（base 环境，不受影响）
用 TensorBoard                   tensorboard --logdir <日志目录> --port 6006
手动跑 VoxCPM Python 代码        conda activate voxcpm 然后 python
切回默认环境                      conda activate base

--------------------------------------------------------------------------------
  常见问题
--------------------------------------------------------------------------------

Q: 运行 sh 脚本报 "bad interpreter: /bin/bash^M"
A: Windows 换行符问题。执行：
   sed -i 's/\r$//' /root/1-run-webui.sh /root/2-run-train.sh /root/3-run-infer.sh

Q: 运行 sh 脚本报 "Permission denied"
A: 没给执行权限。执行：
   chmod +x /root/1-run-webui.sh /root/2-run-train.sh /root/3-run-infer.sh

Q: conda activate voxcpm 报错
A: 先执行 conda init && source ~/.bashrc

Q: WebUI 启动后本地浏览器打不开
A: 确认端口转发已设置。参考 https://www.autodl.com/docs/ssh_proxy/

Q: 模型目录不存在
A: 脚本会提示下载命令，按提示执行即可（用 ModelScope 下载，不需要开代理）。

Q: Jupyter 打不开了
A: 确认在 base 环境。执行 conda activate base 后再试。

Q: 训练时 SSH 断了，训练也停了
A: 下次用 screen 挂后台再跑。screen -S train && bash /root/2-run-train.sh

================================================================================
```

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


### 切换环境速查

```bash
conda activate voxcpm    # 切到 VoxCPM 专用环境
conda activate base      # 切回 AutoDL 默认环境
conda deactivate         # 退出当前环境（回到 base 或上一层）
```
