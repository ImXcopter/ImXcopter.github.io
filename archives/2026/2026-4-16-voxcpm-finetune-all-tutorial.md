# VoxCPM 全量微调完整指南（VoxCPM1.5 + VoxCPM2 通用）

本文档提供 VoxCPM 模型全量微调的完整流程，包括数据准备、处理、环境部署和训练步骤。
适用于 VoxCPM1.5 和 VoxCPM2 两个版本。

---

## 一、数据准备

### 1.1 音频素材要求

确保原始音频满足以下质量标准:

- **纯净度要求**：无背景音乐、无环境杂音
- **推荐来源**：YouTube 视频/音频等高质量资源
- **格式建议**：WAV 格式（无损音质）
- **采样率**：不限（脚本会自动处理响度归一化，训练时 dataloader 自动重采样到目标值）

### 1.2 音频预处理

使用 **Capcut** 进行音频清理:

1. 剪除片头片尾含音乐的部分
2. 去除中间的杂音片段
3. 导出为 WAV 格式

### 1.3 语音转写

使用 **Parakeet** 工具对 WAV 音频进行自动语音识别（ASR），生成 SRT 字幕文件。

---

## 二、SRT 字幕处理

### 2.1 处理平台

访问 [Google AI Studio](https://aistudio.google.com) 或使用 Claude 等 AI 辅助处理。

### 2.2 处理 SRT 文件

将以下 Prompt 提供给 AI 模型，并附上原始 SRT 文件:

```
【角色设定】
你是一位专业的 TTS(语音合成)数据集处理专家，擅长进行高精度的全量微调的数据对齐与清洗。

【任务目标】
我将提供一份由 ASR(自动语音识别)生成的原始 SRT 字幕文件。请你将其处理为高质量的全量微调训练数据清单。

【核心执行标准（必须严格遵守）】

1. 时间轴精度(最高优先级)
   - 严禁修改原始时间戳的精度！严禁四舍五入！
   - 原始字幕段落数据是多少毫秒（例如 00:27:20,960），输出必须保持一致，不得改为 00:27:20,000
   - 字幕段落合并时，新片段的“开始时间”取第一句的开始，“结束时间”取最后一句的结束。

2. 时长控制
   - 目标字幕段落时长：4-28 秒为最佳时长。最短不能短于3秒，最长不超过 30 秒，这是极限！
   - 可以通过合并短字幕段落（短句）来达到目标字幕段落时长的要求。

3. 语义完整性（句子合并核心逻辑）
   - 语义优先级 > 时长优先级
   - 必须确保句子的完整性，严禁在半句话（如逗号后、从句中）截断。
   - 可以将连续的 ASR 短字幕段落（短句）合并为完整的意群（完整的陈述句、复句），形成一条长段落（长句）。
   - 如果字幕段落（短句）的时间长度小于时长控制中要求的最短时长，且无法与后面的字幕段落（句子）合并后达到段落（句子）意思的完整性要求，可以谨慎考虑丢弃这个字幕段落（句子）！
   - 如果字幕段落（长句）的时间长度大于时长控制中要求的最长极限时长，可以谨慎考虑丢弃！

4. 文本清洗与精修
   - 标点：修改或添加正确的标点符号（逗号、句号、问号、引号等），这决定了训练好后模型生成的语气和韵律。
   - 拼写：修正明显的 ASR 识别错误（单词拼写错误；单词发音相同，但是是错误的单词，放入句子中明显与句意不符的）
   - 格式：修正英文的大小写规范（例如句首大写、专有名词大写）
   - 重点：如果单词识别错误的（并非单词发音相同，单词错误），可以丢弃删除这一整段字幕段落！

5. 修复后的srt文件，要进行复查。避免时间戳，单词，字幕段落（句子）合并等错误的发生！

【输出格式】
请直接输出处理后的 SRT 内容,格式如下：

1
00:00:00,000 --> 00:00:12,240
[这里是修正后的、完整的、带有标点符号的文本]

2
00:00:13,120 --> 00:00:26,160
[下一段文本...]

【输出文件名】
请将输出的文件名直接保存为：01.srt，02.srt......去除掉transcription_前缀！
```

或者将下面的内容保存成prompt.txt（例如给Claude Code读取使用）

```
【角色设定】
你是一位专业的 TTS（语音合成）数据集处理专家，擅长进行高精度的全量微调数据对齐与清洗。
本清洗任务的最终产物是 (音频片段, 文本) 对：每一个输出段的文本都必须与它时间戳覆盖的那段音频的实际发音完全一致——这是 TTS 训练的硬约束。

【任务目标】
我将提供一份由 ASR（自动语音识别）生成的原始 SRT 字幕文件。请你将其处理为高质量的全量微调训练数据清单。


【核心执行标准（按优先级从高到低）】

1. 时间轴精度（最高优先级）

   1.1 严禁修改原始时间戳的精度！严禁四舍五入！原数据 00:27:20,960，输出必须保持一致，不得写成 00:27:20,000。

   1.2 字幕段落合并时，新片段的起始时间 = 第一个原始段的起始，结束时间 = 最后一个原始段的结束。

   1.3 输出段的起止时间戳必须各自出现在原始 SRT 的某个时间点上。不能凭空"延长"或"缩短"一个段落，只能把若干完整的原始段首尾相接地合并。

   1.4 允许段落之间存在时间间隙（两段之间音频中没有讲话内容，或者某段已被整段丢弃）。TTS 训练每条样本独立，段间不必连续。


2. 时长控制

   2.1 目标字幕段落时长：4–28 秒为最佳，3–30 秒为极限。

   2.2 合并短段时，优先在语义断点处分段：
       - 强断点（优选）：句号、问号、感叹号
       - 弱断点（可用）：分号、冒号、长语气停顿后的逗号
       - 绝不断点：从句中间、介词短语中间、固定搭配内部
       - 结构断点：如 "Tip number one."、"Rule number six." 这类标题句若单独过短，优先与紧接其后的说明句合并，形成"标题 + 内容"一体。

   2.3 当某原始段时长极短（小于最短时长要求）时：
       - 能与前或后段合并并保持语义完整 → 合并
       - 无法合并（语义独立、合并后超极限、或前后段已接近最大时长要求），按第 6 条整段删除规则处理

   2.4 当某原始段时长极长（大于最大时长要求）时：
       - 若内部有强断点可拆分 → 按断点拆成两个或以上输出段
       - 完全没有断点、拆会破坏语义 → 按第 6 条整段删除规则处理

3. 语义完整性

   3.1 语义优先级 > 时长优先级。

   3.2 严禁在半句话截断（逗号后、从句中、限定语中）。

   3.3 把破碎的 ASR 短段合并为完整的意群（完整陈述句、完整复句、完整问答对），形成一条语义自洽的长段。


4. 文本清洗与精修

   4.1 标点
       - 句末必须有 . ? !
       - 对话引述用逗号分隔（He says, Good morning.）
       - 并列列表用逗号（apples, bananas, oranges）
       - 省略句（话没说完）用三点省略号（I like to...）

   4.2 大小写（按英语一般标准）
       - 句子首字母大写
       - 第一人称 I 始终大写
       - 专有名词首字母大写（人名、地名、品牌、语言、星期、月份、节目名、书名、虚构名等）
       - 直接引语首字母大写（He says, Good morning. / She asks, Who was the band? / I say, Thanks.）
       - 缩写/首字母缩略词全大写（TV、UK、USA）

   4.3 数字与时间格式
       - 时间用冒号分隔（10:30、1:25），不用句点（10.30）
       - 百分比保留 % 符号（30%、50%）
       - 小数保留点（1.5 billion）

   4.4 拼写
       - 修正 ASR 同音/近音识别错误（前提：音频读的是正确词、ASR识别错了。详见第 5 条判断流程）
       - 修正明显的 ASR 识别错误（单词拼写错误、连读误加 s 等）
       - 重点：不要"美化"文本——音频读什么，文本就写什么

   4.5 结构修复
       - 两个完整短句被 ASR 合并成一行时，补上断句标点（例："I hear birds they sing" → "I hear birds. They sing"）
       - ASR 丢失句首冠词时补回（仅在音频中确实读了该冠词的情况下）

5. 文本-音频一致性（TTS 训练数据硬约束）

   本条是 TTS 训练任务特有的核心约束，必须严格执行。

   5.1 每一条输出 SRT 的文本必须与它覆盖的音频片段的实际发音完全匹配。模型学的是"听到这段音频时，该如何生成/解读这串文字"。任何文本-音频不匹配都会污染训练信号。

   5.2 当你看到 ASR 识别的词"看起来像错"时，要先判断：
       A. ASR 错了，但音频读的是正确词
          → 修正文本（按第 4.4 条执行）
       B. ASR 没错，音频里发音者本身就读错/读偏了
          → 修正文本会让文本与音频脱节，此时必须整段删除，不能修正（按第 6 条执行）

   5.3 当无法仅凭文本确认是 A 还是 B 时，标注为"需人工裁决"，由人工听原音频确认后再处理。


6. 整段删除的场景与判断

   6.1 必须删除（无条件）：
       - 音频发音与正确文本不符（见 5.2-B）
       - 错位重复段（ASR 把某句内容错放到另一时间点）
       - 时长超过最大时长要求且无法拆分
       - 整段音频内容是噪音/静音/非讲话

   6.2 建议删除：
       - 时长不足最小时长要求且无法与相邻段合并成完整意群

   6.3 不要删除：
       - 单独的感叹、应答（"Yum."、"Good job."）若能与相邻段合并的

   6.4 删除流程：
       - 整段从输出 SRT 中剔除，时间戳随之消失
       - 剔除后，剩余段落从 1 开始重新连续编号

7. 文本-时间戳错位检查

   合并多段时容易把某个原始段的文字写进了相邻输出段的文本里，使输出段时间戳覆盖范围与其文字实际对应的原始段不一致。这种错位不增删词，但会造成严重的音频-文本错位，必须杜绝。

   7.1 合并规则：每个输出段的文字 = 它时间戳覆盖的所有原始段的文字按顺序拼接（再按第 4 条做标点/大小写/格式规范化）。

   7.2 修正原则：文字跟随时间戳。若发现错位，把文字挪回它对应的输出段，时间戳不变。

8. 复查清单

   处理完成后，必须逐项复查：

   8.1 时间戳
       - 每段时长 ∈ [最小要求时长, 最大要求时长]
       - 时间戳单调递增（本段 start ≥ 上段 end）
       - 输出段起止戳都在原文件某段的起/止戳中
       - 编号从 1 连续递增

   8.2 文本-时间戳对齐
       - 每个输出段的文字与其覆盖原始段的拼接文字内容一致（允许的差异仅限第 4 条文本精修和第 5 条音频一致性修正）

   8.3 标点与大小写
       - 句末标点齐全
       - 直接引语首字母大写
       - 专有名词首字母大写
       - 时间数字格式统一（冒号）

   8.4 ASR 错误处理
       - 第 5.4 条列出的高发同音/近音词已逐一核查
       - 所有修正都区分清楚 5.2-A（修文本）还是 5.2-B（删整段）

   8.5 被删除段的复查
       - 每条删除都有明确理由
       - 不该删的没被误删


【输出格式】


请直接输出处理后的 SRT 内容，格式如下：

1
00:00:00,000 --> 00:00:12,240
[修正后的、完整的、带有标点符号的文本]

2
00:00:13,120 --> 00:00:26,160
[下一段文本...]


【输出文件名】

请将输出的文件名直接保存为：01.srt、02.srt……去除掉 transcription_ 前缀。

```
---

## 三、音频分割

### 3.1 分割工具说明

使用 `segment_audio.py` 脚本根据处理后的 SRT 文件分割音频。该脚本基于 Silero VAD 实现精准切分，主要功能包括：

- **EBU R128 响度归一化**：统一所有音频的响度（可选 loudnorm 或 peak 方法）
- **Silero VAD 精准语音检测**：在 SRT 时间轴的基础上进一步精准定位语音边界
- **尾静音精确裁剪**：防止 VoxCPM 训练时"生成停不下来"的问题（官方硬限制尾静音 ≤ 500ms）
- **前后静音可控填充**：裁剪后添加精确的前后静音
- **纯静音片段自动跳过**：能量低于阈值的片段被自动排除
- **追加模式**：多个音频文件可以分批处理，序号和 JSONL 自动续接
- **采样率通用**：不强制重采样，训练时由 HuggingFace datasets Audio 列自动重采样到 yaml 目标（VoxCPM1.5=44100Hz, VoxCPM2=16000Hz）

### 3.2 配置说明

脚本顶部的配置常量：

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `VOXCPM_FT_PROJECT_PATH` | (需修改) | 虚拟环境项目路径 |
| `AUDIO_FILENAME` | (需修改) | 当前要处理的音频文件名 |
| `SRT_FILENAME` | (需修改) | 对应的 SRT 字幕文件名 |
| `ENABLE_NORMALIZATION` | `True` | 是否启用响度归一化 |
| `NORMALIZATION_METHOD` | `"loudnorm"` | 归一化方法（loudnorm / peak） |
| `TARGET_LUFS` | `-23.0` | EBU R128 目标响度 |
| `TARGET_TP_DB` | `-1.5` | 真实峰值上限 |
| `BUFFER_MS` | `100` | SRT 时间定位前后扩展区域（ms） |
| `VAD_THRESHOLD` | `0.5` | Silero VAD 检测阈值（0-1） |
| `FRONT_SILENCE_MS` | `250` | 切片前端静音填充（ms） |
| `END_SILENCE_MS` | `250` | 切片尾端静音填充（ms，必须 ≤ 500） |
| `MIN_VOICE_ENERGY_DBFS` | `-50.0` | 纯静音判定阈值（dBFS） |
| `KEEP_PREPROCESSED_FILE` | `False` | 是否保留中间母带文件 |

### 3.3 使用方法

1. 将 WAV 音频文件和对应的 SRT 文件放在脚本同目录下
2. 修改脚本顶部的 `AUDIO_FILENAME` 和 `SRT_FILENAME`
3. 运行脚本

处理多个音频文件时，依次修改文件名并重新运行——脚本支持追加模式，会自动续接序号和 JSONL。

输出：
- 音频切片保存在 `datasheet/` 目录下，文件名格式 `0001.wav`、`0002.wav`...
- 训练清单保存为 `datasheet/train_data.jsonl`，每行格式：
  ```json
  {"audio": "path/to/0001.wav", "text": "...", "duration": 12.76}
  ```

### 3.4 完整代码

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
VoxCPM 训练数据音频切分脚本
基于 SRT + Silero VAD 精准切分
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

# 虚拟环境项目路径（用于环境检查和模型下载）
VOXCPM_FT_PROJECT_PATH = r"D:\Project\TTS_ASR_Tools\VoxCPM_FT"

# 要处理的音频和 SRT 文件（脚本所在目录下）
AUDIO_FILENAME = "01.wav"            # 音频文件名
SRT_FILENAME = "01.srt"              # SRT 字幕文件名

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
END_SILENCE_MS = 250                  # 切片尾端添加的静音时长（单位：毫秒；VoxCPM 官方硬限制：必须 ≤ 500）
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
    - 不需要强制重采样，因为训练时 HuggingFace datasets Audio 列会自动重采样到
      yaml 里 sample_rate 字段指定的目标（VoxCPM1.5=44100, VoxCPM2=16000）
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

def check_prerequisites() -> bool:
    """检查前置条件：文件存在性与配置项合法性"""
    script_dir = get_script_dir()

    # 检查音频文件
    audio_path = script_dir / AUDIO_FILENAME
    if not audio_path.exists():
        print(f"错误：音频文件不存在")
        print(f"路径：{audio_path}")
        print(f"请检查配置常量 AUDIO_FILENAME 是否正确")
        return False

    # 检查 SRT 文件
    srt_path = script_dir / SRT_FILENAME
    if not srt_path.exists():
        print(f"错误：SRT 文件不存在")
        print(f"路径：{srt_path}")
        print(f"请检查配置常量 SRT_FILENAME 是否正确")
        return False

    # 配置项合法性：尾静音硬上限（VoxCPM 官方警告 > 0.5s 会导致生成停不下来）
    if END_SILENCE_MS > 500:
        print(f"错误：END_SILENCE_MS={END_SILENCE_MS} ms 超过 500 ms 硬上限")
        print(f"VoxCPM 官方警告：尾静音过长（>0.5s）会导致生成停不下来")
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

        # 7. 保存音频文件（采样率继承母带，训练时由 dataloader 自动重采样）
        output_filename = f"{current_index:04d}.wav"
        output_path = output_dir / output_filename
        sf.write(output_path, audio_final, sr)

        # 8. 使用 ffprobe 获取精确 duration（仅用于写入 JSONL，不做越界过滤）
        duration = get_duration_with_ffprobe(output_path)

        # 9. 构造训练数据
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
    6. 保存并生成 JSONL 条目

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
    voxcpm_ft_path = Path(VOXCPM_FT_PROJECT_PATH)
    if not voxcpm_ft_path.exists():
        print(f"错误：VoxCPM_FT 项目目录不存在")
        print(f"路径：{voxcpm_ft_path}")
        print(f"请检查配置常量 VOXCPM_FT_PROJECT_PATH 是否正确")
        print(f"程序终止！")
        sys.exit(1)

    # 检查虚拟环境 Python 是否存在
    venv_python = voxcpm_ft_path / "runtime" / "Scripts" / "python.exe"
    if not venv_python.exists():
        print(f"错误：VoxCPM_FT Python 虚拟环境不存在")
        print(f"路径：{venv_python}")
        print(f"请确认虚拟环境已正确创建")
        print(f"程序终止！")
        sys.exit(1)

    # 检查当前 Python 环境
    current_python = Path(sys.executable).resolve()

    # 如果不在虚拟环境中，自动启动虚拟环境
    if current_python != venv_python:
        print(f"检测到未在虚拟环境中运行")
        print(f"启动 VoxCPM_FT Python 虚拟环境...")
        print()

        # 设置环境变量
        env = os.environ.copy()

        # 设置 Silero VAD 模型下载目录
        models_dir = voxcpm_ft_path / "models"
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
    print("VoxCPM_FT 虚拟环境启动完毕 ✓")
    print()

    # 执行实际的音频切分流程
    execute_segmentation()

def execute_segmentation() -> None:
    """执行音频切分的实际逻辑（在虚拟环境中运行）"""
    # 获取脚本所在目录
    script_dir = get_script_dir()

    print(f"📁 工作目录: {script_dir}")
    print(f"📁 虚拟环境: {VOXCPM_FT_PROJECT_PATH}")
    print()

    # 检查前置条件
    print("检查前置条件...")
    if not check_prerequisites():
        sys.exit(1)
    print("前置条件检查通过 ✓")
    print()

    print(f"⚙️  配置参数:")
    print(f"   - 音频文件: {AUDIO_FILENAME}")
    print(f"   - SRT 文件: {SRT_FILENAME}")
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
    audio_path = script_dir / AUDIO_FILENAME
    srt_path = script_dir / SRT_FILENAME
    data_dir = script_dir / "datasheet"
    jsonl_path = data_dir / "train_data.jsonl"

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
    voxcpm_ft_path = Path(VOXCPM_FT_PROJECT_PATH)
    models_dir = voxcpm_ft_path / "models"
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

    # 保存/追加 JSONL 训练清单
    print("-" * 60)
    print("保存训练数据清单...")

    os.makedirs(data_dir, exist_ok=True)

    # 追加模式：打开文件并追加
    mode = 'a' if start_index > 0 else 'w'
    with open(jsonl_path, mode, encoding='utf-8') as f:
        for item in training_data:
            f.write(json.dumps(item, ensure_ascii=False) + '\n')

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
    print("🎵 VoxCPM 训练数据准备 - 基于 SRT + Silero VAD 精准切分")
    print("=" * 60)
    print()

    # 运行音频切分（自动处理虚拟环境）
    run_segment_audio()

if __name__ == '__main__':
    main()
```

---

## 四、AutoDL 算力云 GPU 环境部署

详细的 AutoDL 环境部署步骤请参考 **[VoxCPM AutoDL 全新环境部署完整指南](https://xcopter.cc/archives/2026/2026-4-16-voxcpm-autodl-deployment-guide.html)**，此处仅列出关键要点：

### 4.1 环境概览

| 项 | 说明 |
|---|---|
| 平台 | AutoDL 算力云 GPU |
| 操作系统 | Ubuntu 22.04 |
| Python | 3.11（独立 conda 环境 `voxcpm`，不影响 base 环境的 Jupyter） |
| PyTorch | 2.8.0+cu128 |
| VoxCPM 源码 | `/root/VoxCPM/`（editable 安装，支持 v1.5 和 v2） |
| VoxCPM2 模型 | `/root/models/VoxCPM2/`（~10 GB） |
| VoxCPM1.5 模型 | `/root/models/VoxCPM1.5/`（~3 GB，可选） |
| 训练数据 | `/root/autodl-tmp/datasheet/` |
| Checkpoint 输出 | `/root/autodl-tmp/checkpoints/` |

### 4.2 交付脚本

部署完成后，`/root/` 下有三个交互式 sh 脚本：

| 脚本 | 功能 |
|---|---|
| `1-run-webui.sh` | 启动推理 WebUI（可选 VoxCPM2 / VoxCPM1.5） |
| `2-run-train.sh` | 启动微调训练（自动扫描 datasheet 下的 yaml 配置） |
| `3-run-infer.sh` | 微调后推理测试（自动识别 LoRA / 全量、v1.5 / v2） |

所有脚本会自动激活 voxcpm conda 环境，不需要手动切换。

---

## 五、训练数据上传

### 5.1 上传切片数据

使用 WinSCP 将本地 `datasheet/` 目录下的所有文件（WAV 切片 + `train_data.jsonl`）上传到 AutoDL：

```
/root/autodl-tmp/datasheet/
```

### 5.2 重写 JSONL 路径

本地生成的 `train_data.jsonl` 里的音频路径是 Windows 格式，需要重写为 Linux 路径：

```python
import json
in_path = "/root/autodl-tmp/datasheet/train_data.jsonl"
out_path = "/root/autodl-tmp/datasheet/train_data_linux.jsonl"
new_dir = "/root/autodl-tmp/datasheet"

with open(in_path, "r", encoding="utf-8") as f_in, open(out_path, "w", encoding="utf-8") as f_out:
    for line in f_in:
        item = json.loads(line)
        old = item["audio"].replace("\\", "/")
        fname = old.split("/")[-1]
        item["audio"] = f"{new_dir}/{fname}"
        f_out.write(json.dumps(item, ensure_ascii=False) + "\n")
print("Done. 重命名 train_data_linux.jsonl 为 train_data.jsonl 即可。")
```

### 5.3 拆分验证集

训练时需要一个独立的验证集来监控过拟合——验证集的数据**不能出现在训练集里**（否则无法检测过拟合）。VoxCPM1.5 和 VoxCPM2 都适用，使用的是同一套训练脚本。

使用 `split_train_val.py` 脚本从原始 JSONL 中随机拆分：

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
训练集 / 验证集拆分脚本

将原始的 train.jsonl 随机拆分为：
  - train_data.jsonl（训练集，90%）
  - val_data.jsonl（验证集，10%）

两个文件内容完全不重复，验证集用于训练时监控过拟合。
原始文件不会被修改。
"""

import json
import random
from pathlib import Path

# ============================================================================
# 配置（用户可修改）
# ============================================================================

INPUT_FILE = "train.jsonl"            # 原始 JSONL 文件名（脚本同目录下）
TRAIN_OUTPUT = "train_data.jsonl"     # 输出的训练集文件名
VAL_OUTPUT = "val_data.jsonl"         # 输出的验证集文件名
VAL_RATIO = 0.10                      # 验证集比例（0.10 = 10%）
RANDOM_SEED = 42                      # 随机种子（固定种子保证每次拆分结果一致）

# ============================================================================
# 主逻辑
# ============================================================================

def main():
    script_dir = Path(__file__).parent.resolve()
    input_path = script_dir / INPUT_FILE
    train_path = script_dir / TRAIN_OUTPUT
    val_path = script_dir / VAL_OUTPUT

    # 检查输入文件
    if not input_path.exists():
        print(f"❌ 输入文件不存在: {input_path}")
        print(f"请将原始 JSONL 文件放在脚本同目录下，文件名为 {INPUT_FILE}")
        return

    # 读取所有行
    with open(input_path, "r", encoding="utf-8") as f:
        lines = [line.strip() for line in f if line.strip()]

    total = len(lines)
    if total == 0:
        print("❌ 输入文件为空")
        return

    # 验证每行是合法 JSON
    for i, line in enumerate(lines):
        try:
            json.loads(line)
        except json.JSONDecodeError as e:
            print(f"❌ 第 {i + 1} 行 JSON 格式错误: {e}")
            return

    # 随机打乱
    random.seed(RANDOM_SEED)
    random.shuffle(lines)

    # 按比例拆分
    val_count = max(1, int(total * VAL_RATIO))
    train_count = total - val_count

    val_lines = lines[:val_count]
    train_lines = lines[val_count:]

    # 写出训练集
    with open(train_path, "w", encoding="utf-8") as f:
        for line in train_lines:
            f.write(line + "\n")

    # 写出验证集
    with open(val_path, "w", encoding="utf-8") as f:
        for line in val_lines:
            f.write(line + "\n")

    # 打印结果
    print("=" * 50)
    print("✅ 拆分完成")
    print("=" * 50)
    print(f"  原始文件:   {input_path.name} ({total} 条)")
    print(f"  训练集:     {train_path.name} ({train_count} 条)")
    print(f"  验证集:     {val_path.name} ({val_count} 条)")
    print(f"  验证集比例: {val_count / total * 100:.1f}%")
    print(f"  随机种子:   {RANDOM_SEED}")
    print()
    print(f"两个文件内容完全不重复，可直接用于训练。")
    print(f"原始文件 {INPUT_FILE} 未被修改。")

if __name__ == "__main__":
    main()
```

将脚本和原始 `train.jsonl` 放在同一目录下，运行 `python split_train_val.py` 即可生成 `train_data.jsonl` 和 `val_data.jsonl`。

### 5.4 准备 yaml 配置文件

在 `/root/autodl-tmp/datasheet/` 下放置一个 yaml 配置文件，文件名格式：
- `voxcpm_finetune_all.yaml` — VoxCPM 全量微调
- `voxcpm_finetune_lora.yaml` — VoxCPM LoRA 微调

yaml 里需要填写的关键路径：
- `pretrained_path`: 基础模型路径（如 `/root/models/VoxCPM2`）
- `train_manifest`: 训练数据清单（如 `/root/autodl-tmp/datasheet/train_data.jsonl`）
- `val_manifest`: 验证集清单（如 `/root/autodl-tmp/datasheet/val_data.jsonl`，填 `null` 表示不使用）
- `save_path`: checkpoint 输出目录（如 `/root/autodl-tmp/checkpoints/xxx`）
- `tensorboard`: TensorBoard 日志目录（如 `/root/autodl-tmp/logs/xxx`）

### 5.5 一个全量微调的 voxcpm_finetune_all.yaml 的文件示例：

数据集有1306条，使用的是AutoDL的RTX PRO 6000进行的训练，数据盘扩容到了350GB。
```
pretrained_path: /root/models/VoxCPM2/
train_manifest: /root/autodl-tmp/datasheet/train_data.jsonl
val_manifest: /root/autodl-tmp/datasheet/val_data.jsonl
sample_rate: 16000        # AudioVAE encoder input rate; must match audio_vae_config.sample_rate
out_sample_rate: 48000    # AudioVAE decoder output rate; used for TensorBoard audio logging
batch_size: 8
grad_accum_steps: 2  # effective batch size = batch_size × grad_accum_steps = 16
num_workers: 8
num_iters: 3000
log_interval: 10
valid_interval: 150
save_interval: 300
learning_rate: 0.00001
weight_decay: 0.01
warmup_steps: 100
max_steps: 3000
max_batch_tokens: 8192
max_grad_norm: 1.0        # gradient clipping max norm; 0 = disabled
save_path: /root/autodl-tmp/checkpoints/finetune_all
tensorboard: /root/autodl-tmp/logs/finetune_all
lambdas:
  loss/diff: 1.0
  loss/stop: 1.0

```
---

## 六、启动训练

```bash
bash /root/2-run-train.sh
```

脚本会自动扫描 `/root/autodl-tmp/datasheet/` 下的 yaml 文件，展示配置信息让你确认后启动训练。

建议在 screen 会话里跑：
```bash
screen -S train
bash /root/2-run-train.sh
# Ctrl+A 然后 D 放后台
# screen -r train 回来看进度
```

训练过程中如果磁盘满了会报如下错误：
```
RuntimeError: [enforce fail at inline_container.cc:858] . PytorchStreamWriter failed writing file data/44: file write failed
```
解决方法：清理不需要的旧 checkpoint，或扩容数据盘。

---

## 七、训练监控

另开一个终端，启动 TensorBoard：
```bash
tensorboard --logdir /root/autodl-tmp/logs/<日志目录> --port 6006
```

通过 AutoDL「自定义服务」或 SSH 隧道转发 6006 端口到本地浏览器。

关注指标：
| 指标 | 健康表现 |
|---|---|
| `loss/diff` | 平稳下降后趋于平缓 |
| `loss/stop` | 前 100-200 步快速下降后保持低位 |
| `grad_norm` | 大致在 0.3-2.0 之间 |
| `val/loss` | 随训练损失下降；如果训练损失继续降但 val/loss 上升 → 停止训练 |

TTS 微调通常 **1-2 个 epoch 足够**。最佳 checkpoint 往往不在最后。

---

## 八、推理验证

训练完成后，使用交互式脚本测试 checkpoint：
```bash
bash /root/3-run-infer.sh
```

脚本会自动扫描所有 checkpoint，引导你选择、输入文本、可选 voice cloning，然后生成 wav。

也可以启动 WebUI 做对比试听：
```bash
bash /root/1-run-webui.sh
```

---

## 九、注意事项

- 确保 GPU 显存充足（VoxCPM2 全量微调约 40 GB，LoRA 约 20 GB）
- 建议使用 screen 或 nohup 挂后台运行训练，防止 SSH 断连
- 训练数据质量直接影响模型效果，务必严格执行 SRT 数据清洗标准
- VoxCPM 官方警告：尾静音过长（> 0.5s）会导致生成停不下来，segment_audio.py 已硬性限制 END_SILENCE_MS ≤ 500
- 不需要手动重采样训练数据——训练时 dataloader 会根据 yaml 的 sample_rate 字段自动重采样
- 定期检查磁盘空间，每个 VoxCPM2 全量 checkpoint 约 5 GB

---

## 十、相关资源

- **Parakeet**：高精度 ASR 工具
- **Google AI Studio**：[https://aistudio.google.com](https://aistudio.google.com)
- **AutoDL 平台**：[https://autodl.com](https://autodl.com)
- **VoxCPM 官方仓库**：[https://github.com/OpenBMB/VoxCPM](https://github.com/OpenBMB/VoxCPM)
- **ModelScope VoxCPM2**：[https://modelscope.cn/models/OpenBMB/VoxCPM2](https://modelscope.cn/models/OpenBMB/VoxCPM2)
- **ModelScope VoxCPM1.5**：[https://modelscope.cn/models/OpenBMB/VoxCPM1.5](https://modelscope.cn/models/OpenBMB/VoxCPM1.5)

---

*更新于：2026-04-16*
