# Meeting — Audio Transcription & Meeting Minutes Generation

> **100% Local Processing**: Audio never leaves your machine. Powered by whisper.cpp GPU acceleration + Claude AI analysis.

---

## 📋 Preview

Sample output from a real meeting recording (Appointment meeting, 2026-06-06):

### Full Transcript

Verbatim transcript with timestamps and speaker labels:

```
[00:00:05] **Speaker A（Host）**：今天我们来讨论一下下一版本的功能规划...
[00:00:30] **Speaker B**：我觉得优先级的排序应该以用户反馈为准...
[00:02:15] **Speaker A（Host）**：同意这个方向，那下面请 Speaker C 介绍一下调研结果。
```

> Note: Speaker diarization works on Chinese audio by default; the conversation content appears in the original language.

### Meeting Minutes

Structured summary:

```markdown
## Topic
（Summary of the core topic）

## Discussion Points
### 1. Topic 1
- Key points...
### 2. Topic 2
- ...

## Decisions & Conclusions
1. Clear consensus...

## Action Items
- [ ] Specific task @Assignee
```

---

## ✨ Features

| Feature | Description |
|---------|-------------|
| 🎯 **Speaker Diarization** | Claude identifies and labels different speakers from conversation context |
| 🚀 **GPU-Accelerated** | RTX 5090D: 30min audio → ~30-60s (~157x encode speed-up) |
| 🔒 **100% Local** | Audio never leaves your machine — no cloud upload required |
| 🧩 **Multi-part Merge** | Automatically detects, sorts, and merges multiple recordings of the same meeting |
| 📂 **Structured Minutes** | Auto-generates meeting minutes with action items and assignees |
| 🗣 **Multi-language** | Handles mixed Chinese-English speech effectively (large-v3 model) |

---

## 📦 Prerequisites

Install these tools locally beforehand:

### 1️⃣ FFmpeg (Audio Processing)

```bash
# Windows
winget install Gyan.FFmpeg

# macOS
brew install ffmpeg
```

### 2️⃣ whisper.cpp (Speech-to-Text, GPU version)

```bash
# Clone repository
git clone https://github.com/ggerganov/whisper.cpp.git ~/whisper.cpp
cd ~/whisper.cpp

# Build with CUDA support (requires CUDA Toolkit + VS Build Tools)
cmake -B build -DCMAKE_BUILD_TYPE=Release -DGGML_CUDA=ON
cmake --build build --config Release --parallel

# Download Chinese-optimized model
bash ./models/download-ggml-model.sh large-v3-q5_0
```

> Mirror for slow HuggingFace access:
> ```bash
> curl -L -o ~/whisper.cpp/models/ggml-large-v3-q5_0.bin \
>   "https://hf-mirror.com/ggerganov/whisper.cpp/resolve/main/ggml-large-v3-q5_0.bin"
> ```

### 3️⃣ NVIDIA GPU + CUDA Toolkit

| GPU | Speed-up |
|-----|----------|
| RTX 5090D (32GB VRAM) | **~157x** encode speed-up |
| Other NVIDIA GPUs | Scales with VRAM & compute capability |

---

## 🚀 Quick Start

### Option 1: Place files in audio/

```
1. Place .mp3/.wav/.m4a files in the audio/ directory
2. Tell Claude → "transcribe this audio"
3. Wait for completion
4. Check output/ for results
```

### Option 2: Send file path directly

```
1. Send Claude the audio file path
2. Claude auto-triggers the skill
3. Done!
```

### Multi-part Recordings

Name sequentially for auto-merge:

```
audio/
├── product_review_001.mp3     ← Same meeting, auto-merged
├── product_review_002.mp3
└── product_review_003.mp3
```

---

## 📁 Directory Structure

```
Meeting/
├── README.md                               # Language selection landing page
├── README.zh-CN.md                         # 中文版文档
├── README.en.md                            # English documentation
├── CLAUDE.md                               # Project instructions (for Claude Code)
├── .gitignore                              # Git ignore rules
├── audio/                                  # 🎵 Audio files (git-ignored)
├── output/                                 # 📄 Transcripts & minutes (git-ignored)
│   ├── 【完整讲话记录】*.md                 # Full transcript with speaker labels
│   └── 【会议纪要】*.md                     # Structured meeting minutes
├── scripts/                                # Helper scripts (optional)
└── .claude/
    ├── skills/
    │   └── meeting-transcribe.md           # 🧠 Core Skill definition
    └── settings.local.json                 # Local config (git-ignored)
```

---

## ⚙️ Tech Stack

| Component | Technology | Role |
|-----------|------------|------|
| 🧠 **AI Orchestration** | Claude Code + `meeting-transcribe` Skill | Pipeline control, speaker diarization, minutes generation |
| 🎤 **Speech-to-Text** | whisper.cpp + `large-v3-q5_0` | GPU-accelerated transcription |
| 🎛 **Audio Processing** | FFmpeg | Format conversion, resampling, segmentation |
| 🖥 **GPU Acceleration** | CUDA 13.2 + RTX 5090D | ~157x encode speed-up |

---

## 🔒 Privacy

- **All audio processing is done locally** — whisper.cpp runs on your local GPU
- Original audio can be deleted anytime after transcription
- Outputs are local Markdown files — nothing is uploaded automatically
- `.gitignore` is configured to exclude audio files and output content
- The GitHub repository only contains skill definitions and project configuration

---

## 🤝 Maintenance

This project uses Git for version control. When improvements are made during use, Claude automatically syncs updates to the GitHub remote.

- **Branch**: `main` — stable version
- **Sync**: Auto commit & push after skill or project file updates

---

## 📝 License

MIT
