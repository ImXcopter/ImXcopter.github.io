# MOSS-TTS 全量微调完整指南

本文档提供 MOSS-TTS 家族全部模型的**全量微调**流程，包括数据准备、SRT 处理、音频分割、数据格式生成、环境部署和训练步骤。
适用于 MOSS-TTS / MOSS-TTSD / MOSS-VoiceGenerator / MOSS-SoundEffect / MOSS-TTS-Local-Transformer / MOSS-TTS-Realtime 全部 6 个模型。

> 本文档内容严格依据 `OpenMOSS/MOSS-TTS` GitHub 官方仓库源码：
> - 微调 pipeline：`moss_tts_delay/finetuning/`、`moss_tts_local/finetuning/`、`moss_tts_realtime/finetuning/` 三套目录及各自的 `README.md`
> - JSONL 格式：各微调 README 第 2 节的真实示例
> - 超参命令行：`sft.py` 的 argparse 声明（见 `moss_tts_delay/finetuning/sft.py:87-142`）
> - 一键启动：各 `run_train.sh`
> - 模型特殊约束：根 `README.md` Released Models 表格 + `docs/moss_ttsd_model_card.md`
>
> `segment_audio.py` 仅在 VoxCPM 同名脚本的基础上改写输出 JSONL 字段（这是 MOSS-TTS 与 VoxCPM 的唯一差异），切分逻辑保持一致。

---

## 前置条件

**必须先完成部署**：参见 [`2026-4-22-moss-tts-autodl-deployment-guide.md`](./2026-4-22-moss-tts-autodl-deployment-guide.md)。你需要：

- ✅ AutoDL 实例（本文档实测：RTX PRO 6000 96GB / 25 核 Xeon 8470Q / 120GB 内存）
- ✅ conda `moss-tts` 环境（Python 3.12 + torch 2.9.1+cu128 + transformers 5.0.0）
- ✅ `/root/MOSS-TTS/` editable 源码
- ✅ HF 缓存里已有 MOSS-Audio-Tokenizer（部署文档第 8.1 步 `HF_ENDPOINT=https://hf-mirror.com hf download OpenMOSS-Team/MOSS-Audio-Tokenizer` 已完成）
- ✅ 你要微调的那个业务模型的基础权重已下载到 `/root/autodl-tmp/models/<model-name>/`
- ✅ `accelerate`、（可选）`deepspeed`、`wandb` 已装（即 `pip install -e ".[torch-runtime,finetune]"` 或 `finetune-deepspeed`）

> **关于 codec 路径约定**：`sft.py` 的 `--codec-path` 传 HF 仓库 ID `OpenMOSS-Team/MOSS-Audio-Tokenizer`——`AutoModel.from_pretrained` 自动去 HF 缓存（`/root/hf_cache`）解析，**不走网络**。训练保存的 checkpoint 里 `processor_config.json` 也会记录这个 HF ID，推理加载 checkpoint 时同样命中缓存。

---

## 本文档数据流总览

```
原始长音频 (YouTube/录音等)
        │
        │  一、音频素材清洗（Capcut 去背景音 / 杂音）
        ▼
纯净 WAV
        │
        │  二、Parakeet ASR 转写 → 原始 SRT
        ▼
原始 SRT（每句 2–5 秒的碎片）
        │
        │  三、AI 辅助清洗（合并成 4–28 秒、加标点、修 ASR 错字）
        ▼
清洗后 SRT
        │
        │  四、segment_audio.py 基于 Silero VAD 精准切分
        ▼
切片 wav/0001.wav, 0002.wav, ... + train_raw.jsonl（MOSS-TTS 格式）
        │
        │  五、prepare_data.py 预编码 audio_codes（MOSS-Audio-Tokenizer 24kHz 32 层 RVQ）
        ▼
train_with_codes.jsonl （可选分片）
        │
        │  六、accelerate launch sft.py 全量微调
        ▼
output/<train_name>/checkpoint-epoch-{0,1,2,...}/
        │
        │  七、bash 3-run-infer.sh 验证效果
        ▼
合成 wav
```

---

## 一、数据准备

### 1.1 音频素材要求

与 VoxCPM 完全相同：

- **纯净度要求**：无背景音乐、无环境杂音
- **推荐来源**：YouTube 视频/音频等高质量资源
- **格式建议**：WAV 格式（无损音质）
- **采样率**：不限（`prepare_data.py` 交给 `MOSS-Audio-Tokenizer` 内部统一重采样到训练目标）

> 与 VoxCPM 的差异：MOSS-Audio-Tokenizer 的目标采样率是 **24 kHz**（见根 README "MOSS-Audio-Tokenizer" 章节："compresses 24kHz raw audio..."），不是 VoxCPM1.5 的 44.1 k 也不是 VoxCPM2 的 16 k。但你**不需要**在本地手动重采样——编码管线会自动处理。

### 1.2 音频预处理（Capcut）

使用 **Capcut** 进行音频清理，与 VoxCPM 完全相同：

1. 剪除片头片尾含音乐的部分
2. 去除中间的杂音片段
3. 导出为 WAV 格式

### 1.3 语音转写（Parakeet）

使用 **Parakeet** 工具对 WAV 音频进行自动语音识别（ASR），生成 SRT 字幕文件。步骤与 VoxCPM 教程相同。

---

## 二、SRT 字幕处理

### 2.1 处理平台

访问 [Google AI Studio](https://aistudio.google.com) 或使用 Claude 等 AI 辅助处理。

### 2.2 清洗 Prompt

**直接复用 VoxCPM 教程的 SRT 清洗 Prompt**（见 `VoxCPM/2026-4-16-voxcpm-finetune-all-tutorial.md` 第二章），该 Prompt 的所有规则（时间轴精度、时长控制、语义完整性、标点/大小写修复、文本-音频一致性、整段删除判断、复查清单）对 MOSS-TTS **同样适用**，因为这些都是 TTS 训练数据的共性硬约束，与具体模型无关。

**唯一要额外考虑的**：官方 README §93 用 "long-speech generation" 和 "remain stable over tens of minutes"（根 `README.md:91,93`）描述 MOSS-TTS 的长音频能力，**未明确单条样本长度上限**。对微调的单条训练样本，仍然推荐 **4–28 秒为最佳、3–30 秒为极限**——这个时长窗口能让 `gradient_accumulation_steps 8` 的默认 batch 不至于因为单样本过长炸显存。

输出文件名格式：`01.srt`、`02.srt`…，与 VoxCPM 保持一致。

---

## 三、音频分割（适配 MOSS-TTS 的 segment_audio.py）

> **运行位置说明**：此切分流程**在本地电脑**（Windows / macOS / Linux 工作站）完成，**不在 AutoDL 云端**。在本地你需要一个已经装好 `soundfile` / `scipy` / `torch` / `ffmpeg` 的 Python 虚拟环境（与 VoxCPM 教程 §三使用的本地虚拟环境一致，参见 `VoxCPM/2026-4-16-voxcpm-finetune-all-tutorial.md` §3.3–3.4）。切分完成后，用 WinSCP 把**切片 WAV + `train_raw.jsonl`** 一起上传到 AutoDL 的 `/root/autodl-tmp/datasets/`，再执行后续 §6 `prepare_data.py` 预编码步骤。
>
> **为什么不在云端切？** Silero VAD 模型要从 GitHub 下载，切分是一次性纯数据处理任务（不用 GPU），在本地跑完再上传切片更省云端计费时间。

### 3.1 与 VoxCPM 的差异点

此脚本**在 VoxCPM 的 `segment_audio.py` 基础上修改输出 JSONL 字段**。核心差异：

| 项 | VoxCPM | MOSS-TTS |
|---|---|---|
| 输出 JSONL 字段 | `{"audio", "text", "duration"}` | 取决于目标任务，见下表 |
| 目标采样率 | yaml 指定（v1.5=44.1k, v2=16k） | 不强制；MOSS-Audio-Tokenizer 内部按 24 kHz 处理 |
| 尾静音硬上限 | 500 ms（VoxCPM 官方警告） | 官方未明确，但仍建议 ≤ 500 ms（训练目标里别学太多尾静音） |
| 切分算法（Silero VAD + EBU R128 + 尾静音裁剪 + 纯静音过滤 + 追加模式） | 同 VoxCPM，**不改** | 同 VoxCPM，**不改** |

### 3.2 JSONL 字段——按任务列出

依据：各微调目录下 `README.md §2`（MOSS-TTS `moss_tts_delay/finetuning/README.md` §2.1–2.4、MOSS-TTS-Local `moss_tts_local/finetuning/README.md` §2.1–2.4、MOSS-TTS-Realtime `moss_tts_realtime/finetuning/README.md` §2）。

```
任务                    JSONL 每行字段示意                                                            备注
---------------------- ------------------------------------------------------------------------------  ----------------
MOSS-TTS                {"audio", "text", "language"}                                                  最基础
MOSS-TTS + 克隆训练     {"audio", "text", "language", "ref_audio"}                                     ref_audio 是单个路径
MOSS-TTSD               {"audio", "text", "language", "reference": [..., null, ...]}                   reference 是 list，允许 null
MOSS-VoiceGenerator     {"audio", "text", "instruction"}                                               instruction 是自然语言描述
MOSS-SoundEffect        {"audio", "ambient_sound"[, "tokens"]}                                         没有 text
MOSS-TTS-Local          同 MOSS-TTS / TTSD / VG / SE（四种任务格式共用）                                 架构不同但输入格式一致
MOSS-TTS-Realtime       {"id", ["ref_wav",] "conversations":[{"role","text","wav"},...]}               完全不同，多轮对话
```

### 3.3 脚本使用方法

1. **设置 `TTS_SFT_PROJECT_PATH`** 指向你的本地 TTS 数据处理虚拟环境项目（可直接复用 VoxCPM 教程里建的 `VoxCPM_FT`，因为依赖完全相同：`soundfile` / `scipy` / `torch` / `torchaudio` / `tqdm`）。脚本首次启动会自动检测并切到该虚拟环境的 Python 跑——你不需要手动 activate，也不需要建新 venv
2. 修改脚本顶部的 `TARGET_TASK`（5 选 1）、`DEFAULT_LANGUAGE` 以及（如需要）`REF_AUDIO_PATH`
3. **设置 `AUDIO_DIR`** 为 WAV + SRT 所在文件夹的绝对路径（例如 `r"D:\TTS_Training\raw_audio"`）。留空字符串 `""` 则默认使用脚本所在目录（与 VoxCPM 同款兼容行为）
4. 可选：设置 `OUT_DIR`（切片 WAV + JSONL 的输出目录）。留空则默认写到 `<AUDIO_DIR>/datasets/`
5. 将 `AUDIO_FILENAME` / `SRT_FILENAME` 改成要处理的文件名（文件名，不是路径，位于 `AUDIO_DIR` 下）
6. 运行脚本（双击或命令行跑均可，脚本会自动切到 `TTS_SFT_PROJECT_PATH` 指定的虚拟环境；Silero VAD 模型首次下载到 `<TTS_SFT_PROJECT_PATH>/models/`）
7. 对每个音频文件依次改 `AUDIO_FILENAME` / `SRT_FILENAME` 重新运行——脚本支持追加模式（序号与 JSONL 自动续接），与 VoxCPM 行为一致

**Realtime 任务不适合这个切分流程**——它需要多轮对话数据，切片是单条 utterance。Realtime 的数据组织见本文 **§五**。

### 3.4 完整代码

将以下脚本保存到**本地电脑**任意位置（例如 `D:\TTS_Training\segment_audio.py`）。**代码结构与 VoxCPM 的 `segment_audio.py` 逐行对齐**（含虚拟环境自动启动、配置打印、时长分布统计、跳过清单汇总、中间文件清理等完整流程），唯一差异是**写 JSONL 时按 `TARGET_TASK` 分支填入对应字段**（依据 `moss_tts_delay/finetuning/README.md §2.1-2.4`）。

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
MOSS-TTS 训练数据音频切分脚本（改写自 VoxCPM 同名脚本）
基于 SRT + Silero VAD 精准切分，输出 MOSS-TTS 家族各任务所需的 JSONL 格式。

核心差异：相对 VoxCPM 版只改了一处——写 JSONL 时按 TARGET_TASK 分支填入对应字段
（依据 moss_tts_delay/finetuning/README.md §2.1-2.4）。母带响度归一化、Silero VAD 精准切分、尾静音精确裁剪、静音填充、纯静音能量守门、追加模式、虚拟环境自动启动、VAD 模型目录重定向等全部与 VoxCPM 版逐行对齐，风格一致。
"""

import sys
import os
from pathlib import Path
from typing import List, Dict, Tuple, Optional, Any
import json
import subprocess

# ============================================================================
# 配置常量（用户可修改）
# ============================================================================

# 虚拟环境项目路径（用于环境检查和 Silero VAD 模型下载目录）
# 可以直接复用 VoxCPM_FT 的本地虚拟环境（依赖一致：soundfile / scipy / torch / torchaudio / tqdm）
TTS_SFT_PROJECT_PATH = r"D:\Project\TTS_ASR_Tools\TTS_SFT"

# MOSS-TTS 家族目标任务：决定输出 JSONL 的字段集合
# 可选值（依据 moss_tts_delay/finetuning/README.md §2.1-2.4）:
#   "moss_tts"             — MOSS-TTS / MOSS-TTS-Local（不含克隆，纯 text+speech）
#   "moss_tts_cloning"     — MOSS-TTS / MOSS-TTS-Local + 单说话人克隆训练（必填 REF_AUDIO_PATH）
#   "moss_ttsd"            — MOSS-TTSD（本脚本按单说话人处理：reference=[REF_AUDIO_PATH]；
#                            多说话人对话需另行手工编辑 JSONL）
#   "moss_voice_generator" — MOSS-VoiceGenerator（instruction 先用占位符，切分后手工补）
#   "moss_sound_effect"    — MOSS-SoundEffect（用 ambient_sound 字段；SRT 文本应直接为音效描述）
TARGET_TASK = "moss_tts"

# 语言代码（MOSS-TTS 家族官方 README 支持多语种；sound_effect 任务不使用该字段）
DEFAULT_LANGUAGE = "zh"

# 参考音频路径（仅 moss_tts_cloning / moss_ttsd 使用；相对 AUDIO_DIR 或绝对路径均可）
REF_AUDIO_PATH = ""

# 音频 + SRT 所在目录（绝对路径）
# 留空字符串 "" 则回退为"脚本所在目录"（保持与 VoxCPM 同款行为）
# 例如: r"D:\TTS_Training\raw_audio"  或  "/Users/me/tts/audio"
AUDIO_DIR = ""

# 输出目录：切片 WAV + train_raw.jsonl 写入的位置
# 留空字符串 "" 则默认使用 <AUDIO_DIR>/<OUT_SUBDIR>（与 VoxCPM 同款行为）
# 例如: r"D:\TTS_Training\datasets"
OUT_DIR = ""
OUT_SUBDIR = "datasets"               # 仅当 OUT_DIR 为空时作为 AUDIO_DIR 下的子目录名

# 要处理的音频和 SRT 文件（文件名，位于 AUDIO_DIR 下）
AUDIO_FILENAME = "01.wav"             # 音频文件名
SRT_FILENAME = "01.srt"               # SRT 字幕文件名

# ─── 步骤 1：响度归一化（母带） ───
ENABLE_NORMALIZATION = True           # 是否启用响度归一化（True=启用，False=跳过该步骤直接用原音频）
NORMALIZATION_METHOD = "loudnorm"     # 归一化方法："loudnorm"（EBU R128 响度，推荐）| "peak"（峰值）
TARGET_LUFS = -23.0                   # loudnorm 目标综合响度（EBU R128 广播标准，单位 LUFS）
TARGET_TP_DB = -1.5                   # loudnorm 真实峰值上限（单位 dBTP，防止削波）
TARGET_LRA = 11.0                     # loudnorm 目标响度范围（单位 LU，越小动态压得越狠）
TARGET_PEAK_DB = -1.0                 # peak 方法的目标峰值电平（单位 dBFS，仅当 METHOD="peak" 时生效）

# ─── 步骤 2：Silero VAD 切分参数 ───
BUFFER_MS = 100                       # SRT 时间定位 Buffer，前后扩展的搜索区域（单位：毫秒）
VAD_THRESHOLD = 0.5                   # Silero VAD 语音检测阈值（0-1，越高越严格）

# ─── 步骤 3：前后静音填充（尾静音精确裁剪始终开启，先裁剪再填充） ───
FRONT_SILENCE_MS = 250                # 切片前端添加的静音时长（单位：毫秒）
END_SILENCE_MS = 250                  # 切片尾端添加的静音时长（单位：毫秒；延用 VoxCPM 官方硬限制：≤ 500）
TAIL_SILENCE_THRESHOLD_DBFS = -40.0   # 尾静音检测阈值（单位 dBFS，低于此值视为静音区，用于精确裁剪）

# ─── 步骤 4：切片质量守门（只做纯静音排除） ───
MIN_VOICE_ENERGY_DBFS = -50.0         # 语音部分平均能量下限（单位 dBFS），低于此值判为纯静音并跳过

# ─── 母带中间文件管理 ───
KEEP_PREPROCESSED_FILE = False        # 处理完成后是否保留中间母带文件（True=保留便于检查，False=自动删除）

# ============================================================================
# 工具函数
# ============================================================================

def clear_screen() -> None:
    """清屏函数"""
    os.system('cls' if os.name == 'nt' else 'clear')

def get_script_dir() -> Path:
    """获取脚本所在目录"""
    return Path(__file__).parent.resolve()

def resolve_paths() -> Tuple[Path, Path]:
    """解析 AUDIO_DIR / OUT_DIR 两个配置项；空字符串时回退到 VoxCPM 同款行为。

    Returns:
        Tuple[Path, Path]: (audio_dir, out_dir)
          - audio_dir: AUDIO_DIR 有值则用它；否则用脚本所在目录（兼容原始行为）
          - out_dir:   OUT_DIR 有值则用它；否则用 audio_dir / OUT_SUBDIR
    """
    audio_dir = Path(AUDIO_DIR).expanduser().resolve() if AUDIO_DIR else get_script_dir()
    out_dir = Path(OUT_DIR).expanduser().resolve() if OUT_DIR else (audio_dir / OUT_SUBDIR)
    return audio_dir, out_dir

def measure_rms_dbfs(audio_array: Any) -> float:
    """计算音频数组的 RMS 能量（dBFS）"""
    import numpy as np
    arr = audio_array.astype(np.float64)
    if arr.size == 0:
        return float('-inf')
    rms = float(np.sqrt(np.mean(arr ** 2)))
    if rms < 1e-10:
        return float('-inf')
    return 20.0 * float(np.log10(rms))

def measure_tail_silence_samples(audio_array: Any, threshold_dbfs: float) -> int:
    """从尾部向前扫描，返回连续静音段的样本数"""
    import numpy as np
    if audio_array.size == 0:
        return 0
    threshold_amp = 10.0 ** (threshold_dbfs / 20.0)
    abs_arr = np.abs(audio_array.astype(np.float64))
    above = np.where(abs_arr > threshold_amp)[0]
    if len(above) == 0:
        return int(audio_array.size)
    last_voice_idx = int(above[-1])
    return int(audio_array.size - 1 - last_voice_idx)

def run_ffmpeg_loudnorm(input_path: Path, output_path: Path) -> bool:
    """
    调用 ffmpeg loudnorm 滤镜做 EBU R128 响度归一化

    ⚠️ 关键细节：ffmpeg 的 loudnorm 滤镜内部会把音频上采样到 192 kHz 做 EBU R128
    测量和增益调整，处理完不会自动降回原采样率。如果不加 `-ar` 参数，输出的
    WAV 文件会变成 192 kHz——这会让后续切片也变成 192 kHz，浪费 4 倍以上的磁盘
    并拖慢训练时的 dataloader 重采样。这里用 soundfile 读一下原采样率，然后
    通过 `-ar <orig_sr>` 强制 ffmpeg 输出回原采样率。
    """
    import soundfile as sf

    # 读取原采样率，用于 -ar 参数
    try:
        info = sf.info(str(input_path))
        orig_sr = int(info.samplerate)
    except Exception as e:
        print(f"  ❌ 无法读取输入文件采样率：{e}")
        return False

    filter_str = f"loudnorm=I={TARGET_LUFS}:TP={TARGET_TP_DB}:LRA={TARGET_LRA}"
    cmd = [
        'ffmpeg', '-y', '-hide_banner', '-loglevel', 'error',
        '-i', str(input_path),
        '-af', filter_str,
        '-ar', str(orig_sr),          # 强制输出回原采样率，防止 loudnorm 内部 192k 上采样残留
        '-c:a', 'pcm_s16le',
        str(output_path),
    ]
    try:
        result = subprocess.run(cmd, capture_output=True, text=True, timeout=1800)
    except FileNotFoundError:
        raise RuntimeError("系统未安装 ffmpeg，无法执行响度归一化。请安装 ffmpeg 后重试。")
    except subprocess.TimeoutExpired:
        print(f"  ❌ ffmpeg loudnorm 超时")
        return False
    if result.returncode != 0:
        print(f"  ❌ ffmpeg loudnorm 失败：\n{result.stderr.strip()[-600:]}")
        return False
    return True

def run_ffmpeg_peak_normalize(input_path: Path, output_path: Path) -> bool:
    """调用 ffmpeg 做峰值归一化（两遍：先 volumedetect 测量，再 volume 补偿）"""
    import re
    # 第一遍：测量当前峰值
    cmd_measure = [
        'ffmpeg', '-hide_banner',
        '-i', str(input_path),
        '-af', 'volumedetect',
        '-f', 'null', '-',
    ]
    try:
        result = subprocess.run(cmd_measure, capture_output=True, text=True, timeout=1800)
    except FileNotFoundError:
        raise RuntimeError("系统未安装 ffmpeg，无法执行峰值归一化。")
    m = re.search(r'max_volume:\s*(-?\d+(?:\.\d+)?)\s*dB', result.stderr)
    if not m:
        print(f"  ❌ 无法从 ffmpeg volumedetect 输出中读取 max_volume")
        return False
    current_peak_db = float(m.group(1))
    gain_db = TARGET_PEAK_DB - current_peak_db
    print(f"  当前峰值 {current_peak_db:+.2f} dBFS → 目标 {TARGET_PEAK_DB:+.2f} dBFS（补偿 {gain_db:+.2f} dB）")
    # 第二遍：应用增益
    cmd_apply = [
        'ffmpeg', '-y', '-hide_banner', '-loglevel', 'error',
        '-i', str(input_path),
        '-af', f'volume={gain_db}dB',
        '-c:a', 'pcm_s16le',
        str(output_path),
    ]
    result = subprocess.run(cmd_apply, capture_output=True, text=True, timeout=1800)
    if result.returncode != 0:
        print(f"  ❌ ffmpeg 峰值补偿失败：\n{result.stderr.strip()[-600:]}")
        return False
    return True

def prepare_master_audio(original_path: Path) -> Optional[Tuple[Path, List[Path]]]:
    """
    母带准备流程（只做响度归一化，保持原始采样率和声道数）

    说明：
    - 不需要强制重采样，因为 MOSS-Audio-Tokenizer 内部按 24 kHz 统一处理
      （见根 README "MOSS-Audio-Tokenizer" 章节 line 634）
    - 不需要强制单声道化，Python 层在 segment_audio_with_vad 里用 np.mean 兜底

    Returns:
        Tuple[Path, List[Path]]: (最终母带路径, 产生的所有中间文件列表)
        失败返回 None
    """
    script_dir = original_path.parent
    stem = original_path.stem
    intermediate_files: List[Path] = []
    current_path = original_path

    # 步骤 1：响度归一化
    print("-" * 60)
    print("步骤 1：响度归一化")
    if ENABLE_NORMALIZATION:
        normalized_path = script_dir / f"{stem}_normalized.wav"
        if NORMALIZATION_METHOD == "loudnorm":
            print(f"  方法：EBU R128 loudnorm")
            print(f"  参数：I={TARGET_LUFS} LUFS, TP={TARGET_TP_DB} dBTP, LRA={TARGET_LRA} LU")
            ok = run_ffmpeg_loudnorm(current_path, normalized_path)
        elif NORMALIZATION_METHOD == "peak":
            print(f"  方法：Peak normalization → {TARGET_PEAK_DB} dBFS")
            ok = run_ffmpeg_peak_normalize(current_path, normalized_path)
        else:
            print(f"  ❌ 未知的 NORMALIZATION_METHOD：{NORMALIZATION_METHOD}")
            return None
        if not ok:
            return None
        intermediate_files.append(normalized_path)
        current_path = normalized_path
        print(f"  ✓ 响度归一化完成 → {normalized_path.name}")
    else:
        print("  ⊘ 已禁用，跳过")
    print()

    return (current_path, intermediate_files)

def get_last_segment_number(jsonl_path: Path) -> int:
    """
    读取 JSONL 文件，获取最后一个音频片段的序号

    Args:
        jsonl_path: JSONL 文件路径

    Returns:
        int: 最后的序号，如果文件不存在或为空返回 0
    """
    if not jsonl_path.exists():
        return 0

    try:
        with open(jsonl_path, 'r', encoding='utf-8') as f:
            lines = f.readlines()

        if not lines:
            return 0

        # 读取最后一行
        last_line = lines[-1].strip()
        if not last_line:
            return 0

        data = json.loads(last_line)
        audio_path = Path(data['audio'])
        filename = audio_path.stem  # 不含扩展名的文件名

        # 提取序号（如 "0010" -> 10）
        try:
            return int(filename)
        except ValueError:
            print(f"  警告：无法解析文件名中的序号：{filename}，从 0 开始")
            return 0

    except Exception as e:
        print(f"  警告：读取 JSONL 文件失败：{e}，从 0 开始")
        return 0

# ============================================================================
# 环境检查
# ============================================================================

VALID_TASKS = {
    "moss_tts", "moss_tts_cloning", "moss_ttsd",
    "moss_voice_generator", "moss_sound_effect",
}

def check_prerequisites() -> bool:
    """检查前置条件：文件存在性与配置项合法性"""
    audio_dir, _ = resolve_paths()

    # 检查 AUDIO_DIR 存在
    if not audio_dir.exists():
        print(f"错误：AUDIO_DIR 不存在")
        print(f"路径：{audio_dir}")
        print(f"请检查配置常量 AUDIO_DIR 是否正确（留空字符串则使用脚本所在目录）")
        return False

    # 检查音频文件
    audio_path = audio_dir / AUDIO_FILENAME
    if not audio_path.exists():
        print(f"错误：音频文件不存在")
        print(f"路径：{audio_path}")
        print(f"请检查配置常量 AUDIO_DIR / AUDIO_FILENAME 是否正确")
        return False

    # 检查 SRT 文件
    srt_path = audio_dir / SRT_FILENAME
    if not srt_path.exists():
        print(f"错误：SRT 文件不存在")
        print(f"路径：{srt_path}")
        print(f"请检查配置常量 AUDIO_DIR / SRT_FILENAME 是否正确")
        return False

    # 配置项合法性：尾静音硬上限
    # MOSS-TTS 官方未明确给出尾静音阈值，这里沿用 VoxCPM 官方的 ≤ 500 ms 经验，
    # 避免训练目标里学到过长尾静音
    if END_SILENCE_MS > 500:
        print(f"错误：END_SILENCE_MS={END_SILENCE_MS} ms 超过 500 ms 硬上限")
        print(f"VoxCPM 官方警告：尾静音过长（>0.5s）会导致生成停不下来；本脚本延用此经验")
        print(f"请将 END_SILENCE_MS 调整为 ≤ 500 后重试")
        return False

    if FRONT_SILENCE_MS < 0 or END_SILENCE_MS < 0:
        print(f"错误：静音时长不能为负数")
        return False

    # 归一化方法合法性
    if ENABLE_NORMALIZATION and NORMALIZATION_METHOD not in ("loudnorm", "peak"):
        print(f"错误：未知的 NORMALIZATION_METHOD：{NORMALIZATION_METHOD}")
        print(f"可选值：'loudnorm' | 'peak'")
        return False

    # MOSS-TTS 家族任务参数合法性
    if TARGET_TASK not in VALID_TASKS:
        print(f"错误：未知的 TARGET_TASK：{TARGET_TASK}")
        print(f"可选值：{' | '.join(sorted(VALID_TASKS))}")
        return False

    if TARGET_TASK == "moss_tts_cloning" and not REF_AUDIO_PATH:
        print(f"错误：TARGET_TASK=moss_tts_cloning 必须设置 REF_AUDIO_PATH")
        print(f"请填写单说话人参考音频的路径")
        return False

    if TARGET_TASK == "moss_ttsd" and not REF_AUDIO_PATH:
        print(f"警告：TARGET_TASK=moss_ttsd 未设置 REF_AUDIO_PATH")
        print(f"输出 JSONL 将没有 reference 字段；如需多说话人请自行手工编辑 JSONL")

    return True

# ============================================================================
# SRT 解析
# ============================================================================

def parse_srt_time(time_str: str) -> float:
    """
    将 SRT 时间格式转换为秒数

    Args:
        time_str: SRT 时间格式字符串，如 "00:01:23,456"

    Returns:
        float: 时间（秒）

    Raises:
        ValueError: 如果时间格式不正确
    """
    try:
        time_str = time_str.strip()
        hours, minutes, seconds_ms = time_str.split(':')
        seconds, milliseconds = seconds_ms.split(',')

        total_seconds = (
            int(hours) * 3600 +
            int(minutes) * 60 +
            int(seconds) +
            int(milliseconds) / 1000.0
        )

        return total_seconds
    except (ValueError, IndexError) as e:
        raise ValueError(f"无效的 SRT 时间格式：{time_str}") from e

def parse_srt_file(srt_path: Path) -> List[Dict[str, Any]]:
    """
    解析 SRT 文件

    Args:
        srt_path: SRT 文件路径

    Returns:
        list: 包含段落信息的列表
    """
    segments = []

    with open(srt_path, 'r', encoding='utf-8') as f:
        lines = f.readlines()

    i = 0
    while i < len(lines):
        line = lines[i].strip()

        # 跳过空行
        if not line:
            i += 1
            continue

        # 段落编号
        try:
            segment_num = int(line)
        except ValueError:
            i += 1
            continue

        # 时间轴
        i += 1
        if i >= len(lines):
            break

        time_line = lines[i].strip()
        if '-->' not in time_line:
            continue

        try:
            parts = time_line.split('-->')
            if len(parts) != 2:
                print(f"  警告：段落 {segment_num} 时间轴格式错误，跳过")
                i += 1
                continue

            start_str, end_str = parts
            start_time = parse_srt_time(start_str)
            end_time = parse_srt_time(end_str)
        except ValueError as e:
            print(f"  警告：段落 {segment_num} 时间轴解析失败：{e}，跳过")
            i += 1
            continue

        # 文本内容（可能多行）
        i += 1
        text_lines = []
        while i < len(lines) and lines[i].strip():
            text_lines.append(lines[i].strip())
            i += 1

        text = ' '.join(text_lines)

        segments.append({
            'index': segment_num,
            'start': start_time,
            'end': end_time,
            'duration': end_time - start_time,
            'text': text
        })

        i += 1

    return segments

# ============================================================================
# 音频处理 - 辅助函数
# ============================================================================

def get_duration_with_ffprobe(audio_path: Path) -> float:
    """
    使用 ffprobe 获取音频文件的精确时长

    Args:
        audio_path: 音频文件路径

    Returns:
        float: 音频时长（秒）
    """
    import soundfile as sf

    try:
        result = subprocess.run(
            [
                'ffprobe',
                '-v', 'error',
                '-show_entries', 'format=duration',
                '-of', 'default=noprint_wrappers=1:nokey=1',
                str(audio_path)
            ],
            capture_output=True,
            text=True,
            timeout=10
        )

        if result.returncode == 0 and result.stdout.strip():
            return float(result.stdout.strip())
    except Exception as e:
        print(f"  警告：ffprobe 获取时长失败，使用 soundfile：{e}")

    # 降级方案：使用 soundfile
    audio, sr = sf.read(audio_path)
    return len(audio) / sr

def build_jsonl_record(item: Dict[str, Any]) -> Dict[str, Any]:
    """
    根据 TARGET_TASK 把基础记录 {audio, text, duration} 转换成对应任务的 JSONL 字段。

    依据：
      - moss_tts_delay/finetuning/README.md §2.1  (moss_tts, moss_tts_cloning)
      - moss_tts_delay/finetuning/README.md §2.2  (moss_ttsd)
      - moss_tts_delay/finetuning/README.md §2.3  (moss_sound_effect)
      - moss_tts_delay/finetuning/README.md §2.4  (moss_voice_generator)
      - moss_tts_local/finetuning/README.md §2.1-2.4（字段与 delay 完全一致）

    注：基础记录里的 `duration` 字段仅用于切分阶段的统计，不写入最终 JSONL
    （MOSS-TTS 官方 JSONL 不要求 duration 字段）。
    """
    audio_path = item['audio']
    text = item['text']

    if TARGET_TASK == "moss_tts":
        return {"audio": audio_path, "text": text, "language": DEFAULT_LANGUAGE}

    elif TARGET_TASK == "moss_tts_cloning":
        rec = {"audio": audio_path, "text": text, "language": DEFAULT_LANGUAGE}
        if REF_AUDIO_PATH:
            rec["ref_audio"] = REF_AUDIO_PATH
        return rec

    elif TARGET_TASK == "moss_ttsd":
        rec = {"audio": audio_path, "text": text, "language": DEFAULT_LANGUAGE}
        if REF_AUDIO_PATH:
            rec["reference"] = [REF_AUDIO_PATH]
        return rec

    elif TARGET_TASK == "moss_voice_generator":
        return {
            "audio": audio_path,
            "text": text,
            "instruction": "TODO: describe the voice style here",
        }

    elif TARGET_TASK == "moss_sound_effect":
        return {"audio": audio_path, "ambient_sound": text}

    else:
        raise ValueError(f"未知 TARGET_TASK: {TARGET_TASK}")

def process_single_segment(
    seg: Dict[str, Any],
    audio: Any,
    sr: int,
    audio_duration: float,
    output_dir: Path,
    current_index: int,
    vad_model: Any,
    get_speech_timestamps: Any,
    resampler: Any,
    target_sr: int,
    buffer_samples: int,
    front_silence_samples: int,
    end_silence_samples: int
) -> Optional[Tuple[Optional[Dict[str, Any]], Optional[str], Optional[str]]]:
    """
    处理单个音频片段

    Returns:
        Tuple[Optional[Dict], Optional[str], Optional[str]]:
            - (training_item, vad_warning, None)       — 成功
            - (None, None, skip_reason)                — 主动跳过（质量守门）
            - None                                     — 未预料的硬错误
    """
    import numpy as np
    import soundfile as sf
    import torch

    srt_start = seg['start']
    srt_end = seg['end']
    text = seg['text']

    # 1. 计算扩展后的搜索区域（加 buffer）
    search_start_time = max(0, srt_start - BUFFER_MS / 1000)
    search_end_time = min(audio_duration, srt_end + BUFFER_MS / 1000)

    search_start_sample = int(search_start_time * sr)
    search_end_sample = int(search_end_time * sr)

    # 提取搜索区域
    search_region = audio[search_start_sample:search_end_sample]

    if len(search_region) == 0:
        reason = f"段落 {current_index:04d} [SRT {srt_start:.2f}s-{srt_end:.2f}s]: 搜索区域为空"
        return (None, None, reason)

    # 2. 使用 Silero VAD 在搜索区域内精准检测（16kHz）
    vad_warning: Optional[str] = None
    try:
        # 转换为 torch.Tensor
        search_tensor = torch.from_numpy(search_region.astype(np.float32))

        # 重采样到 16000 Hz（仅用于 VAD 检测）
        search_16k = resampler(search_tensor)

        # VAD 检测
        speech_timestamps = get_speech_timestamps(
            search_16k,
            vad_model,
            sampling_rate=target_sr,
            threshold=VAD_THRESHOLD,
            return_seconds=True
        )

        # 3. 确定精准的语音边界
        if speech_timestamps:
            # VAD 检测成功：使用第一个和最后一个语音段
            first_segment = speech_timestamps[0]
            last_segment = speech_timestamps[-1]

            # 语音在搜索区域内的相对位置（秒）
            relative_start = first_segment['start']
            relative_end = last_segment['end']

            # 转换为原始音频的绝对位置（样本点，保持原始采样率）
            absolute_start_sample = search_start_sample + int(relative_start * sr)
            absolute_end_sample = search_start_sample + int(relative_end * sr)
        else:
            # VAD 检测失败：退回到 SRT 时间轴
            vad_warning = f"段落 {current_index:04d}: VAD 未检测到语音，已退回到 SRT 时间轴"
            absolute_start_sample = int(srt_start * sr)
            absolute_end_sample = int(srt_end * sr)

        # 边界检查
        absolute_start_sample = max(0, absolute_start_sample)
        absolute_end_sample = min(absolute_end_sample, len(audio))

        if absolute_end_sample <= absolute_start_sample:
            reason = f"段落 {current_index:04d} [SRT {srt_start:.2f}s-{srt_end:.2f}s]: 无效的时间范围"
            return (None, None, reason)

        # 4. 提取精准的语音内容（保持原始采样率）
        audio_content = audio[absolute_start_sample:absolute_end_sample]

        # 4a. 尾静音精确裁剪（必做：在填充之前把 VAD 残留的尾静音刮干净）
        if len(audio_content) > 0:
            tail_sil_samples = measure_tail_silence_samples(
                audio_content, TAIL_SILENCE_THRESHOLD_DBFS
            )
            if tail_sil_samples > 0 and tail_sil_samples < len(audio_content):
                audio_content = audio_content[:len(audio_content) - tail_sil_samples]

        if len(audio_content) == 0:
            reason = f"段落 {current_index:04d} [SRT {srt_start:.2f}s-{srt_end:.2f}s]: 尾静音裁剪后内容为空"
            return (None, None, reason)

        # 5. 语音能量守门（跳过纯静音片段）
        voice_dbfs = measure_rms_dbfs(audio_content)
        if voice_dbfs < MIN_VOICE_ENERGY_DBFS:
            reason = (
                f"段落 {current_index:04d} [SRT {srt_start:.2f}s-{srt_end:.2f}s]: "
                f"语音能量 {voice_dbfs:.1f} dBFS 低于阈值 {MIN_VOICE_ENERGY_DBFS} dBFS（判为纯静音）"
            )
            return (None, None, reason)

        # 6. 添加前后静音
        front_silence = np.zeros(front_silence_samples, dtype=audio.dtype)
        end_silence = np.zeros(end_silence_samples, dtype=audio.dtype)
        audio_final = np.concatenate([front_silence, audio_content, end_silence])

        # 7. 保存音频文件（采样率继承母带，MOSS-Audio-Tokenizer 训练时按 24 kHz 统一处理）
        output_filename = f"{current_index:04d}.wav"
        output_path = output_dir / output_filename
        sf.write(output_path, audio_final, sr)

        # 8. 使用 ffprobe 获取精确 duration（仅用于切分阶段的统计，不写入最终 JSONL）
        duration = get_duration_with_ffprobe(output_path)

        # 9. 构造基础训练数据（JSONL 字段由 build_jsonl_record 在写入时按 TARGET_TASK 分支产出）
        training_item: Dict[str, Any] = {
            'audio': str(output_path.absolute()),
            'text': text,
            'duration': round(duration, 2),
        }

        return (training_item, vad_warning, None)

    except (IOError, ValueError, RuntimeError) as e:
        print(f"  ⚠️  段落 {current_index:04d} 处理失败: {e}")
        import traceback
        traceback.print_exc()
        return None
    except KeyboardInterrupt:
        print(f"\n用户中断处理")
        raise

# ============================================================================
# 音频处理 - 主函数
# ============================================================================

def segment_audio_with_vad(
    audio_path: Path,
    segments: List[Dict[str, Any]],
    output_dir: Path,
    vad_model: Any,
    get_speech_timestamps: Any,
    start_index: int,
    buffer_ms: int = BUFFER_MS,
    front_silence_ms: int = FRONT_SILENCE_MS,
    end_silence_ms: int = END_SILENCE_MS
) -> Tuple[List[Dict[str, Any]], List[str], List[str]]:
    """
    使用 Silero VAD 精准切分母带音频

    处理流程：
    1. 根据 SRT 时间轴大致定位（扩展 buffer）
    2. 在该区域内使用 VAD 精准检测语音边界（内部临时重采样到 16kHz 给 Silero VAD）
    3. 提取精准的语音内容（保持母带原始采样率），并精确裁剪尾静音
    4. 质量守门：纯静音能量检查
    5. 添加前后静音填充
    6. 保存并生成基础 JSONL 条目（{audio, text, duration}）

    Args:
        audio_path: 母带音频文件路径（已完成响度归一化，保持原始采样率和声道数）
        segments: SRT 段落列表
        output_dir: 输出目录
        vad_model: Silero VAD 模型
        get_speech_timestamps: VAD 工具函数
        start_index: 起始序号
        buffer_ms: SRT 时间定位 Buffer（毫秒）
        front_silence_ms: 前端填充静音时长（毫秒）
        end_silence_ms: 尾端填充静音时长（毫秒）

    Returns:
        Tuple[List, List, List]: (训练数据列表, VAD 回退警告列表, 被跳过片段原因列表)
    """
    import numpy as np
    import soundfile as sf
    import torchaudio
    from tqdm import tqdm

    # 加载母带音频（保持原始采样率）
    print(f"📂 加载母带: {audio_path.name}")
    audio, sr = sf.read(audio_path)

    # 立体声 → 单声道（训练管道期望单声道；Python 层兜底，不依赖 ffmpeg）
    if len(audio.shape) > 1:
        audio = np.mean(audio, axis=1)

    audio_duration = len(audio) / sr
    print(f"✅ 母带加载成功 - 采样率: {sr} Hz, 时长: {audio_duration:.2f} 秒")

    # 创建输出目录
    os.makedirs(output_dir, exist_ok=True)

    # Silero VAD 需要 16000 Hz，准备重采样器（仅用于 VAD 检测）
    target_sr = 16000
    resampler = torchaudio.transforms.Resample(orig_freq=sr, new_freq=target_sr)

    # 处理数据
    training_data: List[Dict[str, Any]] = []
    vad_fallback_warnings: List[str] = []
    skip_reasons: List[str] = []

    buffer_samples = int(buffer_ms / 1000 * sr)
    front_silence_samples = int(front_silence_ms / 1000 * sr)
    end_silence_samples = int(end_silence_ms / 1000 * sr)

    current_index = start_index

    for seg in tqdm(segments, desc="切分音频"):
        current_index += 1

        result = process_single_segment(
            seg=seg,
            audio=audio,
            sr=sr,
            audio_duration=audio_duration,
            output_dir=output_dir,
            current_index=current_index,
            vad_model=vad_model,
            get_speech_timestamps=get_speech_timestamps,
            resampler=resampler,
            target_sr=target_sr,
            buffer_samples=buffer_samples,
            front_silence_samples=front_silence_samples,
            end_silence_samples=end_silence_samples,
        )

        if result is None:
            # 未预料的硬错误，算作跳过但原因未知
            skip_reasons.append(f"段落 {current_index:04d}: 未预料的处理错误（见上方 traceback）")
            continue

        training_item, vad_warning, skip_reason = result

        if skip_reason is not None:
            skip_reasons.append(skip_reason)
            tqdm.write(f"  ⊘ {skip_reason}")
            continue

        if training_item is not None:
            training_data.append(training_item)

        if vad_warning:
            vad_fallback_warnings.append(vad_warning)
            tqdm.write(f"  ⚠️  {vad_warning}")

    print(f"\n✅ 切分完成: {len(training_data)} 个有效片段")
    if skip_reasons:
        print(f"⊘  跳过了 {len(skip_reasons)} 个无效片段（详情见末尾汇总）")

    return training_data, vad_fallback_warnings, skip_reasons

# ============================================================================
# 主程序
# ============================================================================

def run_segment_audio() -> None:
    """执行音频切分（自动启动虚拟环境）"""
    # 检查虚拟环境目录是否存在
    tts_sft_path = Path(TTS_SFT_PROJECT_PATH)
    if not tts_sft_path.exists():
        print(f"错误：TTS_SFT 项目目录不存在")
        print(f"路径：{tts_sft_path}")
        print(f"请检查配置常量 TTS_SFT_PROJECT_PATH 是否正确")
        print(f"程序终止！")
        sys.exit(1)

    # 检查虚拟环境 Python 是否存在
    venv_python = tts_sft_path / "runtime" / "Scripts" / "python.exe"
    if not venv_python.exists():
        print(f"错误：TTS_SFT Python 虚拟环境不存在")
        print(f"路径：{venv_python}")
        print(f"请确认虚拟环境已正确创建")
        print(f"程序终止！")
        sys.exit(1)

    # 检查当前 Python 环境
    current_python = Path(sys.executable).resolve()

    # 如果不在虚拟环境中，自动启动虚拟环境
    if current_python != venv_python:
        print(f"检测到未在虚拟环境中运行")
        print(f"启动 TTS_SFT Python 虚拟环境...")
        print()

        # 设置环境变量
        env = os.environ.copy()

        # 设置 Silero VAD 模型下载目录
        models_dir = tts_sft_path / "models"
        models_dir.mkdir(parents=True, exist_ok=True)
        env['TORCH_HOME'] = str(models_dir)

        # 使用虚拟环境的 Python 重新运行本脚本（传递所有命令行参数）
        result = subprocess.run(
            [str(venv_python), __file__] + sys.argv[1:],
            env=env
        )

        if result.returncode != 0:
            print(f"错误：音频切分进程异常退出，返回码：{result.returncode}")
            sys.exit(1)

        # 子进程执行完毕，退出主进程
        sys.exit(0)

    # 到这里说明已经在虚拟环境中了
    print("TTS_SFT 虚拟环境启动完毕 ✓")
    print()

    # 执行实际的音频切分流程
    execute_segmentation()

def execute_segmentation() -> None:
    """执行音频切分的实际逻辑（在虚拟环境中运行）"""
    # 解析路径：AUDIO_DIR / OUT_DIR 为空时回退到脚本所在目录（与 VoxCPM 同款行为）
    audio_dir, out_dir = resolve_paths()

    print(f"📁 脚本目录: {get_script_dir()}")
    print(f"📁 音频目录: {audio_dir}")
    print(f"📁 输出目录: {out_dir}")
    print(f"📁 虚拟环境: {TTS_SFT_PROJECT_PATH}")
    print()

    # 检查前置条件
    print("检查前置条件...")
    if not check_prerequisites():
        sys.exit(1)
    print("前置条件检查通过 ✓")
    print()

    print(f"⚙️  配置参数:")
    print(f"   - 目标任务:   {TARGET_TASK}")
    print(f"   - 默认语言:   {DEFAULT_LANGUAGE}")
    print(f"   - 参考音频:   {REF_AUDIO_PATH if REF_AUDIO_PATH else '(未设置)'}")
    print(f"   - 音频文件:   {AUDIO_FILENAME}")
    print(f"   - SRT 文件:   {SRT_FILENAME}")
    if ENABLE_NORMALIZATION:
        if NORMALIZATION_METHOD == "loudnorm":
            print(f"   - 响度归一化: loudnorm I={TARGET_LUFS} LUFS, TP={TARGET_TP_DB} dBTP, LRA={TARGET_LRA}")
        else:
            print(f"   - 响度归一化: peak → {TARGET_PEAK_DB} dBFS")
    else:
        print(f"   - 响度归一化: 已禁用")
    print(f"   - Buffer 时长: {BUFFER_MS} ms")
    print(f"   - VAD 阈值: {VAD_THRESHOLD}")
    print(f"   - 前后静音: 前 {FRONT_SILENCE_MS} ms / 尾 {END_SILENCE_MS} ms（尾静音始终精确裁剪后再填充）")
    print(f"   - 纯静音守门: 能量 ≥ {MIN_VOICE_ENERGY_DBFS} dBFS")
    print()

    # 设置路径
    audio_path = audio_dir / AUDIO_FILENAME
    srt_path = audio_dir / SRT_FILENAME
    data_dir = out_dir
    data_dir.mkdir(parents=True, exist_ok=True)
    jsonl_path = data_dir / "train_raw.jsonl"

    # 检查追加模式
    start_index = get_last_segment_number(jsonl_path)
    if start_index > 0:
        print(f"📝 检测到现有训练数据，追加模式启动")
        print(f"   上次最后序号: {start_index:04d}")
        print(f"   本次起始序号: {start_index + 1:04d}")
        print()
    else:
        print(f"📝 新建训练数据，从序号 0001 开始")
        print()

    # 导入依赖
    print("-" * 60)
    print("导入依赖库...")
    try:
        import torch
        import torchaudio
        import numpy as np
        import soundfile as sf
        from tqdm import tqdm
    except ImportError as e:
        print(f"错误：缺少必要的库 - {e}")
        print("\n请在虚拟环境中安装依赖：")
        print("pip install torch torchaudio numpy soundfile tqdm")
        sys.exit(1)
    print("依赖库导入成功 ✓")
    print()

    # 加载 Silero VAD 模型
    print("-" * 60)
    print("加载 Silero VAD 模型...")

    # 设置 Silero VAD 模型下载目录
    tts_sft_path = Path(TTS_SFT_PROJECT_PATH)
    models_dir = tts_sft_path / "models"
    models_dir.mkdir(parents=True, exist_ok=True)

    # 设置 torch.hub 模型目录
    torch.hub.set_dir(str(models_dir))

    # 同时设置环境变量
    os.environ['TORCH_HOME'] = str(models_dir)

    print(f"VAD 模型目录: {models_dir}")

    try:
        vad_model, utils = torch.hub.load(
            repo_or_dir='snakers4/silero-vad',
            model='silero_vad',
            force_reload=False
        )
        get_speech_timestamps = utils[0]
        print("Silero VAD 模型加载完成 ✓")
        print()
    except Exception as e:
        print(f"❌ Silero VAD 模型加载失败: {e}")
        print("\n提示：首次使用会自动下载模型到指定目录")
        print(f"模型目录: {models_dir}")
        import traceback
        traceback.print_exc()
        sys.exit(1)

    # 解析 SRT
    print("-" * 60)
    print("解析 SRT 文件...")
    segments = parse_srt_file(srt_path)
    print(f"📝 解析完成: {len(segments)} 个段落")
    print()

    # 准备母带（步骤 1：响度归一化，保持原采样率和声道）
    master_result = prepare_master_audio(audio_path)
    if master_result is None:
        print("❌ 母带准备失败，程序终止")
        sys.exit(1)
    master_path, intermediate_files = master_result

    # 切分音频（步骤 2 VAD 切分 + 步骤 3 静音填充 + 步骤 4 质量守门）
    print("-" * 60)
    print("步骤 2-4：VAD 切分 / 静音填充 / 质量守门")
    print()

    training_data, vad_warnings, skip_reasons = segment_audio_with_vad(
        audio_path=master_path,
        segments=segments,
        output_dir=data_dir,
        vad_model=vad_model,
        get_speech_timestamps=get_speech_timestamps,
        start_index=start_index,
        buffer_ms=BUFFER_MS,
        front_silence_ms=FRONT_SILENCE_MS,
        end_silence_ms=END_SILENCE_MS,
    )

    print()

    # 检查是否有数据
    if len(training_data) == 0:
        print("=" * 60)
        print("⚠️  警告：未成功切分任何音频片段！")
        print("=" * 60)
        print("可能原因：")
        print("  - SRT 时间轴全部无效")
        print("  - VAD 检测全部失败且搜索区域为空")
        print("  - 音频文件损坏")
        print()
        sys.exit(1)

    # 保存/追加 JSONL 训练清单（按 TARGET_TASK 选字段）
    print("-" * 60)
    print(f"保存训练数据清单（TARGET_TASK={TARGET_TASK}）...")

    os.makedirs(data_dir, exist_ok=True)

    # 追加模式：打开文件并追加
    mode = 'a' if start_index > 0 else 'w'
    with open(jsonl_path, mode, encoding='utf-8') as f:
        for item in training_data:
            record = build_jsonl_record(item)
            f.write(json.dumps(record, ensure_ascii=False) + '\n')

    print(f"✅ 数据已保存到: {jsonl_path}")
    print()

    # 统计信息
    total_duration = sum(item['duration'] for item in training_data)

    print("-" * 60)
    print("📊 本次处理统计:")
    print(f"   - 片段数: {len(training_data)}")
    print(f"   - 总时长: {total_duration / 60:.2f} 分钟 ({total_duration:.2f} 秒)")
    print(f"   - 平均时长: {total_duration / len(training_data):.2f} 秒/片段")
    print(f"   - 起始序号: {start_index + 1:04d}")
    print(f"   - 结束序号: {start_index + len(training_data):04d}")

    # 时长分布
    durations = [item['duration'] for item in training_data]
    under = sum(1 for d in durations if d < 3)
    in_range = sum(1 for d in durations if 3 <= d <= 25)
    over = sum(1 for d in durations if d > 25)

    print(f"\n📊 时长分布:")
    print(f"   - < 3 秒:  {under} ({under/len(durations)*100:.1f}%)")
    print(f"   - 3-25 秒: {in_range} ({in_range/len(durations)*100:.1f}%)")
    print(f"   - > 25 秒: {over} ({over/len(durations)*100:.1f}%)")
    print()

    # VAD 退回警告
    if vad_warnings:
        print("-" * 60)
        print(f"⚠️  VAD 检测退回警告 ({len(vad_warnings)} 条)")
        print("-" * 60)
        print("以下片段 VAD 未检测到语音，已退回使用 SRT 时间轴：\n")
        for warning in vad_warnings:
            print(f"   - {warning}")
        print()

    # 质量守门跳过清单（步骤 4 的汇总报告）
    if skip_reasons:
        print("-" * 60)
        print(f"⊘  质量守门跳过清单 ({len(skip_reasons)} 条)")
        print("-" * 60)
        print("以下片段未通过质量守门检查，未写入训练清单：\n")
        for reason in skip_reasons:
            print(f"   - {reason}")
        print()

    # 清理中间母带文件
    if not KEEP_PREPROCESSED_FILE:
        cleaned = []
        for f in intermediate_files:
            try:
                if f.exists():
                    f.unlink()
                    cleaned.append(f.name)
            except OSError as e:
                print(f"⚠️  清理中间文件失败 {f.name}: {e}")
        if cleaned:
            print(f"🧹 已清理中间母带文件: {', '.join(cleaned)}")
            print()
    else:
        print(f"💾 已保留中间母带文件:")
        for f in intermediate_files:
            print(f"   - {f}")
        print()

    print("=" * 60)
    print("🎉 处理完成！")
    print("=" * 60)
    print()
    print("📂 输出文件:")
    print(f"   - 音频片段: {data_dir}")
    print(f"   - 训练清单: {jsonl_path}")
    print()

def main() -> None:
    """主程序入口"""
    clear_screen()

    print("=" * 60)
    print("🎵 MOSS-TTS 训练数据准备 - 基于 SRT + Silero VAD 精准切分")
    print("=" * 60)
    print()

    # 运行音频切分（自动处理虚拟环境）
    run_segment_audio()

if __name__ == '__main__':
    main()
```

**本地运行环境要求**（**不在 AutoDL 上跑**）：

- **`TTS_SFT_PROJECT_PATH` 指向的目录里有一个可用的 Python 虚拟环境**，目录布局需满足 `<TTS_SFT_PROJECT_PATH>/runtime/Scripts/python.exe`（Windows 便携式 Python / `venv --copies` 默认布局）。直接复用 VoxCPM 教程里建好的 `VoxCPM_FT` 虚拟环境即可，依赖完全一致
- 该虚拟环境里已装：`torch`、`torchaudio`、`numpy`、`soundfile`、`tqdm`（VoxCPM 教程的本地环境默认就有这些）
- 系统级装了 `ffmpeg` / `ffprobe`，在 PATH 里能直接调用（脚本用 ffmpeg 做响度归一化、用 ffprobe 读精确时长）
- 首次运行脚本时，会从 GitHub 拉取 Silero VAD 模型（由脚本自动重定向到 `<TTS_SFT_PROJECT_PATH>/models/`）→ 家庭/办公网络通常直连即可；如果 GitHub 访问受限，在本机开代理（VPN/Clash）后再跑

**运行方式**（脚本自动检测并切到虚拟环境，无需手动 activate）：

```powershell
# Windows PowerShell（直接用系统 Python 或任意 Python 跑都行，脚本内部会重启到 TTS_SFT 虚拟环境）
python "D:\TTS_Training\segment_audio.py"
```

```bash
# macOS / Linux（注意：run_segment_audio() 里默认用 Windows 路径 runtime\Scripts\python.exe，
# 跨平台用户需要把该路径按你的 venv 布局改成 bin/python）
python /path/to/segment_audio.py
```

> 首次跑会看到 `检测到未在虚拟环境中运行 / 启动 TTS_SFT Python 虚拟环境...` 提示，这是脚本把自己重新启动到 `TTS_SFT_PROJECT_PATH` 指定的 venv 里，属于正常流程。

### 3.5 切好之后上传到云端

切分产物（由 `OUT_DIR` 决定，默认是 `<AUDIO_DIR>/datasets/`）：

```
<OUT_DIR>/
├── 0001.wav
├── 0002.wav
├── ...
└── train_raw.jsonl        ← JSONL 里 "audio" 是**本地绝对路径**
```

用 WinSCP / scp / rsync 把整个 `<OUT_DIR>` 传到 AutoDL 的 `/root/autodl-tmp/datasets/`。

> ⚠️ **JSONL 里的本地路径必须重写为云端路径**。脚本在本地写的 JSONL 里 `"audio": "D:\\TTS_Training\\raw_audio\\datasets\\0001.wav"` 之类，到云端后要替换成 `"audio": "/root/autodl-tmp/datasets/0001.wav"`。在 AutoDL 上执行：
>
> ```bash
> cd /root/autodl-tmp/datasets
> # 反斜杠转正斜杠、去盘符、替换本地目录前缀为云端前缀（把左侧 D:\\TTS_Training\\raw_audio\\datasets\\ 换成你的实际本地前缀）
> python -c "
> import json, sys
> PREFIX_OLD = 'D:\\\\TTS_Training\\\\raw_audio\\\\datasets\\\\'
> PREFIX_NEW = '/root/autodl-tmp/datasets/'
> with open('train_raw.jsonl','r',encoding='utf-8') as f, open('train_raw.jsonl.new','w',encoding='utf-8') as g:
>     for line in f:
>         d = json.loads(line)
>         if 'audio' in d:
>             d['audio'] = d['audio'].replace(PREFIX_OLD, PREFIX_NEW).replace('\\\\', '/')
>         g.write(json.dumps(d, ensure_ascii=False)+'\n')
> "
> mv train_raw.jsonl.new train_raw.jsonl
> ```
>
> 同样的路径重写对 `ref_audio` / `reference` / `reference_audio` / `ref_wav` 字段也适用——如果 JSONL 里包含它们，把上面脚本里的 `'audio'` 换成对应字段名跑一遍，或统一对整行做字符串替换。

---

## 四、特殊任务的数据组织

### 4.1 MOSS-VoiceGenerator — 补 instruction

`segment_audio.py` 对 `moss_voice_generator` 任务只能生成占位 instruction，你需要**逐条用自然语言描述音色**再训练。举例（见 `moss_tts_delay/finetuning/README.md §2.4`）：

```jsonl
{"audio":"/root/autodl-tmp/datasets/0001.wav","text":"My old back is really giving me trouble these days.","instruction":"A tired, hoarse elderly voice complaining slowly with a faint groan."}
{"audio":"/root/autodl-tmp/datasets/0002.wav","text":"Hey there, stranger!","instruction":"Hearty, jovial tavern owner's voice, loud and welcoming with a slightly gruff tone."}
```

可以用 AI（如 Claude）批量听切片后生成 instruction，或人工书写。

### 4.2 MOSS-SoundEffect — 场景描述即文本

`segment_audio.py` 对 `moss_sound_effect` 任务直接把 SRT 的文本原样放进 `ambient_sound`。这要求你的 SRT **本身就是音效描述**，不是对白。示例（见 `§2.3`）：

```jsonl
{"audio":"./data/rain.wav","ambient_sound":"Rolling thunder with steady rainfall."}
{"audio":"./data/footsteps.wav","ambient_sound":"Clear footsteps echoing on concrete at a steady rhythm.","tokens":160}
```

`tokens` 字段是可选的目标 audio 时长 token 数（12.5 Hz，例如 160 ≈ 12.8 秒），用于时长控制训练。

### 4.3 MOSS-TTSD — 多人对话需要手工编辑 reference

`moss_ttsd` 任务的真实数据一般不是一条 wav 对应一条 SRT，而是一条 **多说话人对话**音频配上 `[S1] ... [S2] ...` 这样的打标文本。建议：

1. 每个说话人各录一条 10–15 秒纯净参考样本，保存为 `ref_s1.wav`、`ref_s2.wav`
2. 对对话音频**整段**作为 target，手写 JSONL（不走 `segment_audio.py`）：

```jsonl
{
  "audio":"/root/autodl-tmp/datasets/dialog_0001.wav",
  "text":"[S1] 这段是第一个说话人的话。[S2] 这段是第二个说话人的话。[S1] 我们接着聊下一轮。",
  "reference":["/root/autodl-tmp/datasets/ref_s1.wav","/root/autodl-tmp/datasets/ref_s2.wav"],
  "language":"zh"
}
```

依据：`moss_tts_delay/finetuning/README.md §2.2`。`reference` 列表的元素可以是 `null`，表示该说话人没有克隆参考。

### 4.4 MOSS-TTSD — 必须做的两件事

读 `moss_tts_delay/finetuning/README.md §2.2` 最后两段。如果你用 `OpenMOSS-Team/MOSS-TTSD-v1.0` 做下游微调：

**① 替换 4 个 Python 文件**

从 HuggingFace `OpenMOSS-Team/MOSS-TTSD-v1.0` 仓库拉取以下 4 个文件，覆盖到 `/root/MOSS-TTS/moss_tts_delay/` 对应位置：

- `processing_moss_tts.py`
- `modeling_moss_tts.py`
- `configuration_moss_tts.py`
- `inference_utils.py`

命令参考（走 hf-mirror 国内镜像，无需代理、无需 token）：

```bash
conda activate moss-tts
HF_ENDPOINT=https://hf-mirror.com python - <<'PY'
from huggingface_hub import hf_hub_download
for fn in ["processing_moss_tts.py","modeling_moss_tts.py","configuration_moss_tts.py","inference_utils.py"]:
    dst = hf_hub_download(repo_id="OpenMOSS-Team/MOSS-TTSD-v1.0", filename=fn,
                          local_dir="/root/MOSS-TTS/moss_tts_delay")
    print("downloaded:", dst)
PY
```

**官方警告**：如果不替换直接训，`train loss 看起来正常但推理会胡言乱语`。

**② 训练 / 预处理都加 `--n-vq 16`**

MOSS-TTSD 用 16 层 RVQ，不是默认的 32。`prepare_data.py` 和 `sft.py` 都要加：

```bash
python moss_tts_delay/finetuning/prepare_data.py ... --n-vq 16
accelerate launch moss_tts_delay/finetuning/sft.py ... --n-vq 16
```

本指南的 `/root/2-run-train.sh` 已经自动注入这个参数（你选菜单 2 的时候）。

---

## 五、MOSS-TTS-Realtime 数据组织（与上面完全不同）

依据：`moss_tts_realtime/finetuning/README.md §2`。

### 5.1 单轮数据

```jsonl
{"id": "000001", "ref_wav": "./data/ref0.wav", "conversations": [{"role": "assistant", "text": "Actually, I noticed that I am very sensitive to other people's emotions.", "wav": "./data/utt0001.wav"}]}
{"id": "000002", "ref_wav": "./data/ref1.wav", "conversations": [{"role": "assistant", "text": "She said she would be here by noon.", "wav": "./data/utt0002.wav"}]}
```

- `ref_wav`：该说话人的参考音频（可省略）
- `conversations[].role`：`user` 或 `assistant`
- `conversations[].text`：该轮文本
- `conversations[].wav`：该轮音频路径

### 5.2 多轮数据

```jsonl
{"id": "000003", "ref_wav": "./data/ref0.wav", "conversations": [
  {"role": "user",      "text": "Hey, I just landed in Paris. Any ideas?", "wav": "./data/u1.wav"},
  {"role": "assistant", "text": "Nice, welcome to Paris!", "wav": "./data/a1.wav"},
  {"role": "user",      "text": "Just a backpack. Nothing too rushed.",    "wav": "./data/u2.wav"},
  {"role": "assistant", "text": "Got it. Start near the Seine...",         "wav": "./data/a2.wav"}
]}
```

**官方约束**：`assistant` 轮必须是同一个说话人（且与 `ref_wav` 同说话人），`user` 轮可以来自不同说话人。单轮与多轮可以**混合**训练以兼顾两种能力。

### 5.3 自动化生成建议

`segment_audio.py` 不直接支持 Realtime 格式。推荐两种落地方式：

- **自然对话录音**：如果你有真实多轮对话录音，手工切分后按每个 turn 写 `conversations[]` 列表。
- **合成多轮**：先用 `moss_tts` 格式切好所有切片，再写一个简单的合并脚本把 N 条连续切片打包成一条 `conversations` 记录。

---

## 六、预处理：prepare_data.py

这一步把每条训练样本的 `audio` 字段（以及可选 `ref_audio` / `reference` / `ref_wav`）提前编码为 `audio_codes`（12.5 Hz 的 32 层 RVQ 整数序列；MOSS-TTSD 是 16 层），避免训练时反复调 codec。

### 6.1 单进程（最简单）

```bash
conda activate moss-tts
cd /root/MOSS-TTS

# 以 MOSS-TTS 为例
python moss_tts_delay/finetuning/prepare_data.py \
    --model-path /root/autodl-tmp/models/MOSS-TTS \
    --codec-path OpenMOSS-Team/MOSS-Audio-Tokenizer \
    --device auto \
    --input-jsonl /root/autodl-tmp/datasets/train_raw.jsonl \
    --output-jsonl /root/autodl-tmp/datasets/train_with_codes.jsonl
```

| 替换表（按你选的任务） |
|---|
| **MOSS-TTS**：`moss_tts_delay/finetuning/prepare_data.py` + `--model-path /root/autodl-tmp/models/MOSS-TTS` |
| **MOSS-TTSD**：同上 + `--model-path /root/autodl-tmp/models/MOSS-TTSD-v1.0 --n-vq 16` |
| **MOSS-VoiceGenerator**：同上 + `--model-path /root/autodl-tmp/models/MOSS-VoiceGenerator` |
| **MOSS-SoundEffect**：同上 + `--model-path /root/autodl-tmp/models/MOSS-SoundEffect` |
| **MOSS-TTS-Local**：`moss_tts_local/finetuning/prepare_data.py` + `--model-path /root/autodl-tmp/models/MOSS-TTS-Local-Transformer` |
| **MOSS-TTS-Realtime**：`moss_tts_realtime/finetuning/prepare_data.py`（**不传 `--model-path`**，只传 `--codec-path`，见 realtime finetuning README §3.1） |

### 6.2 默认编码策略说明

`prepare_data.py` **默认会**把参考音频一起编码。识别的字段因 pipeline 而异：

| Pipeline | 识别的参考字段 | 依据 |
|---|---|---|
| `moss_tts_delay` / `moss_tts_local` | `ref_audio` / `reference` / `reference_audio` | `moss_tts_delay/finetuning/prepare_data.py:67-70, 124`（`--skip-reference-audio-codes` help 文本 + `collect_reference_paths` 字段枚举） |
| `moss_tts_realtime` | 额外识别 `ref_wav`（以及 `conversations[].wav`） | `moss_tts_realtime/finetuning/prepare_data.py:248-269` |

如果你的参考音频和目标切片大面积重复（重复编码浪费），加 `--skip-reference-audio-codes` 禁用（见 delay 版 finetuning `README.md §3.1` line 119-128）。

### 6.3 单卡 96GB 的编码吞吐

MOSS-Audio-Tokenizer 本体才 ~1B，编码是高度并行的纯前向。单卡 RTX PRO 6000 直接跑，不用 `accelerate launch`。几千条样本通常 5–20 分钟。

### 6.4 验证

```bash
wc -l /root/autodl-tmp/datasets/train_with_codes.jsonl
head -1 /root/autodl-tmp/datasets/train_with_codes.jsonl | python -c "import sys,json; d=json.load(sys.stdin); print(list(d.keys()))"
```

期望能看到 `audio_codes` 键。

---

## 七、启动全量微调：sft.py

依据：`moss_tts_delay/finetuning/README.md §4`、`sft.py:87-142` 的 argparse。

### 7.1 sft.py 核心超参（三套 pipeline 完全一致，除 realtime 没有 `--channelwise-loss-weight`）

来自 `sft.py` 的 argparse 源码（`moss_tts_delay/finetuning/sft.py:87-142`）：

| 参数 | 默认值 | 含义 |
|---|---|---|
| `--model-path` | `OpenMOSS-Team/MOSS-TTS` | 基础模型路径。**必改为本地绝对路径**（如 `/root/autodl-tmp/models/MOSS-TTS`）——否则 HF 会尝试下载原版 8B 大模型 |
| `--codec-path` | `OpenMOSS-Team/MOSS-Audio-Tokenizer` | 分词器路径。**默认值就是对的**（HF 仓库 ID），HF 会从 `/root/hf_cache` 缓存解析；也可传本地路径 |
| `--train-jsonl` | **必填** | 可是单文件、目录、glob、逗号列表 |
| `--output-dir` | `output` | checkpoint 输出根目录 |
| `--per-device-batch-size` | `1` | 单卡 batch |
| `--gradient-accumulation-steps` | `1` | 建议 8 |
| `--learning-rate` | `1e-5` | |
| `--weight-decay` | `0.1` | |
| `--adam-beta1 / --adam-beta2` | `0.9 / 0.95` | |
| `--warmup-ratio` | `0.03` | |
| `--lr-scheduler-type` | `linear` | |
| `--num-epochs` | `3` | |
| `--max-grad-norm` | `1.0` | |
| `--mixed-precision` | `bf16` | |
| `--attn-implementation` | `auto` | |
| `--n-vq` | `None` | **MOSS-TTSD 必填 `--n-vq 16`** |
| `--gradient-checkpointing` | 关 | **8B 单卡必须显式开启** |
| `--channelwise-loss-weight` | `1,32` | 文本头 vs 32 个音频头的损失权重；两值或 n_vq+1 值 |
| `--wandb-project` | `None` | 填就启用 W&B 日志 |
| `--seed` | `42` | |

### 7.2 Checkpoint 保存策略

`sft.py` 按 epoch 保存（见 `moss_tts_delay/finetuning/sft.py:695`）：

```
<output-dir>/
  checkpoint-epoch-0/   ← 每个目录里都包含 config + tokenizer + processor + runtime .py
  checkpoint-epoch-1/   ← 都能直接 AutoModel.from_pretrained(...) 做推理
  checkpoint-epoch-2/
```

### 7.3 单卡 96GB 资源策略

| 模型 | 参数 | 单卡 96GB 全量可行性 | 推荐 accelerate 配置 |
|---|---|---|---|
| MOSS-TTS-Local-Transformer | 1.7B | ✅ 轻松 | `accelerate launch --num_processes 1 --mixed_precision bf16` |
| MOSS-VoiceGenerator | 1.7B | ✅ 轻松 | 同上 |
| MOSS-TTS-Realtime | 1.7B | ✅ 轻松 | 同上 |
| **MOSS-TTS** | 8B | ✅ 可行（bf16 + gradient_checkpointing） | 同上 |
| **MOSS-TTSD-v1.0** | 8B | ✅ 可行（同上 + `--n-vq 16`） | 同上 |
| **MOSS-SoundEffect** | 8B | ✅ 可行（同上） | 同上 |

> **关于 ZeRO-3**：仓库自带的 `configs/accelerate_zero3_8b.yaml` 默认 `num_processes: 8`（多卡场景），在单卡 96GB 上**不需要用**。如果你跑 8B 时显存紧张，优先加 `--gradient-checkpointing`；如果仍然不够，再手工改一份 zero3 yaml（把 `num_processes` 改 1，把 `offload_optimizer_device` / `offload_param_device` 改成 `cpu`），利用 120GB 内存做 CPU offload。

### 7.4 MOSS-TTS（8B）全量微调命令（单卡）

```bash
conda activate moss-tts
cd /root/MOSS-TTS

accelerate launch --num_processes 1 --mixed_precision bf16 \
    moss_tts_delay/finetuning/sft.py \
    --model-path /root/autodl-tmp/models/MOSS-TTS \
    --codec-path OpenMOSS-Team/MOSS-Audio-Tokenizer \
    --train-jsonl /root/autodl-tmp/datasets/train_with_codes.jsonl \
    --output-dir /root/autodl-tmp/checkpoints/moss_tts_sft \
    --per-device-batch-size 1 \
    --gradient-accumulation-steps 8 \
    --learning-rate 1e-5 \
    --warmup-ratio 0.03 \
    --num-epochs 3 \
    --mixed-precision bf16 \
    --channelwise-loss-weight 1,32 \
    --gradient-checkpointing
```

### 7.5 MOSS-TTSD（8B，n_vq=16）全量微调命令

```bash
accelerate launch --num_processes 1 --mixed_precision bf16 \
    moss_tts_delay/finetuning/sft.py \
    --model-path /root/autodl-tmp/models/MOSS-TTSD-v1.0 \
    --codec-path OpenMOSS-Team/MOSS-Audio-Tokenizer \
    --train-jsonl /root/autodl-tmp/datasets/train_with_codes.jsonl \
    --output-dir /root/autodl-tmp/checkpoints/moss_ttsd_sft \
    --per-device-batch-size 1 \
    --gradient-accumulation-steps 8 \
    --learning-rate 1e-5 \
    --warmup-ratio 0.03 \
    --num-epochs 3 \
    --mixed-precision bf16 \
    --channelwise-loss-weight 1,16 \
    --n-vq 16 \
    --gradient-checkpointing
```

### 7.6 MOSS-VoiceGenerator（1.7B）

```bash
accelerate launch --num_processes 1 --mixed_precision bf16 \
    moss_tts_delay/finetuning/sft.py \
    --model-path /root/autodl-tmp/models/MOSS-VoiceGenerator \
    --codec-path OpenMOSS-Team/MOSS-Audio-Tokenizer \
    --train-jsonl /root/autodl-tmp/datasets/train_with_codes.jsonl \
    --output-dir /root/autodl-tmp/checkpoints/moss_voice_generator_sft \
    --per-device-batch-size 1 \
    --gradient-accumulation-steps 8 \
    --learning-rate 1e-5 \
    --warmup-ratio 0.03 \
    --num-epochs 3 \
    --mixed-precision bf16 \
    --channelwise-loss-weight 1,32 \
    --gradient-checkpointing
```

### 7.7 MOSS-SoundEffect（8B）

```bash
accelerate launch --num_processes 1 --mixed_precision bf16 \
    moss_tts_delay/finetuning/sft.py \
    --model-path /root/autodl-tmp/models/MOSS-SoundEffect \
    --codec-path OpenMOSS-Team/MOSS-Audio-Tokenizer \
    --train-jsonl /root/autodl-tmp/datasets/train_with_codes.jsonl \
    --output-dir /root/autodl-tmp/checkpoints/moss_sound_effect_sft \
    --per-device-batch-size 1 \
    --gradient-accumulation-steps 8 \
    --learning-rate 1e-5 \
    --warmup-ratio 0.03 \
    --num-epochs 3 \
    --mixed-precision bf16 \
    --channelwise-loss-weight 1,32 \
    --gradient-checkpointing
```

### 7.8 MOSS-TTS-Local-Transformer（1.7B）

```bash
accelerate launch --num_processes 1 --mixed_precision bf16 \
    moss_tts_local/finetuning/sft.py \
    --model-path /root/autodl-tmp/models/MOSS-TTS-Local-Transformer \
    --codec-path OpenMOSS-Team/MOSS-Audio-Tokenizer \
    --train-jsonl /root/autodl-tmp/datasets/train_with_codes.jsonl \
    --output-dir /root/autodl-tmp/checkpoints/moss_tts_local_sft \
    --per-device-batch-size 1 \
    --gradient-accumulation-steps 8 \
    --learning-rate 1e-5 \
    --warmup-ratio 0.03 \
    --num-epochs 3 \
    --mixed-precision bf16 \
    --channelwise-loss-weight 1,32 \
    --gradient-checkpointing
```

### 7.9 MOSS-TTS-Realtime（1.7B）

注意：`moss_tts_realtime/finetuning/sft.py` 的参数与 delay/local 相同但**没有 `--channelwise-loss-weight` 和 `--gradient-checkpointing`**（realtime finetuning README §4.4 明确列出了 realtime 版支持的参数，没有这两项）。

```bash
accelerate launch --num_processes 1 --mixed_precision bf16 \
    moss_tts_realtime/finetuning/sft.py \
    --model-path /root/autodl-tmp/models/MOSS-TTS-Realtime \
    --codec-path OpenMOSS-Team/MOSS-Audio-Tokenizer \
    --train-jsonl /root/autodl-tmp/datasets/train_with_codes.jsonl \
    --output-dir /root/autodl-tmp/checkpoints/moss_tts_realtime_sft \
    --per-device-batch-size 1 \
    --gradient-accumulation-steps 8 \
    --learning-rate 1e-5 \
    --warmup-ratio 0.03 \
    --num-epochs 3 \
    --mixed-precision bf16
```

---

## 八、一键启动（使用部署时交付的 2-run-train.sh）

部署文档 `2026-4-22-moss-tts-autodl-deployment-guide.md` §11.2 已经给出了 `2-run-train.sh`——它会：

1. 交互式菜单让你选 6 个任务之一
2. 自动选对 `moss_tts_delay` / `moss_tts_local` / `moss_tts_realtime` pipeline
3. 对 MOSS-TTSD 自动注入 `--n-vq 16`
4. 串联 `prepare_data.py`（通过官方的 `run_train.sh` 入口）→ `sft.py`
5. 8B 模型自动开 `gradient_checkpointing`

推荐流程：

```bash
# 1. 切分好数据，得到 /root/autodl-tmp/datasets/train_raw.jsonl
# 2. 用 screen 挂后台启动训练
screen -S train
bash /root/2-run-train.sh
# 选任务 → 确认 → 开始训练
# Ctrl+A 然后 D 把 screen 放后台
# screen -r train 回来看进度
```

---

## 九、训练监控

### 9.1 训练日志

`sft.py` 已经在控制台输出 timestamped prefix、`global_batch_size`、`step_time`、`steps_per_sec`、`samples_per_sec`、`eta`（见各 finetuning README §4.4 末尾）。

### 9.2 W&B（可选）

```bash
pip install wandb
wandb login   # 在浏览器粘贴 API key

# 训练命令追加:
--wandb-project moss-tts-finetune \
--wandb-run-name my-run-v1 \
--wandb-tags zh,cloning
```

### 9.3 显存与 GPU 利用率

另开终端：

```bash
watch -n 2 nvidia-smi
```

单卡 96GB 跑 8B 微调典型占用：`70–90 GB`（开 gradient_checkpointing 后显著下降）。

---

## 十、训练后推理验证

**方法 A：用交付的 `3-run-infer.sh`**（最简单，见部署文档 §11.3）：

```bash
bash /root/3-run-infer.sh
# 选 checkpoint → 输入文本 → 可选 voice cloning → 输出 wav
```

**方法 B：手写 Python**（与根 README 的 quickstart 完全一致，只是把模型路径换成 checkpoint 目录）：

> 这里 `AutoProcessor.from_pretrained(ckpt, ...)` **不触发** HF 网络——因为 `ckpt` 里 `processor_config.json` 的 `audio_tokenizer_name_or_path` 字段在训练时由 `sft.py --codec-path OpenMOSS-Team/MOSS-Audio-Tokenizer` 写入的就是这个 HF ID（见 `moss_tts_delay/finetuning/sft.py:350-353` 的 `copy_inference_assets`），而 HF 缓存里已经有这个 repo（部署文档第 8.1 步下的），所以 HF 直接从 `/root/hf_cache` 解析本地缓存，不联网。

```python
# 示例：加载 MOSS-TTS 微调 checkpoint 做推理
import importlib.util, torch, torchaudio
from transformers import AutoModel, AutoProcessor

torch.backends.cuda.enable_cudnn_sdp(False)
torch.backends.cuda.enable_flash_sdp(True)
torch.backends.cuda.enable_mem_efficient_sdp(True)
torch.backends.cuda.enable_math_sdp(True)

ckpt = "/root/autodl-tmp/checkpoints/moss_tts_sft/checkpoint-epoch-2"
device, dtype = "cuda", torch.bfloat16
attn = "flash_attention_2" if importlib.util.find_spec("flash_attn") else "sdpa"

proc = AutoProcessor.from_pretrained(ckpt, trust_remote_code=True)
proc.audio_tokenizer = proc.audio_tokenizer.to(device)
model = AutoModel.from_pretrained(ckpt, trust_remote_code=True,
                                   attn_implementation=attn, torch_dtype=dtype).to(device).eval()

conversation = [[proc.build_user_message(text="微调后的声音听起来怎么样？")]]
batch = proc(conversation, mode="generation")
out = model.generate(input_ids=batch["input_ids"].to(device),
                     attention_mask=batch["attention_mask"].to(device),
                     max_new_tokens=4096)
audio = proc.decode(out)[0].audio_codes_list[0]
torchaudio.save("/root/autodl-tmp/outputs/finetuned_sample.wav",
                audio.unsqueeze(0), proc.model_config.sampling_rate)
print("OK")
```

**方法 C：启动 Gradio WebUI 对比原版 vs 微调版**（手动启）：

```bash
# 改 1-run-webui.sh 里菜单 1 的 MODEL_PATH 为 /root/autodl-tmp/checkpoints/xxx/checkpoint-epoch-2
# 或直接在命令行覆盖:
cd /root/MOSS-TTS
python clis/moss_tts_app.py \
    --model_path /root/autodl-tmp/checkpoints/moss_tts_sft/checkpoint-epoch-2 \
    --port 6006 --host 0.0.0.0
```

---

## 十一、常见问题速查

| 症状 | 根因 / 对策 |
|---|---|
| **推理胡言乱语，但训练 loss 正常** | 极高概率是 MOSS-TTSD 漏了替换 4 个 `.py` 文件或漏了 `--n-vq 16`。回到本文 §4.4 重做。 |
| **OOM 爆显存（8B 模型）** | 先确认 `--gradient-checkpointing` 已打开。再确认 `attn-implementation` 不是 `eager`。还不行，改用 ZeRO-3 yaml（本文 §7.3 有提示）。 |
| **FlashAttention 相关报错，尤其是 MOSS-TTS-Realtime** | Realtime 的 local transformer 用 StaticCache，与 FA2 不兼容（见 `moss_tts_realtime/infer.py` 的官方注释）。训练和推理都用 `--attn-implementation sdpa`。 |
| **`cuDNN SDPA backend` 报错** | 必须在入口脚本顶部加四行官方防御代码（见根 README 237-241）。`sft.py` 内部已经处理，你只在**自己写推理代码**时要手工加。 |
| **预处理时 reference audio 编码很慢** | `prepare_data.py` 默认会编码 `ref_audio`/`reference`。如果它与 target audio 大量重复，加 `--skip-reference-audio-codes`（见 finetuning README §3.1）。 |
| **训练中途重启想从最后一个 checkpoint 继续** | 官方 `sft.py` 不自带 `--resume-from-checkpoint`。目前的做法是把 `--model-path` 指向上一次最后的 `checkpoint-epoch-N`（它是完整的可加载 HF 模型），重新训剩余 epoch。 |
| **想同时训练多个任务** | 分开训。每个任务产出独立的 `checkpoint-epoch-*`，不要在同一个 `--output-dir` 混用。 |
| **DataLoader 很慢** | `--num-workers 4`（默认 0），CPU 25 核完全够用。 |
| **Silero VAD 第一次下不下来** | `torch.hub` 要访问 GitHub。运行 `segment_audio.py` 前先 `source /etc/network_turbo`。 |
| **切分时 SRT 里时间戳和音频不对齐** | VoxCPM 教程 §2 的 AI 清洗 Prompt 第 1 条"时间轴精度（最高优先级）"严禁改原始时间戳——如果 SRT 被 AI 错误地做了四舍五入，这里就会跑偏。重做一遍 SRT 清洗。 |

---

## 十二、关键事实依据索引

每一条设计决策都能回溯到仓库文件：

| 设计决策 | 依据位置 |
|---|---|
| torch 2.9.1+cu128 / torchaudio 2.9.1+cu128 / torchcodec 0.8.1 / transformers 5.0.0 钉版 | `pyproject.toml:44-49`（`torch-runtime` extras） |
| Python `>=3.10`（仓库强制） / Python 3.12（部署指南选择，非仓库强制） | `pyproject.toml:10`（`requires-python = ">=3.10"`） |
| `accelerate launch` + `sft.py` 分离 pipeline 三件套 | `moss_tts_delay/finetuning/README.md §1` |
| JSONL 每个任务的字段（MOSS-TTS / TTSD / VG / SE） | `moss_tts_delay/finetuning/README.md §2.1-2.4` |
| MOSS-TTSD 用 `n_vq=16` 且需替换 4 个 `.py` | `moss_tts_delay/finetuning/README.md §2.2` 第 75-86 行 |
| MOSS-TTS-Realtime 的 `conversations` JSONL 结构 | `moss_tts_realtime/finetuning/README.md §2` |
| Realtime 的 `prepare_data.py` 无 `--model-path` | `moss_tts_realtime/finetuning/README.md §3.1` + `run_train.sh:64-69` |
| `sft.py` 的所有命令行参数 | `moss_tts_delay/finetuning/sft.py:87-142` |
| `--channelwise-loss-weight 1,32` 默认 & 两种写法 | `sft.py:130-140` + `README.md §4.4` |
| checkpoint 按 epoch 保存 | `sft.py:695` |
| `run_train.sh` 的环境变量入口 | `moss_tts_delay/finetuning/run_train.sh` + `README.md §6` |
| MOSS-Audio-Tokenizer 24 kHz / 12.5 Hz / 32 层 RVQ | 根 `README.md` "MOSS-Audio-Tokenizer" 章节 |
| cuDNN SDPA 必须禁用 | 根 `README.md:237-241` |
| Realtime StaticCache 与 FA2 冲突 | `moss_tts_realtime/infer.py:41` 注释 |
| 自带的 3 个 accelerate yaml（ddp_8gpu / fsdp / zero3） | `*/finetuning/configs/*.yaml` |

---

## 十三、全流程总结

```
AutoDL 开机
    ↓
conda activate moss-tts   ←  部署文档已经装好
    ↓
一、Capcut 清理原始录音 → 纯净 WAV
    ↓
二、Parakeet ASR → 原始 SRT
    ↓
三、AI 清洗 SRT（复用 VoxCPM Prompt） → 01.srt / 02.srt / ...
    ↓
四、改 segment_audio.py 顶部 TARGET_TASK / DEFAULT_LANGUAGE / REF_AUDIO_PATH
    对每个 0X.wav 依次跑 → /root/autodl-tmp/datasets/*.wav + train_raw.jsonl
    ↓
五、(MOSS-TTSD 专属) 替换 4 个 .py 文件 + 记得 --n-vq 16
    (VoiceGenerator 专属) 手工补 instruction 字段
    (TTSD / Realtime) 手工整合多说话人 / 多轮对话
    ↓
六、prepare_data.py 预编码 audio_codes → train_with_codes.jsonl
    ↓
七、bash /root/2-run-train.sh 选模型 → screen -S train 挂后台
    ↓
    等训练完（每 epoch 一个 checkpoint-epoch-N）
    ↓
八、bash /root/3-run-infer.sh 选 checkpoint → 听效果
    ↓
    ✅ 如果效果不满意：调 learning-rate / num-epochs / channelwise-loss-weight 重训
    ✅ 如果效果 OK：直接把 checkpoint-epoch-N 当成新的基础模型用
```

---

## 附录：与 VoxCPM 教程的对照速查

| 项 | VoxCPM | MOSS-TTS |
|---|---|---|
| conda 环境 | `voxcpm` Python 3.11 | `moss-tts` Python 3.12 |
| PyTorch | 2.8.0+cu128 | 2.9.1+cu128 |
| transformers | （未强制） | **5.0.0** 钉版 |
| 模型数量 | 2（v1.5 / v2） | **6**（TTS / TTSD / VG / SE / Local / Realtime），共享一个 Audio Tokenizer |
| 训练入口 | `scripts/train_voxcpm_finetune.py --config_path xxx.yaml` | `accelerate launch <arch>/finetuning/sft.py --<cli-args>` |
| 训练配置方式 | yaml 文件 | accelerate yaml（结构/并行） + sft.py 命令行参数（超参） |
| 数据清单格式 | `{"audio","text","duration"}` | 按任务不同（见本文 §3.2） |
| 目标采样率 | yaml `sample_rate` 字段（16k/44.1k） | 固定 24 kHz（MOSS-Audio-Tokenizer 内部） |
| 切分脚本 | `segment_audio.py`（输出 VoxCPM 格式） | `segment_audio.py` **本文版**（输出 MOSS-TTS 格式） |
| 尾静音硬限制 | 500 ms（VoxCPM 官方警告） | 官方未明确硬限制，本文建议延用 250 ms |
| SRT 清洗 Prompt | **完全复用** | **完全复用** |
| Capcut / Parakeet 工作流 | **完全复用** | **完全复用** |
| LoRA 微调支持 | yaml 里 `lora:` 块 | 本仓库**不提供** LoRA 训练脚本（`community/norwegian-lora` 是第三方贡献，独立体系） |

至此 MOSS-TTS 家族的全量微调流程与 VoxCPM 对齐完毕。
