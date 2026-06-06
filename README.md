# Meeting — 会议录音转文字 & 纪要生成 / Audio Transcription & Meeting Minutes

> **🇨🇳 全程本地运行**：音频不上传云端，基于 whisper.cpp GPU 加速 + Claude 智能分析。
> **🇬🇧 Fully local processing**: Audio never leaves your machine. Powered by whisper.cpp GPU acceleration + Claude AI analysis.

---

## 📋 效果预览 / Preview

以下是对一次实际会议录音的处理结果示例 / Below is a sample output from a real meeting recording (2026-06-06):

### 🇨🇳 完整讲话记录 / 🇬🇧 Full Transcript

带时间戳和说话人区分的逐字稿 / Verbatim transcript with timestamps and speaker labels:

```
[00:00:05] **Speaker A（主持人 / Host）**：今天我们来讨论一下下一版本的功能规划...
[00:00:30] **Speaker B**：我觉得优先级的排序应该以用户反馈为准...
[00:02:15] **Speaker A（主持人 / Host）**：同意这个方向，那下面请 Speaker C 介绍一下调研结果。
```

### 🇨🇳 会议纪要 / 🇬🇧 Meeting Minutes

结构化摘要 / Structured summary:

```markdown
## 会议主题 / Topic
（核心议题概述 / Summary of the core议题）

## 讨论要点 / Discussion Points
### 1. 话题一 / Topic 1
- 主要观点 / Key points...

## 决议与结论 / Decisions & Conclusions
1. 明确的共识 / Clear consensus...

## 待办事项 / Action Items
- [ ] 具体任务 / Task @负责人 / Assignee
```

---

## ✨ 功能特性 / Features

| 🇨🇳 特性 | 🇬🇧 Feature | 说明 / Description |
|----------|-------------|-------------------|
| 🎯 **自动说话人区分** | **Speaker Diarization** | Claude 基于对话上下文区分说话人并推断角色 / Identifies speakers from conversation context |
| 🚀 **GPU 极速转录** | **GPU-Accelerated** | RTX 5090D: 30min audio → ~30-60s transcription |
| 🔒 **完全本地处理** | **100% Local** | 音频不离开你的电脑 / Audio never leaves your machine |
| 🧩 **多段合并** | **Multi-part Merge** | 同会议多段录音自动排序合并 / Auto-merges multiple recordings of the same meeting |
| 📂 **结构化纪要** | **Structured Minutes** | 自动生成待办事项和负责人 / Auto-generated action items with assignees |
| 🗣 **多语言支持** | **Multi-language** | 中英混合场景也能较好处理 / Handles mixed Chinese-English speech well |

---

## 📦 前置依赖 / Prerequisites

需要你提前在本地准备好以下工具 / Install these tools beforehand:

### 1️⃣ FFmpeg（音频处理 / Audio Processing）

```bash
# Windows
winget install Gyan.FFmpeg

# macOS
brew install ffmpeg
```

### 2️⃣ whisper.cpp（语音转文字 / Speech-to-Text）

```bash
# 克隆 / Clone
git clone https://github.com/ggerganov/whisper.cpp.git ~/whisper.cpp

# CUDA 版编译（需 CUDA Toolkit + Visual Studio Build Tools）
cmake -B build -DCMAKE_BUILD_TYPE=Release -DGGML_CUDA=ON
cmake --build build --config Release --parallel

# 下载中文优化模型 / Download Chinese-optimized model
bash ./models/download-ggml-model.sh large-v3-q5_0
```

### 3️⃣ NVIDIA GPU + CUDA Toolkit

| GPU | 加速比 / Speed-up |
|-----|-------------------|
| RTX 5090D (32GB) | **~157x** encode 加速 / speed-up |
| 其他 NVIDIA GPU | 按显存/算力递减 / Scales with VRAM & compute |

---

## 🚀 快速使用 / Quick Start

### 🇨🇳 方式一：放入 audio/ 目录 / 🇬🇧 Option 1: Place files in audio/

```
1. 把 .mp3/.wav/.m4a 放入 audio/ 目录 / Place files in audio/
2. 告诉 Claude → "处理这个音频" / Tell Claude "transcribe this audio"
3. 等待自动完成 / Wait for completion
4. 在 output/ 查看结果 / Check output/ for results
```

### 🇨🇳 方式二：直接发送文件路径 / 🇬🇧 Option 2: Send file path directly

```
1. 发给 Claude 音频路径 / Send Claude the audio file path
2. Claude 自动触发 Skill / Claude auto-triggers the skill
3. 完成！/ Done!
```

### 🇨🇳 多段同会议录音 / 🇬🇧 Multi-part Recordings

按时间顺序命名即可自动合并 / Name sequentially for auto-merge:

```
audio/
├── 产品评审_001.mp3     ← 同一会议 自动合并 / Same meeting, auto-merged
├── 产品评审_002.mp3
└── 产品评审_003.mp3
```

---

## 📁 目录结构 / Directory Structure

```
Meeting/
├── README.md                               # 🇨🇳 本文档 / 🇬🇧 This file
├── CLAUDE.md                               # 🇨🇳 项目说明（Claude 读取）/ 🇬🇧 Project instructions (for Claude)
├── 目标.md                                 # 🇨🇳 原始需求 / 🇬🇧 Original requirements
├── .gitignore                              # Git 忽略规则 / Git ignore rules
├── audio/                                  # 🎵 音频文件（不上传 git）/ Audio files (git-ignored)
├── output/                                 # 📄 转录结果与纪要（不上传 git）/ Transcripts & minutes (git-ignored)
│   ├── 【完整讲话记录】*.md                 # 🇨🇳 完整对话 / 🇬🇧 Full transcript
│   └── 【会议纪要】*.md                     # 🇨🇳 会议纪要 / 🇬🇧 Meeting minutes
├── scripts/                                # 辅助脚本 / Helper scripts (optional)
├── input/                                  # 其他输入 / Other input (optional)
└── .claude/
    ├── skills/
    │   └── meeting-transcribe.md           # 🧠 Core Skill 定义 / Skill definition
    └── settings.local.json                 # 🇨🇳 本地配置（不上传 git）/ Local config (git-ignored)
```

---

## ⚙️ 技术栈 / Tech Stack

| 组件 / Component | 技术 / Technology | 🇨🇳 角色 / 🇬🇧 Role |
|------------------|-------------------|---------------------|
| 🧠 **AI 编排 / Orchestration** | Claude Code + `meeting-transcribe` Skill | 流程控制、说话人区分、纪要生成 / Pipeline orchestration, speaker diarization, minutes generation |
| 🎤 **语音转文字 / Speech-to-Text** | whisper.cpp + `large-v3-q5_0` | GPU 加速音频转录 / GPU-accelerated transcription |
| 🎛 **音频处理 / Audio Processing** | FFmpeg | 格式转换、重采样、分段 / Format conversion, resampling, segmentation |
| 🖥 **GPU 加速 / GPU Acceleration** | CUDA 13.2 + RTX 5090D | ~157x encode 加速 / encode speed-up |

---

## 🔒 隐私说明 / Privacy

| 🇨🇳 | 🇬🇧 |
|-----|-----|
| 所有音频处理在本地完成 | All audio processing is done locally |
| whisper.cpp 在本地 GPU 上运行 | whisper.cpp runs on your local GPU |
| 输出为本地 Markdown 文件 | Outputs are local Markdown files |
| `.gitignore` 已配置为不上传音频和输出内容 | `.gitignore` excludes audio & output for privacy |
| GitHub 仓库仅包含 Skill 定义和项目配置 | GitHub repo only contains skill definitions & project config |

---

## 🤝 协作维护 / Maintenance

🇨🇳 本项目使用 Git 管理版本。对项目进行优化时，Claude 会自动将更新同步到 GitHub 远端仓库。

🇬🇧 This project uses Git for version control. When improvements are made, Claude automatically syncs updates to the GitHub remote.

- **分支 / Branch**：`main` — 稳定版 / Stable
- **同步 / Sync**：修改 Skill 或项目文件后自动 commit + push / Auto commit & push after updates

---

## 📝 许可证 / License

MIT
