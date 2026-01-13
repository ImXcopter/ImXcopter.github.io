# VoxCPM 全量微调完整指南

本文档提供 VoxCPM 模型全量微调的完整流程，包括数据准备、处理和训练步骤。

---

## 一、数据准备

### 1.1 音频素材要求

确保原始音频满足以下质量标准:

- **纯净度要求**:无背景音乐、无环境杂音
- **推荐来源**:YouTube 视频/音频等高质量资源
- **格式建议**:WAV 格式(无损音质)

### 1.2 音频预处理

使用 **Capcut** 进行音频清理:

1. 剪除片头片尾含音乐的部分
2. 去除中间的杂音片段
3. 导出为 WAV 格式

### 1.3 语音转写

使用 **Parakeet** 工具对 WAV 音频进行自动语音识别(ASR),生成 SRT 字幕文件。

---

## 二、SRT 字幕处理

### 2.1 处理平台

访问 [Google AI Studio](https://aistudio.google.com) 使用 AI 辅助处理。

### 2.2 处理 Prompt

将以下 Prompt 提供给 AI 模型,并附上原始 SRT 文件:

```
【角色设定】
你是一位专业的 TTS(语音合成)数据集处理专家,擅长进行高精度的 SFT 数据对齐与清洗。

【任务目标】
我将提供一份由 ASR(自动语音识别)生成的原始 SRT 字幕文件。请你将其处理为高质量的 SFT 训练数据清单。

【核心执行标准(必须严格遵守)】

1. 时间轴精度(最高优先级)
   - 严禁修改原始时间戳的精度!严禁四舍五入!
   - 原始数据是多少毫秒(例如 00:27:20,960),输出必须保持一致,不得改为 00:27:20,000
   - 合并时,新片段的"开始时间"取第一句的开始,"结束时间"取最后一句的结束

2. 时长控制(适配大显存)
   - 目标切片时长:5-12 秒
   - 12 秒为推荐阈值,最多不超过 15 秒
   - 通过合并短句来达到此时长

3. 语义完整性(核心逻辑)
   - 语义优先级 > 时长优先级
   - 必须确保句子的完整性,严禁在半句话(如逗号后、从句中)截断
   - 将破碎的 ASR 短句合并为完整的意群(完整的陈述句、复句)

4. 文本清洗与精修
   - 标点:修改或添加正确的标点符号(逗号、句号、问号、引号等),这决定了模型的语气和韵律
   - 拼写:修正明显的 ASR 识别错误(单词错误、拼写错误)
   - 格式:修正英文的大小写规范(句首大写、专有名词大写)

【输出格式】
请直接输出处理后的 SRT 内容,格式如下:

1
00:00:00,000 --> 00:00:12,240
[这里是修正后的、完整的、带有标点符号的文本]

2
00:00:13,120 --> 00:00:26,160
[下一段文本...]

【待处理的字幕内容如下】:
[在此粘贴原始 SRT 内容]
```

### 2.3 音频分割

使用 `segment_audio.py` 脚本根据处理后的 SRT 文件分割音频:

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
AUDIO_FILENAME = "A01.wav"                # 音频文件名
SRT_FILENAME = "A01.srt"        # SRT 文件名

# VAD 参数
BUFFER_MS = 100           # SRT 时间定位 Buffer（前后扩展时长，单位：毫秒）
SILENCE_MS = 200          # 前后静音时长（单位：毫秒）
VAD_THRESHOLD = 0.5       # VAD 检测阈值（0-1，越高越严格）

# 音频采样率要求
REQUIRED_SAMPLE_RATE = 44100  # VoxCPM1.5 要求 44.1kHz

# ============================================================================
# 工具函数
# ============================================================================

def clear_screen() -> None:
    """清屏函数"""
    os.system('cls' if os.name == 'nt' else 'clear')

def get_script_dir() -> Path:
    """获取脚本所在目录"""
    return Path(__file__).parent.resolve()

def get_audio_sample_rate(audio_path: Path) -> Optional[int]:
    """
    使用 ffprobe 获取音频采样率

    Args:
        audio_path: 音频文件路径

    Returns:
        int: 采样率，失败返回 None

    Raises:
        RuntimeError: 如果系统未安装 ffprobe
    """
    try:
        result = subprocess.run(
            [
                'ffprobe',
                '-v', 'error',
                '-select_streams', 'a:0',
                '-show_entries', 'stream=sample_rate',
                '-of', 'default=noprint_wrappers=1:nokey=1',
                str(audio_path)
            ],
            capture_output=True,
            text=True,
            timeout=10
        )

        if result.returncode == 0 and result.stdout.strip():
            return int(result.stdout.strip())
        else:
            return None

    except FileNotFoundError:
        raise RuntimeError("系统未安装 ffprobe，无法检查音频采样率。请安装 ffmpeg 后重试。")
    except subprocess.TimeoutExpired:
        print(f"  警告：ffprobe 检查超时")
        return None
    except Exception as e:
        print(f"  ffprobe 检查失败：{e}")
        return None

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
    """检查前置条件"""
    script_dir = get_script_dir()

    # 检查音频文件
    audio_path = script_dir / AUDIO_FILENAME
    if not audio_path.exists():
        print(f"错误：音频文件不存在")
        print(f"路径：{audio_path}")
        print(f"请检查配置常量 AUDIO_FILENAME 是否正确")
        return False

    # 检查音频采样率
    print(f"检查音频采样率...")
    try:
        sample_rate = get_audio_sample_rate(audio_path)
    except RuntimeError as e:
        print(f"错误：{e}")
        return False

    if sample_rate is None:
        print(f"错误：无法获取音频采样率")
        return False

    if sample_rate != REQUIRED_SAMPLE_RATE:
        print(f"错误：音频采样率不符合要求")
        print(f"  当前采样率：{sample_rate} Hz")
        print(f"  要求采样率：{REQUIRED_SAMPLE_RATE} Hz")
        print(f"  VoxCPM1.5 必须使用 44.1kHz 音频")
        print(f"程序终止！")
        return False

    print(f"音频采样率检查通过：{sample_rate} Hz ✓")

    # 检查 SRT 文件
    srt_path = script_dir / SRT_FILENAME
    if not srt_path.exists():
        print(f"错误：SRT 文件不存在")
        print(f"路径：{srt_path}")
        print(f"请检查配置常量 SRT_FILENAME 是否正确")
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
    silence_samples: int
) -> Optional[Tuple[Dict[str, Any], Optional[str]]]:
    """
    处理单个音频片段

    Args:
        seg: SRT 段落信息
        audio: 原始音频数组
        sr: 采样率
        audio_duration: 音频总时长
        output_dir: 输出目录
        current_index: 当前片段序号
        vad_model: Silero VAD 模型
        get_speech_timestamps: VAD 工具函数
        resampler: 重采样器
        target_sr: 目标采样率（16kHz）
        buffer_samples: Buffer 样本数
        silence_samples: 静音样本数

    Returns:
        Optional[Tuple[Dict, Optional[str]]]: (训练数据字典, VAD警告信息) 或 None（失败）
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
        print(f"  ⚠️  段落 {current_index:04d}: 搜索区域为空，跳过")
        return None

    # 2. 使用 Silero VAD 在搜索区域内精准检测（16kHz）
    vad_warning = None
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

            # 转换为原始音频的绝对位置（样本点，保持原始 44.1kHz）
            absolute_start_sample = search_start_sample + int(relative_start * sr)
            absolute_end_sample = search_start_sample + int(relative_end * sr)

        else:
            # VAD 检测失败：退回到 SRT 时间轴
            vad_warning = f"段落 {current_index:04d}: VAD 未检测到语音，使用 SRT 时间轴"
            absolute_start_sample = int(srt_start * sr)
            absolute_end_sample = int(srt_end * sr)

        # 边界检查
        absolute_start_sample = max(0, absolute_start_sample)
        absolute_end_sample = min(absolute_end_sample, len(audio))

        if absolute_end_sample <= absolute_start_sample:
            print(f"  ⚠️  段落 {current_index:04d}: 无效的时间范围，跳过")
            return None

        # 4. 提取精准的语音内容（保持原始 44.1kHz）
        audio_content = audio[absolute_start_sample:absolute_end_sample]

        # 5. 添加前后静音
        front_silence = np.zeros(silence_samples, dtype=audio.dtype)
        end_silence = np.zeros(silence_samples, dtype=audio.dtype)
        audio_final = np.concatenate([front_silence, audio_content, end_silence])

        # 6. 保存音频文件
        output_filename = f"{current_index:04d}.wav"
        output_path = output_dir / output_filename
        sf.write(output_path, audio_final, sr)

        # 7. 验证保存的音频采样率
        verify_sr = get_audio_sample_rate(output_path)
        if verify_sr != REQUIRED_SAMPLE_RATE:
            print(f"  ⚠️  警告：保存的音频采样率不正确 ({verify_sr} Hz)")

        # 8. 使用 ffprobe 获取精确 duration
        duration = get_duration_with_ffprobe(output_path)

        # 9. 返回训练数据
        training_item = {
            'audio': str(output_path.absolute()),
            'text': text,
            'duration': round(duration, 2)
        }

        return (training_item, vad_warning)

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
    silence_ms: int = SILENCE_MS
) -> Tuple[List[Dict[str, Any]], List[str]]:
    """
    使用 Silero VAD 精准切分音频

    处理流程：
    1. 根据 SRT 时间轴大致定位（扩展 buffer）
    2. 在该区域内使用 VAD 精准检测语音边界（16kHz）
    3. 提取精准的语音内容（保持原始 44.1kHz）
    4. 添加前后静音
    5. 保存并获取 duration

    Args:
        audio_path: 原始音频文件路径
        segments: SRT 段落列表
        output_dir: 输出目录
        vad_model: Silero VAD 模型
        get_speech_timestamps: VAD 工具函数
        start_index: 起始序号
        buffer_ms: SRT 时间定位 Buffer（毫秒）
        silence_ms: 前后静音时长（毫秒）

    Returns:
        tuple: (训练数据列表, VAD 失败警告列表)
    """
    import numpy as np
    import soundfile as sf
    import torch
    import torchaudio
    from tqdm import tqdm

    # 加载原始音频
    print(f"📂 加载音频: {audio_path.name}")
    audio, sr = sf.read(audio_path)

    # 验证采样率
    if sr != REQUIRED_SAMPLE_RATE:
        print(f"  错误：音频采样率不匹配")
        print(f"    期望：{REQUIRED_SAMPLE_RATE} Hz")
        print(f"    实际：{sr} Hz")
        sys.exit(1)

    # 如果是立体声，转为单声道
    if len(audio.shape) > 1:
        audio = np.mean(audio, axis=1)

    audio_duration = len(audio) / sr
    print(f"✅ 音频加载成功 - 采样率: {sr} Hz, 时长: {audio_duration:.2f} 秒")

    # 创建输出目录
    os.makedirs(output_dir, exist_ok=True)

    # Silero VAD 需要 16000 Hz，准备重采样器（仅用于 VAD 检测）
    target_sr = 16000
    resampler = torchaudio.transforms.Resample(orig_freq=sr, new_freq=target_sr)

    # 处理数据
    training_data = []
    vad_fallback_warnings = []
    skipped = 0

    buffer_samples = int(buffer_ms / 1000 * sr)
    silence_samples = int(silence_ms / 1000 * sr)

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
            silence_samples=silence_samples
        )

        if result is None:
            skipped += 1
            continue

        training_item, vad_warning = result
        training_data.append(training_item)

        if vad_warning:
            vad_fallback_warnings.append(vad_warning)
            # 实时反馈 VAD 失败
            tqdm.write(f"  ⚠️  {vad_warning}")

    print(f"\n✅ 切分完成: {len(training_data)} 个片段")
    if skipped > 0:
        print(f"⚠️  跳过了 {skipped} 个无效片段")

    return training_data, vad_fallback_warnings

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
    print(f"   - Buffer 时长: {BUFFER_MS} ms")
    print(f"   - 静音时长: {SILENCE_MS} ms (前后)")
    print(f"   - VAD 阈值: {VAD_THRESHOLD}")
    print(f"   - 采样率要求: {REQUIRED_SAMPLE_RATE} Hz")
    print()

    # 设置路径
    audio_path = script_dir / AUDIO_FILENAME
    srt_path = script_dir / SRT_FILENAME
    data_dir = script_dir / "data"
    jsonl_path = data_dir / "train.jsonl"

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
        get_speech_timestamps, save_audio, read_audio, VADIterator, collect_chunks = utils
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

    # 切分音频
    print("-" * 60)
    print("开始切分音频...")
    print()

    training_data, vad_warnings = segment_audio_with_vad(
        audio_path=audio_path,
        segments=segments,
        output_dir=data_dir,
        vad_model=vad_model,
        get_speech_timestamps=get_speech_timestamps,
        start_index=start_index,
        buffer_ms=BUFFER_MS,
        silence_ms=SILENCE_MS
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
    short = sum(1 for d in durations if d < 3)
    normal = sum(1 for d in durations if 3 <= d <= 10)
    long = sum(1 for d in durations if d > 10)

    print(f"\n📊 时长分布:")
    print(f"   - < 3 秒: {short} ({short/len(durations)*100:.1f}%)")
    print(f"   - 3-10 秒: {normal} ({normal/len(durations)*100:.1f}%) ✅ 推荐")
    print(f"   - > 10 秒: {long} ({long/len(durations)*100:.1f}%)")
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

所有分割好的音频片段将自动保存在 `/data` 目录下。

---

## 三、模型训练

### 3.1 环境配置

1. 访问 [AutoDL 平台](https://autodl.com/)
2. 在应用市场搜索 **VoxCPM-1.5-TTS-WEB-UI**
3. 租用 **RTX PRO 6000** GPU(推荐配置)，同时增加数据盘的容量，默认的数据盘一般情况下不够用，因为要存储训练好的模型！

训练过程中遇到如下的报错，就说明硬盘塞满了：
```
[train] step 980: loss/diff: 0.638490, loss/stop: 0.000001, lr: 0.000008, epoch: 19.454094, grad_norm: 0.882882, log interval: 4.73s
[train] step 990: loss/diff: 0.634882, loss/stop: 0.000001, lr: 0.000008, epoch: 19.652605, grad_norm: 0.967825, log interval: 4.65s
[train] step 1000: loss/diff: 0.618469, loss/stop: 0.000002, lr: 0.000008, epoch: 19.851117, grad_norm: 0.960394, log interval: 4.77s
Traceback (most recent call last):
File "/root/miniconda3/lib/python3.12/site-packages/torch/serialization.py", line 967, in save
_save(
File "/root/miniconda3/lib/python3.12/site-packages/torch/serialization.py", line 1268, in _save
zip_file.write_record(name, storage, num_bytes)
RuntimeError: [enforce fail at inline_container.cc:858] . PytorchStreamWriter failed writing file data/44: file write failed

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
File "/root/VoxCPM/scripts/train_voxcpm_finetune.py", line 360, in <module>
train(**yaml_args)
File "/root/miniconda3/lib/python3.12/site-packages/argbind/argbind.py", line 159, in cmd_func
return func(*cmd_args, **kwargs)
^^^^^^^^^^^^^^^^^^^^^^^^^
File "/root/VoxCPM/scripts/train_voxcpm_finetune.py", line 261, in train
save_checkpoint(model, optimizer, scheduler, save_dir, step, pretrained_path)
File "/root/VoxCPM/scripts/train_voxcpm_finetune.py", line 348, in save_checkpoint
torch.save(optimizer.state_dict(), folder / "optimizer.pth")
File "/root/miniconda3/lib/python3.12/site-packages/torch/serialization.py", line 966, in save
with _open_zipfile_writer(f) as opened_zipfile:
File "/root/miniconda3/lib/python3.12/site-packages/torch/serialization.py", line 798, in exit
self.file_like.write_end_of_file()
RuntimeError: [enforce fail at inline_container.cc:664] . unexpected pos 757021440 vs 757021328
```

### 3.2 数据上传

使用 **WinSCP** 或其他 SFTP 工具,将 `/data` 目录下的所有音频文件上传至:

```
/root/autodl-tmp/data/
```

### 3.3 配置文件修改

编辑配置文件 `conf/voxcpm_v1.5/voxcpm_finetune_all.yaml`,主要修改:

- **数据路径**:指向上传的数据目录
- **训练参数**:batch size、learning rate、训练步数等
- **其他配置**:可咨询 AI 助手获取最佳实践建议

以下是使用RTX PRO 6000 时训练使用的配置文件：
```
pretrained_path: /root/models/VoxCPM-1.5/
train_manifest: /root/autodl-tmp/data/train.jsonl
val_manifest: null
sample_rate: 44100
batch_size: 16
grad_accum_steps: 1  # Gradient accumulation steps, >1 can increase effective batch size without increasing memory
num_workers: 8
num_iters: 3000
log_interval: 10
valid_interval: 500
save_interval: 500
learning_rate: 0.00001 
weight_decay: 0.01
warmup_steps: 100
max_steps: 3000
max_batch_tokens: 8192  # Example: single batch can have at most 16k tokens, with batch_size=4, each sample can have at most 4096 tokens
save_path: /root/autodl-tmp/checkpoints/finetune_all
tensorboard: /root/autodl-tmp/logs/finetune_all
lambdas:
  loss/diff: 1.0
  loss/stop: 1.0
```
训练好的模型会在/root/autodl-tmp/checkpoints/finetune_all
训练的日志会在/root/autodl-tmp/logs/finetune_all
可以把日志喂给gemini，让它通过日志分析理论上哪个模型效果最好。

注意修改训练.sh的py文件的名称，不是voxcpm_finetune_lora.yaml，而是voxcpm_finetune_all.yaml
```
cd /root/VoxCPM

python scripts/train_voxcpm_finetune.py --config_path conf/voxcpm_v1.5/voxcpm_finetune_all.yaml
echo '------------------------end.'
```
### 3.4 启动训练

按照 AutoDL 平台提供的训练脚本执行即可:

```
sh train.sh
```

---

## 四、注意事项

- 确保 GPU 显存充足(推荐 48GB 以上)
- 定期保存训练检查点,防止训练中断
- 监控训练日志,及时调整超参数
- 数据质量直接影响模型效果,务必严格执行数据清洗标准

---

## 五、相关资源

- **Parakeet**:高精度 ASR 工具
- **Google AI Studio**:[https://aistudio.google.com](https://aistudio.google.com)
- **AutoDL 平台**:[https://autodl.com](https://autodl.com)

---

*发布于:2026-01-13*
