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
- **whisper.cpp**: 克隆到 `~/whisper.cpp`，**CUDA 版编译**并下载 `large-v3-turbo` 模型后量化为 `q8_0`
- **NVIDIA GPU + CUDA Toolkit**: RTX 5090D (32GB VRAM)，CUDA 13.2

详见 `.claude/skills/meeting-transcribe.md` 中的前置依赖检查部分。

## 目录结构

```
Meeting/
├── README.md                               # 语言选择入口页
├── README.zh-CN.md                         # 中文版文档
├── README.en.md                            # English documentation
├── CLAUDE.md                               # 本文件
├── audio/                                  # 音频文件（用户放置）和预处理中间文件
├── output/                                 # 输出目录
│   ├── 【完整讲话记录】*.md                 # 完整转录 + 说话人标注
│   └── 【会议纪要】*.md                     # 结构化会议纪要
├── scripts/                                # 辅助脚本（可选）
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

## 版本管理

本项目的 Skill 定义（`.claude/skills/meeting-transcribe.md`）和项目配置（`CLAUDE.md`、`README.md`、`.gitignore` 等）
使用 Git 管理并同步至 GitHub 仓库 [Deathleaves/Audio-to-Text-to-Notes](https://github.com/Deathleaves/Audio-to-Text-to-Notes)。

### 自动同步规则

当你在使用过程中对项目进行以下操作时，Claude 会自动 commit 并 push 到 GitHub：

1. **优化 Skill**（`.claude/skills/meeting-transcribe.md`）
   - 改进说话人区分逻辑
   - 优化会议纪要模板
   - 修复流程中的错误
   - 完善错误处理
   - 新增功能步骤

2. **更新项目配置**
   - 修改 `CLAUDE.md`
   - 修改 `README.md`
   - 修改 `.gitignore`
   - 修改 `目标.md`

3. **添加辅助脚本**（`scripts/` 目录下）
   - 新增自动化工具
   - 改进工作流程

### 不上传的内容

- `audio/` 目录下的音频文件（隐私数据）
- `output/` 目录下的转录结果和纪要（隐私数据）
- `.claude/settings.local.json`（可能含 token）
- `.env` 文件（密钥信息）

### 注意事项

- 每次 commit 前会检查是否有实质性变更（避免空提交）
- commit 信息会清晰说明变更内容
- 如需手动同步，使用标准 git 命令即可
