# Meeting - 会议录音转文字与纪要生成

## 项目目的

将会议录音自动转为带说话人区分的完整文本，并生成结构化会议纪要。
全程本地处理，不上传任何音频到云端。

## 工作方式

当用户在本项目中发送音频文件（.mp3 / .m4a / .wav / .flac / .ogg 等）时，
自动触发 `meeting-transcribe` Skill，执行以下流程：

1. FFmpeg 自动预处理音频（格式检测、转码、按需分段）
2. 调用本地 whisper.cpp **GPU 加速**进行语音转文字（RTX 5090D, ~157x encode 加速）
3. Claude 分析文本，区分说话人
4. 输出「完整讲话记录」Markdown
5. 输出「会议纪要」Markdown

两个输出文件保存在 `output/` 目录下。

## 前置依赖（需用户提前配置）

- **FFmpeg**: `winget install Gyan.FFmpeg` (Windows) / `brew install ffmpeg` (Mac)
- **whisper.cpp**: 克隆到 `~/whisper.cpp`，**CUDA 版编译**并下载 `large-v3-q5_0` 模型
- **NVIDIA GPU + CUDA Toolkit**: RTX 5090D (32GB VRAM)，CUDA 13.2

详见 `.claude/skills/meeting-transcribe.md` 中的前置依赖检查部分。

## 目录结构

```
Meeting/
├── CLAUDE.md                               # 本文件
├── 目标.md                                 # 项目需求
├── audio/                                  # 音频文件（用户放置）和预处理中间文件
├── output/                                 # 输出目录
│   ├── 【完整讲话记录】*.md                 # 完整转录 + 说话人标注
│   └── 【会议纪要】*.md                     # 结构化会议纪要
└── .claude/
    └── skills/
        └── meeting-transcribe.md           # Skill 定义
```

## 使用方式

1. 将音频文件放入 `audio/` 目录（或发送文件路径给 Claude）
2. 告诉 Claude "处理这个音频" 
3. 等待 Skill 自动完成
4. 在 `output/` 目录查看结果

如果是同一场会议的多段录音，按时间顺序命名（如 `会议_001.mp3`、`会议_002.mp3`），
Skill 会自动识别并按顺序合并处理。
