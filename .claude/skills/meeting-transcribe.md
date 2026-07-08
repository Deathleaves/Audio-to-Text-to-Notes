---
name: meeting-transcribe
description: 将会议录音自动转为带说话人区分的文本，并生成会议纪要。触发方式：用户在 Meeting 项目中发送音频文件即可。
---

# 会议录音转文字 + 纪要生成

## 概述

当用户发送音频文件时，自动执行以下流程：
1. FFmpeg 音频预处理（自动检测格式、时长，按需分段）
2. whisper.cpp **GPU 加速**语音转文字（本地部署，RTX 5090D，中文 large-v3-turbo Q8 模型）
3. Claude 分析文本，区分说话人
4. 输出「完整讲话记录」Markdown 文件
5. Claude 生成「会议纪要」Markdown 文件

---

## 环境初始化（每次执行前自动运行）

> **⚠️ 已知问题：Skill 自动调用失效**
> 当前系统无法自动识别 `.claude/skills/meeting-transcribe.md`，调用 `Skill: meeting-transcribe` 会返回 "Unknown skill"。
> **变通方案**：直接读取 `.claude/skills/meeting-transcribe.md` 并按其中的流程手动执行。

### Windows 路径约定（⚠️ 必须遵守）

在 Windows 上执行 whisper.cpp 时，**必须使用 `.exe` 后缀**：

```bash
# ✅ 正确
~/whisper.cpp/build/bin/whisper-cli.exe -m ...

# ❌ 错误 — 会报 "command not found"
~/whisper.cpp/build/bin/whisper-cli -m ...
```

> **原因**：Windows Git Bash 不会自动补全 `.exe`，与 Linux/Mac 不同。

### PATH 环境变量

**不需要手动设置 PATH**。FFmpeg 已通过 winget 安装到系统 PATH 中，whisper.cpp 使用绝对路径调用，CUDA runtime 由 whisper.cpp 自动加载。

> **历史教训**：之前尝试用 `find` 命令自动查找 FFmpeg 和 CUDA 路径并设置 PATH，
> 但 `find` 在 Windows 上经常返回空结果导致 `dirname: missing operand` 错误。
> 如果 FFmpeg 或 whisper-cli 不在 PATH 中，直接使用 `which ffmpeg` 或绝对路径即可。

**GPU 可用性验证**（每次转录前确认）：
```bash
nvidia-smi --query-gpu=name,memory.total --format=csv,noheader 2>/dev/null
```

## 前置依赖检查（首次执行时引导用户）

在执行流程前，先检查必要工具是否就绪。

### 检查 FFmpeg

```bash
ffmpeg -version 2>/dev/null && echo "FFmpeg OK" || echo "FFmpeg NOT FOUND"
```

如果未安装，告知用户：
> **Windows**: `winget install Gyan.FFmpeg`  
> **Mac**: `brew install ffmpeg`

### 检查 whisper.cpp（需 CUDA/GPU 版本）

> **⚠️ Windows 上必须用 `.exe` 后缀**

```bash
~/whisper.cpp/build/bin/whisper-cli.exe --version 2>/dev/null | head -1
```

验证 GPU 支持（在转录输出中应显示 `CUDA0` 设备和 VRAM 信息）：
```bash
~/whisper.cpp/build/bin/whisper-cli.exe -m ~/whisper.cpp/models/ggml-large-v3-turbo-q8_0.bin -l zh -f /dev/null 2>&1 | grep -i cuda || echo "⚠ GPU 可能未启用"
```

如果未找到，告知用户：

> 请先部署 whisper.cpp（只需一次）：
> **Windows**：
> 1. 安装前置工具：`winget install Kitware.CMake Gyan.FFmpeg BrechtSanders.WinLibs.POSIX.UCRT`
> 2. 安装 **CUDA Toolkit**：`winget install Nvidia.CUDA`
> 3. 安装 **Visual Studio Build Tools**（含 C++ 工作负载）
> 4. 编译 CUDA 版 whisper.cpp：
> ```bash
> git clone https://github.com/ggerganov/whisper.cpp.git ~/whisper.cpp
> cd ~/whisper.cpp
> cmake -B build -DCMAKE_BUILD_TYPE=Release -DGGML_CUDA=ON
> cmake --build build --config Release --parallel
> ```
> 5. 下载模型：
> ```bash
> bash ./models/download-ggml-model.sh large-v3-turbo
> # 然后量化为 Q8_0：
> ~/whisper.cpp/build/bin/whisper-quantize.exe ~/whisper.cpp/models/ggml-large-v3-turbo.bin ~/whisper.cpp/models/ggml-large-v3-turbo-q8_0.bin q8_0
> ```

### 检查 whisper.cpp 模型文件

```bash
test -f ~/whisper.cpp/models/ggml-large-v3-turbo-q8_0.bin && echo "Model OK" || echo "Model NOT FOUND"
```

如果未找到，告知用户：

> whisper.cpp 模型文件未下载，请运行：
> ```bash
> cd ~/whisper.cpp && bash ./models/download-ggml-model.sh large-v3-turbo
> ```
> 下载后量化为 Q8_0：
> ```bash
> ~/whisper.cpp/build/bin/whisper-quantize.exe ~/whisper.cpp/models/ggml-large-v3-turbo.bin ~/whisper.cpp/models/ggml-large-v3-turbo-q8_0.bin q8_0
> ```

如果所有依赖就绪，直接进入流程。

---

## 执行流程

### Step 0: 确认输入

1. 确认用户提供的音频文件路径
2. 如果有多个音频文件，询问用户：
   - 这些是否是同一场会议的分段录音？（是 → 按文件名排序合并处理）
   - 还是多场不同的会议？（是 → 分别处理，每场输出独立的两个文件）
3. 如果用户指出了会议名称/主题，记录用于命名输出文件；否则从文件名推断

### Step 1: 音频文件信息检测

对每个音频文件，使用 ffprobe 获取基本信息：

```bash
ffprobe -v quiet -show_entries format=duration,format_name,size -of json <音频文件路径>
```

向用户展示音频文件摘要：
- 文件名、格式、时长
- 如果有多个文件：总时长、文件数
- 预计处理时间（**GPU RTX 5090D**：30 分钟音频约 30-60 秒，1 小时约 1-2 分钟）

**⚠️ 重要：告知用户预计处理时间，让用户确认后再继续！**

### Step 2: FFmpeg 预处理

对每个音频文件，转为 whisper.cpp 标准输入格式：

```bash
# 检测时长
DURATION=$(ffprobe -v quiet -show_entries format=duration -of csv=p=0 "<输入文件>")
AUDIO_SECONDS=${DURATION%.*}

# ⚠️ 关键规则：无论音频长短，都必须先归一化音量，再去除静音，再分段！
# 1. 音量归一化：低音量音频（如 Mac 录音 RMS -33dB）会导致 whisper.cpp 产生幻觉
# 2. 去除静音：避免转录伪影（同一段话被反复识别数百次）
# 3. 分段：每段 ≤ 15 分钟，避免模型"遗忘"前面语境

# 第一步：音量归一化（loudnorm），转为 16kHz 单声道
# 先测量音频响度
MEASURE=$(ffmpeg -y -i "<输入文件>" -af "loudnorm=I=-23:TP=-1.5:LRA=20:print_format=json" -f null - 2>&1 | grep -A99 '"input_i"' | head -10)
INPUT_I=$(echo "$MEASURE" | grep '"input_i"' | sed 's/.*: *"\([^"]*\)".*/\1/')
INPUT_TP=$(echo "$MEASURE" | grep '"input_tp"' | sed 's/.*: *"\([^"]*\)".*/\1/')
INPUT_LRA=$(echo "$MEASURE" | grep '"input_lra"' | sed 's/.*: *"\([^"]*\)".*/\1/')
INPUT_THRESH=$(echo "$MEASURE" | grep '"input_thresh"' | sed 's/.*: *"\([^"]*\)".*/\1/')
OFFSET=$(echo "$MEASURE" | grep '"target_offset"' | sed 's/.*: *"\([^"]*\)".*/\1/')

# 双通道归一化（更精确）
ffmpeg -y -i "<输入文件>" \
  -af "loudnorm=I=-23:TP=-1.5:LRA=20:measured_I=${INPUT_I}:measured_TP=${INPUT_TP}:measured_LRA=${INPUT_LRA}:measured_thresh=${INPUT_THRESH}:offset=${OFFSET}:linear=true" \
  -ar 16000 -ac 1 -c:a pcm_s16le "<输出前缀>_normalized.wav"

# 检查归一化后音量
NORM_RMS=$(ffmpeg -y -i "<输出前缀>_normalized.wav" -af "astats" -f null - 2>&1 | grep "RMS level" | head -1 | sed 's/.*: *\(-[0-9.]*\).*/\1/')
echo ">>> 音量归一化完成，RMS: ${NORM_RMS}dB（目标约 -23dB）"

# 第二步：去除首尾静音（归一化后阈值 -40dB 更合理）
ffmpeg -y -i "<输出前缀>_normalized.wav" \
  -af "silenceremove=start_periods=1:start_duration=3:start_threshold=-40dB:stop_periods=1:stop_duration=3:stop_threshold=-40dB" \
  -ar 16000 -ac 1 -c:a pcm_s16le "<输出前缀>_denoised.wav"

# 清理中间文件
rm -f "<输出前缀>_normalized.wav"

# 第三步：根据时长决定是否分段
DENOISED_DURATION=$(ffprobe -v quiet -show_entries format=duration -of csv=p=0 "<输出前缀>_denoised.wav")
DENOISED_SECONDS=${DENOISED_DURATION%.*}

if [ "$DENOISED_SECONDS" -gt 900 ]; then
  # 分段（每 15 分钟一段）
  ffmpeg -y -i "<输出前缀>_denoised.wav" \
    -f segment -segment_time 900 \
    -c:a pcm_s16le "<输出前缀>_%03d.wav"

  # 清理中间文件
  rm -f "<输出前缀>_denoised.wav"
  echo ">>> 音频已归一化音量、去除静音并分为多段（每段 ≤ 15 分钟）"
else
  # 直接使用
  mv "<输出前缀>_denoised.wav" "<输出文件>.wav"
  echo ">>> 音频已归一化音量、去除首尾静音并转换为 16kHz 单声道 WAV"
fi
```

> **为什么先归一化音量？** 低音量音频（如 Mac 录音 RMS -33dB）会让 whisper.cpp 在低音量区域产生"幻觉"——将同一段话反复识别为转录结果。先归一化到标准响度（-23 LUFS），模型就能正确识别。
>
> **为什么双通道 loudnorm？** 第一通道测量音频实际响度，第二通道用测量值精确归一化，比单通道更准确。
>
> **为什么归一化后静音阈值改为 -40dB？** 归一化后音量提升到标准水平，-40dB 阈值能更准确地检测真正的静音段。

**输出路径规范**：所有预处理后的 WAV 文件放在项目 `audio/` 目录下。

### Step 3: whisper.cpp GPU 加速转录

对每个预处理后的 WAV 文件，执行转录（**GPU 自动启用**，无需额外参数）：

> **⚠️ Windows 上必须用 `.exe` 后缀**

```bash
~/whisper.cpp/build/bin/whisper-cli.exe \
  -m ~/whisper.cpp/models/ggml-large-v3-turbo-q8_0.bin \
  -l zh \
  -osrt \
  -otxt \
  -f "<WAV 文件路径>" \
  -of "<输出路径前缀>"
```

> **GPU 加速**：RTX 5090D (32GB VRAM)，whisper-cli 编译时已启用 CUDA，运行时自动检测 GPU。
> 模型完全加载到 GPU 显存（~0.8GB），encode 时间比 CPU 快 **~157x**。

参数说明：
- `-m`: 模型文件路径
- `-l zh`: 指定中文语言（避免自动检测错误）
- `-osrt`: 输出 SRT 格式（带毫秒级时间戳，用于后续说话人区分）
- `-otxt`: 同时输出纯文本
- `-f`: 输入音频文件
- `-of`: 输出文件路径前缀（会生成 `.srt` 和 `.txt` 两个文件）

**输出文件**：将转录结果保存在 `output/` 目录下，命名格式为 `<会议名>_transcript.srt` 和 `<会议名>_transcript.txt`。

**多段处理**：如果 Step 2 产生了多个分段，按顺序对每个分段执行转录，然后将文本内容按时间顺序合并（注意分段可能有时间偏移，需要累加时间）。

### Step 3.5: 检查转录质量（⚠️ 重要）

在继续 Step 4 之前，先检查 SRT 输出是否存在以下问题：

#### 3.5.1 重复文本检测

如果 SRT 中出现**同一段话被反复识别**（如"你要求名证的时候一定要注意"出现数百次），
说明音频末尾有静音或回音导致 whisper.cpp 产生**转录伪影**。

**处理方式**：
- 在完整讲话记录中标注 `[转录伪影]`，说明该段内容为无效重复
- 不要将伪影内容计入有效转录文字数
- 在文档开头用 `> ⚠️ 说明` 告知用户存在转录伪影及原因

#### 3.5.2 专业术语偏差

whisper.cpp 对人名、课程名、专业术语的识别可能不准确（如"全球的人口"可能是某个人名或机构名）。

**处理方式**：
- 无法确认的词用 `[听不清]` 标注
- 明显是专有名词但识别错误的，用 `[听不清：可能是XXX]` 标注猜测

#### 3.5.3 转录结果为空

如果转录结果几乎为空或只有少量无意义文字，检查音频是否只有静音/噪音。

---

## Step 4: Claude 分析转录文本 + 说话人区分

读取 whisper.cpp 输出的 SRT 文件（和 TXT 文件作为辅助），执行以下分析：

#### 4.1 解析 SRT 时间戳
- SRT 格式：序号 → `开始时间 --> 结束时间` → 文本 → 空行
- 提取每段的起止时间和文本内容
- 多段合并时，后续分段的时间需加上前面分段的总时长

#### 4.2 推断说话人
根据以下线索推断说话人切换点：
- **对话轮转**：一问一答的结构
- **观点对立**：出现 "但是"、"我不同意"、"我觉得" 等转折
- **称呼语**：出现 "XX你觉得呢"、"XX来说一下" 等指名道姓的表达
- **话题引导**：出现 "接下来"、"下面"、"我们讨论一下" 等主持会议的语言
- **语义边界**：前后文主题明显变化

#### 4.3 标注规则
- 首次出现的说话人标注为 **Speaker A**，第二个为 **Speaker B**，以此类推
- 如果上下文能推断角色（如"我是产品部的XX"），则在括号中标注角色，如 **Speaker A（产品经理）**
- 如果某段话明显是会议主持人说的（如开场白、流程引导），标注为 **Speaker A（主持人）**
- 合并相邻的同一说话人的连续段落
- 短路消除：如果某段极短（< 3 秒）且夹在同一说话人的两段之间，可能是转录碎片，合并处理

#### 4.4 输出「完整讲话记录」Markdown

写入 `output/<日期>【完整讲话记录】<会议名>.md`：

```markdown
# 完整讲话记录：<会议名称>

**日期**：YYYY-MM-DD
**时长**：HH:MM:SS
**音频文件**：xxx.mp3, xxx.mp3
**转录引擎**：whisper.cpp large-v3-turbo-q8_0
**说话人区分**：Claude 自动分析

---

[00:00:05] **Speaker A（主持人）**：今天我们来讨论一下下一版本的功能规划...

[00:00:30] **Speaker B**：我觉得优先级的排序应该以用户反馈为准...

[00:02:15] **Speaker A（主持人）**：同意这个方向，那下面请 Speaker C 介绍一下调研结果。

[00:02:30] **Speaker C**：好的，上周我们调研了 100 位用户...

...（全量内容，不省略）

---

> 共识别 N 位说话人 | 总段落数：M | 转录文字数：K
```

**注意**：
- 保留所有转录内容，不省略、不总结
- 时间戳使用 `[HH:MM:SS]` 格式
- 如果 whisper.cpp 转录有明显错误（如识别出的无意义字词），用 `[听不清]` 标记替换，不要保留乱码

### Step 5: Claude 生成「会议纪要」Markdown

基于 Step 4 的完整讲话记录，生成结构化的会议纪要。

写入 `output/<日期>【会议纪要】<会议名>.md`：

```markdown
# 会议纪要：<会议名称>

## 基本信息
- **日期**：YYYY-MM-DD
- **时长**：HH:MM:SS
- **参会人**：（从对话中推断，列出可识别的角色和姓名）
- **记录方式**：AI 自动转录 + 摘要生成

## 会议主题

用 1-2 句话概括本次会议的核心议题和目的。

## 讨论要点

### 1. <话题一>
- 主要观点和讨论内容
- 各方立场（如果存在分歧）

### 2. <话题二>
- ...

> 按讨论的时间顺序或逻辑分组，每个话题 3-5 个要点

## 决议与结论

1. 明确达成的共识或决定
2. ...

## 待办事项

- [ ] <具体任务> @<负责人>（如果对话中提到）⏱ <截止时间>（如果提到）
- [ ] ...

## 下次会议

（如果对话中提到下次会议时间或议题，在此记录；否则省略此节）

---
*由 Claude 基于 whisper.cpp 转录文本自动生成，如有遗漏或错误请修正*
```

**纪要质量要求**：
- **准确**：只基于实际对话内容，不添加虚构信息
- **完整**：覆盖所有讨论过的话题，不遗漏
- **简洁**：每个要点精炼到 1-3 句话
- **可执行**：待办事项具体明确，包含负责人（如果对话中提到）
- 如果对话中无法确定某个信息（如日期、参会人姓名），标注 `[待确认]` 而非编造

### Step 6: 清理中间文件

转录完成后，清理预处理产生的临时文件，只保留最终输出：

```bash
# 删除预处理后的 WAV 文件
rm -f "<audio/目录下的_preprocessed.wav 文件>"

# 删除转录中间文件
rm -f "<output/目录下的 _transcript.srt 文件>"
rm -f "<output/目录下的 _transcript.txt 文件>"
```

只保留 `output/` 目录下的两个 Markdown 文件：
- `<日期>【完整讲话记录】<会议名>.md`
- `<日期>【会议纪要】<会议名>.md`

### Step 7: 输出总结

向用户展示处理结果摘要：

```
✅ 处理完成！

📁 输出文件：
  - output/【完整讲话记录】<会议名>.md  （完整对话，N 个说话人）
  - output/【会议纪要】<会议名>.md       （结构化纪要）

📊 统计：
  - 音频时长：HH:MM:SS
  - 转录字数：X 字
  - 识别说话人：N 位
  - 处理用时：M 分钟
```

---

## 错误处理

| 情况 | 处理方式 |
|------|----------|
| FFmpeg 未安装 | 引导安装 `winget install Gyan.FFmpeg` (Windows) / `brew install ffmpeg` (Mac)，暂停流程 |
| whisper.cpp 未部署 | 引导部署（见前置依赖检查），暂停流程 |
| whisper.cpp 模型未下载 | 引导下载模型，暂停流程 |
| GPU 未被检测到 | 检查 CUDA 驱动和 Runtime 是否正常，验证 `nvidia-smi` |
| 音频文件不存在/损坏 | 告知用户并跳过该文件 |
| 转录结果为空 | 检查音频是否只有静音/噪音，建议用户确认音频质量 |
| 单文件 > 2 小时 | 警告用户处理时间，建议先手动分段 |
| whisper.cpp 中途崩溃 | 重试一次，如仍失败告知用户（可能是模型或显存问题） |
| CUDA OOM（显存不足） | 降级到 CPU 模式（添加 `-ng` 参数禁用 GPU） |

---

## 注意事项

1. **GPU 加速**：RTX 5090D (32GB VRAM, Blackwell, FP4 原生)，30 分钟音频约 30-60 秒完成。GPU 默认自动启用。
2. **中文优化**：whisper.cpp 使用 `-l zh` 强制中文识别，避免中英混合时自动检测跳转。
3. **说话人区分是推断**：Claude 基于文本上下文推断，不是声纹识别，准确性取决于对话的清晰度。在实际使用中告知用户这一限制。
4. **隐私**：所有音频处理都在用户本地完成（whisper.cpp 本地运行），不经过任何云端服务。
5. **输出文件**：两个 Markdown 文件都保存在 `output/` 目录，如果文件已存在则自动追加序号（如 `【会议纪要】XXX_2.md`）。
6. **日期前置**：输出文档中日期（YYYY-MM-DD 格式）必须醒目地写在文档前面，方便按时间查找和归档。
7. **Windows `.exe` 后缀**：在 Windows 上执行 whisper-cli 时**必须**加 `.exe` 后缀，否则会报 "command not found"。

## 快捷指令

用户只需要：
1. 把音频文件放入项目目录（或发送文件路径）
2. 说 "处理这个音频" 或类似指令
3. 等待完成

无需手动指定格式、采样率等参数 — 所有预处理都由 Skill 自动完成。
