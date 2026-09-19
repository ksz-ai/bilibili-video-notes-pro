# Bilibili 视频结构化笔记（Pro 版）

> 把任意 Bilibili 视频 URL 转成可离线的单页 HTML 深度笔记，包含内容驱动的 SVG 图解与章节关键帧截图。

`bilibili-video-notes-pro` 是一个 TRAE 技能（Skill），输入一个 B 站视频地址，即可自动产出：

- **单页 HTML 笔记**：封面、章节、SVG 图解、关键帧全部 base64 内嵌，浏览器双击即可离线打开
- **章节关键帧截图**：ffmpeg 在每章首/中/末抽帧，按"亮度 + 方差"自动挑选最具代表性的一帧
- **内容驱动 SVG 图解**：根据章节内容特征自动选择流程图、时间轴、对比矩阵、金字塔分层等图型
- **纯文本时间戳**：时间点为文字提示，不带任何外部跳转链接，HTML 内不含 `<audio>`，杜绝误触播放

## 核心能力

1. **章节关键帧自动抽取**：每章抽 3 帧候选，按亮度中位 + 方差适中自动选代表帧
2. **SVG 视觉设计语言**：统一的卡片化、数据字号 ≥12px、深底浅字对比度规范
3. **关键帧 base64 直嵌 HTML**：无需任何外部资源，可离线查看

## 版本演进

| 版本 | 说明 |
|---|---|
| v1 / V5.0 | 完整管线：HTML + PDF + 切片 PNG + 关键帧 + SVG |
| v2 / V5.1 | 移除所有外部跳转链接，时间戳降级为纯文本（修复误触音频播放）|
| v3 / V5.2 | 不再生成 PDF，最终交付仅 HTML + 内嵌关键帧 |

## 快速开始

在 TRAE 中给一个 B 站视频 URL，并表达生成视频笔记的意图（如"生成视频笔记"、"要离线版视频笔记"、"带关键帧截图的笔记"）。

### 环境依赖

- `ffmpeg` / `ffprobe`
- Python3 + `pillow` / `numpy` / `faster-whisper`

```bash
pip install faster-whisper pillow numpy --break-system-packages
```

> 注意：v5.2 起不再需要 `weasyprint` / `playwright` / `wkhtmltoimage` 等渲染引擎。

## 处理流程

1. 抓取元数据 + 封面
2. 下载视频与音频双流（视频轨用于抽关键帧）
3. Whisper 转写（`faster-whisper`，中文，VAD 过滤）
4. 按自然结构拆章
5. 抽取章节关键帧（Pro 新增）
6. 内容驱动 SVG + 视觉设计语言
7. 生成离线单页 HTML

## 输出文件

- 最终交付：`notes_<BVID>_v5.html`（单页 HTML，可离线打开）
- 中间产物：`meta.json` / `cover.jpg` / `video_raw.mp4` / `audio.mp3` / `transcript.json` / `keys/<chid>.jpg` / `chapters.json`

## 设计规范摘录

SVG 深色主题（底 `#0b1220`）：

- 强调色 1：`#fde68a`（暖黄）、强调色 2：`#22d3ee`（青）、强调色 3：`#fb7185`（玫红）
- 主文字 `#f1f5f9`、次文字 `#cbd5e1`、辅助文字 `#94a3b8`（最低，禁止更深）
- 数据字号 ≥36px，描述字号 ≥12px

## 许可

本项目为个人学习用途。