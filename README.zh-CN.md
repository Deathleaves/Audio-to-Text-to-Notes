# Meeting — 会议录音转文字 & 纪要生成

> **全程本地运行**：音频不上传云端，基于 whisper.cpp GPU 加速 + Claude 智能分析。

---

## 📋 效果预览

以下是对一次实际会议录音的处理结果示例（Appointment 会议，2026-06-06）：

### 完整讲话记录

带时间戳和说话人区分的逐字稿：

```
[00:00:05] **Speaker A（主持人）**：今天我们来讨论一下下一版本的功能规划...
[00:00:30] **Speaker B**：我觉得优先级的排序应该以用户反馈为准...
[00:02:15] **Speaker A（主持人）**：同意这个方向，那下面请 Speaker C 介绍一下调研结果。
```

### 会议纪要

结构化摘要：

```markdown
## 会议主题
（核心议题概述）

## 讨论要点
### 1. 话题一
- 主要观点...
### 2. 话题二
- ...

## 决议与结论
1. 明确的共识...

## 待办事项
- [ ] 具体任务 @负责人
```

---

## ✨ 功能特性

| 特性 | 说明 |
|------|------|
| 🎯 **自动说话人区分** | Claude 基于对话上下文区分不同说话人，并推断角色（如主持人、产品经理） |
| 🚀 **GPU 极速转录** | RTX 5090D 本地加速，30 分钟音频约 30-60 秒完成（~157x encode 加速） |
| 🔒 **完全本地处理** | 音频不离开你的电脑，无需联网上传任何数据 |
| 🧩 **多段合并** | 同一会议的多段录音自动识别、排序、合并处理 |
| 📂 **结构化纪要** | 自动生成包含待办事项和负责人标记的会议纪要 |
| 🗣 **多语言支持** | 中英混合场景也能较好处理（large-v3 模型） |

---

## 📦 前置依赖

需要你提前在本地准备好以下工具：

### 1️⃣ FFmpeg（音频处理）

```bash
# Windows
winget install Gyan.FFmpeg

# macOS
brew install ffmpeg
```

### 2️⃣ whisper.cpp（语音转文字，GPU 版）

```bash
# 克隆仓库
git clone https://github.com/ggerganov/whisper.cpp.git ~/whisper.cpp
cd ~/whisper.cpp

# CUDA 版编译（需安装 CUDA Toolkit + Visual Studio Build Tools）
cmake -B build -DCMAKE_BUILD_TYPE=Release -DGGML_CUDA=ON
cmake --build build --config Release --parallel

# 下载中文优化模型
bash ./models/download-ggml-model.sh large-v3-q5_0
```

> 如果 HuggingFace 访问慢，可用镜像：
> ```bash
> curl -L -o ~/whisper.cpp/models/ggml-large-v3-q5_0.bin \
>   "https://hf-mirror.com/ggerganov/whisper.cpp/resolve/main/ggml-large-v3-q5_0.bin"
> ```

### 3️⃣ NVIDIA GPU + CUDA Toolkit

| GPU | 加速比 |
|-----|--------|
| RTX 5090D (32GB VRAM) | **~157x** encode 加速 |
| 其他 NVIDIA GPU | 按显存和算力递减 |

---

## 🚀 快速使用

### 方式一：放入 audio/ 目录

```
1. 把 .mp3/.wav/.m4a 文件放入 audio/ 目录
2. 告诉 Claude → "处理这个音频"
3. 等待自动完成
4. 在 output/ 查看结果
```

### 方式二：直接发送文件路径

```
1. 发给 Claude 音频文件的路径
2. Claude 会自动触发 Skill
3. 完成！
```

### 多段同会议录音

按时间顺序命名即可自动合并：

```
audio/
├── 产品评审_001.mp3     ← 自动识别为同一会议，按序号合并
├── 产品评审_002.mp3
└── 产品评审_003.mp3
```

---

## 📁 目录结构

```
Meeting/
├── README.md                               # 语言选择入口页
├── README.zh-CN.md                         # 中文版文档
├── README.en.md                            # English documentation
├── CLAUDE.md                               # 项目说明（Claude 读取）
├── .gitignore                              # Git 忽略规则
├── audio/                                  # 🎵 音频文件（不上传 git）
├── output/                                 # 📄 转录结果与纪要（不上传 git）
│   ├── 【完整讲话记录】*.md                 # 完整对话 + 说话人标注
│   └── 【会议纪要】*.md                     # 结构化会议纪要
├── scripts/                                # 辅助脚本（可选）
└── .claude/
    ├── skills/
    │   └── meeting-transcribe.md           # 🧠 Core Skill 定义
    └── settings.local.json                 # 本地配置（不上传 git）
```

---

## ⚙️ 技术栈

| 组件 | 技术 | 角色 |
|------|------|------|
| 🧠 **AI 编排** | Claude Code + `meeting-transcribe` Skill | 流程控制、说话人区分、纪要生成 |
| 🎤 **语音转文字** | whisper.cpp + `large-v3-q5_0` | GPU 加速音频转录 |
| 🎛 **音频处理** | FFmpeg | 格式转换、重采样、分段 |
| 🖥 **GPU 加速** | CUDA 13.2 + RTX 5090D | ~157x encode 加速 |

---

## 🔒 隐私说明

- **所有音频处理在本地完成**，whisper.cpp 在本地 GPU 上运行
- 转录完成后，原始音频可随时删除
- 输出为本地 Markdown 文件，不会自动上传任何内容
- 本项目 `.gitignore` 已配置为不上传音频文件和输出内容
- GitHub 仓库仅包含 Skill 定义和项目配置

---

## 🤝 协作维护

本项目使用 Git 管理版本。当你在使用过程中对项目进行优化时，Claude 会自动将更新同步到 GitHub 远端仓库。

- **分支**：`main` — 稳定版本
- **同步**：修改 Skill 或项目文件后自动 commit + push

---

## 📝 许可证

MIT
