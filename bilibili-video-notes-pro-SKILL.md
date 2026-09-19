---
name: "bilibili-video-notes-pro"
description: "Turns any Bilibili video URL into a polished offline single-page HTML note with content-driven SVG diagrams and per-chapter keyframe screenshots. Enhanced version with keyframe extraction, SVG visual design rules, and pure-text timestamps (no clickable jump links, no PDF output). Invoke when user gives a Bilibili URL and wants a deep, visually-rich offline HTML study note."
---

# Bilibili 视频结构化笔记（Pro 版）

把任意 Bilibili 视频 URL 转成"可离线的单页 HTML 深度笔记"，最终交付物是**单页 HTML + 关键帧截图（base64 内嵌，可离线打开）**。**不生成 PDF，不带任何跳转链接**——时间戳仅为文本形式的时间点提示。

> **历史变更**：
> - v1（V5.0）：完整管线（HTML + 关键帧 + SVG + 三级渲染回退 + 切片 PNG）
> - v2（V5.1）：**取消所有外部跳转链接**（用户反馈 HTML 中点击会触发音频播放，且多次点击会重叠播放），时间戳降级为纯文本装饰
> - v3（V5.2）：**不再生成 PDF**（用户不需要），最终交付只有 HTML + 关键帧截图（base64 内嵌）

在 `bilibili-video-notes` 基础上，新增三个核心能力：

1. **章节关键帧自动抽取**：用 ffmpeg 在每章首/中/末抽 3 帧候选，按"亮度 + 方差"自动选最代表帧，嵌到章节内
2. **SVG 视觉设计语言**：统一的卡片化、数据字号≥12、深底浅字对比度规则、画布高度防错位规范
3. **关键帧预览图直接 base64 进 HTML**：浏览器可直接打开 HTML，无需任何外部资源

## 触发场景

用户给 B 站视频 URL（`https://www.bilibili.com/video/<BVID>/?...`）并要求：
- "生成视频笔记" / "生成 HTML 笔记"
- "我要离线版的视频笔记" / "结构化总结"
- "带关键帧截图的笔记"

**不适用场景**：
- 用户明确要 PDF（此时回退到基础版 skill 或用 weasyprint 自行二次渲染）
- 用户希望在 HTML 中点击时间戳跳转视频（v5.1 已移除该功能，请先告知用户）

## 完整流程（8 步）

### 步骤 0：语言与工具自检

1. 语言 = 用户输入语言（全流程一致）
2. 检查工具：`ffmpeg` / `ffprobe` / Python3 / `pillow` / `numpy` / `faster-whisper`。缺则装：`pip install <pkg> --break-system-packages`
3. **v5.2 起不再需要** `weasyprint` / `pdftoppm` / `wkhtmltoimage` / `playwright` 等渲染引擎（HTML 直接交付）

### 步骤 1：抓元数据 + 封面

B站官方 Web API + curl（带 UA + Referer）：

```bash
UA="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"
BVID="<从URL解析>"
curl -s -H "User-Agent: $UA" -H "Referer: https://www.bilibili.com/video/${BVID}/" \
  "https://api.bilibili.com/x/web-interface/view?bvid=${BVID}" -o meta.json
curl -s -H "User-Agent: $UA" -H "Referer: https://www.bilibili.com/video/${BVID}/" \
  "https://api.bilibili.com/x/tag/archive/tags?aid=<AID>" -o tags.json
curl -s -o cover.jpg "<pic URL>"
```

### 步骤 2：下载视频 + 音频

**同时下视频和音频两条流**（Pro 版区别：需要视频轨来抽关键帧）：

```bash
# 取音视频最佳流
curl -s "https://api.bilibili.com/x/player/playurl?bvid=${BVID}&cid=${CID}&qn=64&fnval=4048&fnver=0&fourk=1"
# 选 max(bandwidth) 的 video[] 和 audio[]

# ffmpeg 下载（自带 UA + Referer 防 SSL 报错）
ffmpeg -y -user_agent "$UA" -headers "Referer: https://www.bilibili.com/video/${BVID}/\r\n" \
  -i "$video_url" -c copy video_raw.mp4 -loglevel error
ffmpeg -y -user_agent "$UA" -headers "Referer: https://www.bilibili.com/video/${BVID}/\r\n" \
  -i "$audio_url" -vn -acodec copy audio_raw.m4a -loglevel error
ffmpeg -y -i audio_raw.m4a -vn -ar 44100 -ac 2 -b:a 192k audio.mp3 -loglevel error
```

时长校验：`ffprobe` vs 元数据 `duration`，差 >5% 标记异常。

> 音频文件**不删除**，但**不链接进 HTML**——HTML 里没有任何 `<audio>` / `<source>` / 音频 src 引用，避免点击触发播放。

### 步骤 3：Whisper 转写

`faster-whisper` + `language="zh"` + `vad_filter=True` + `compute_type="int8"` + `beam_size=5`。末段时间差 <10% 通过。

模型下载失败：用 `https://hf-mirror.com` 镜像或复用本地缓存目录。

### 步骤 4：按自然结构拆章

每章包括：
- 时间范围（纯文本展示，不带链接）
- 核心问题、关键步骤、关键引用
- **关键帧时间锚点**（章首/中/末 3 个时间点）

### 步骤 5：抽章节关键帧（Pro 新增）

```python
import subprocess, numpy as np
from PIL import Image
from pathlib import Path

def pick_representative(candidates):
    """从候选帧按"亮度中位 + 方差适中"自动选代表帧。
    评分：-(abs(mean-110) + abs(std-65))"""
    best, bs = None, None
    for p in candidates:
        arr = np.array(Image.open(p).convert('L'))
        m, sd = float(arr.mean()), float(arr.std())
        score = -(abs(m-110) + abs(sd-65))
        if best is None or score > bs:
            best, bs = p, score
    return best

# 关键帧文件存到 keys/ 目录
Path('keys').mkdir(exist_ok=True)
chs = json.load(open('chapters.json'))
for c in chs:
    cid, s, e = c['id'], c['start'], c['end']
    # 末章 ch8 跳过末尾黑屏候选
    last_offset = e - 2 if cid != 'ch8' else e - 4
    mid = (s + e) // 2
    cands = []
    for t in [s + 2, mid, last_offset]:
        if t < 0 or t >= e:
            t = s + (e - s) // 3
        out = f'keys/{cid}_t{t}.jpg'
        subprocess.run([
            'ffmpeg', '-y', '-ss', str(t), '-i', 'video_raw.mp4',
            '-frames:v', '1', '-q:v', '3', '-vf', 'scale=600:-1',
            out, '-loglevel', 'error'
        ], check=True)
        cands.append(out)
    best = pick_representative(cands)
    if best:
        Path(f'keys/{cid}.jpg').write_bytes(Path(best).read_bytes())
```

### 步骤 6：内容驱动 SVG + 视觉设计语言（核心）

**图型选择**：

| 内容特征 | 推荐图型 |
|---|---|
| 流程/步骤 | 流程链（3-5 个圆角矩形 + 箭头）|
| 概念/分层 | 分层图（倒三角金字塔）|
| 时间变化 | 时间轴（横线 + 节点 + 标注）|
| 对比 | 对比矩阵（左右两栏卡片 + 中间对比元素）|
| 数据/数字 | 大数字 + 进度条 |
| 因果 | 因果链路 |
| 关系/中心放射 | 中心圆 + 放射线 + 外置标签 |

**SVG 视觉设计语言规则**（V5 实战验证）：

```
1. 画布高度：
   - 单章简单图：380-420px
   - 双卡片对比：460-480px
   - 三层结构（对比矩阵 + 节点扩展）：460-480px
   - 复合图（5 维放射）：460-480px
2. 配色（深色主题 #0b1220 底）：
   - 强调色 1：#fde68a（暖黄）
   - 强调色 2：#22d3ee（青）
   - 强调色 3：#fb7185（玫红）
   - 主文字：#f1f5f9
   - 次文字：#cbd5e1（必用）
   - 辅助文字：#94a3b8（最小）
   - 描边：#334155
   - **禁止**：#64748b 以下亮度的文字（深底上看不清）
3. 字号：数据字号 ≥36px（字重 900），描述字号 ≥12px
4. 箭头 marker：14×14
5. 卡片：圆角 rx=12~18，stroke 1.5~2px
6. 留白：每元素与边缘 ≥10px
7. 渐变：linearGradient 仅作强调（数字、胶囊）
8. 标签防重叠：放射图标签外置到外圈
```

**自动选型模板库**（可选增强）：

```python
TEMPLATE_KEYWORDS = {
    'timeline':       ['时间','历程','数字','顺序','节点'],
    'compare_matrix': ['对比','比较','vs','角度','差异','向量','直线','拆分'],
    'concept_map':    ['关系','并列','切开','手段','目的','结构'],
    'split_dual':     ['拆分','双','分开','对比','自由'],
    'layered_3':      ['分类','三层','等级','三种','三类'],
    'radial_5':       ['五点','五','维度','五维'],
    'flow_chain':     ['链条','递进','流程','顺序','三段'],
}
```

**Python 数据陷阱**：
- f-string 内不嵌套花括号表达式（如 `{a-{b//2}}` 崩语法），先算中间变量
- 长列表 / 字典配对检查括号
- SVG 标签必须在 `<svg>...</svg>` 内，不要溢出到字符串外

### 步骤 7：生成单页 HTML（v5.2：无任何跳转链接、无 `<audio>` 标签）

**关键约束（v5.1/v5.2 新增）**：

| 元素 | 状态 | 写法 |
|---|---|---|
| `▶ 在 B 站打开原视频` 按钮 | **删除** | — |
| 章节目录小标题 | 改成"时间点提示" | `<h3>章节目录 · 时间点提示</h3>` |
| 章节右上角时间范围 | 改为纯文本 `<span>` | `<span class="ch-jump">⏱ 00:00–00:15</span>` |
| 关键帧图片外层 | 改为 `<div>` 容器，不再 `<a>` 包裹 | `<div class="ch-key">…</div>` |
| 关键帧图右下角 caption | 去掉"· 点击跳转" | `<span class="cap">关键帧 @ 00:00</span>` |
| 引文里的时间标签 | 改为纯文本 `<span>` | `<span class="qt-time">00:08</span>` |
| 章节目录副链接 | 改为 `<span class="t-ts">` | `<span class="t-ts">00:00–00:15</span>` |
| 任何 `<a href="...bilibili.com...">` | **禁止** | — |
| 任何 `<audio>` / `<source>` / 音频 src | **禁止** | — |

**完整 HTML 结构**：

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <style>
    :root {
      --bg:#0b1220; --panel:#0f172a; --panel2:#1e293b;
      --border:#334155; --fg:#f1f5f9; --fg2:#cbd5e1; --fg3:#94a3b8;
      --accent:#fde68a; --accent2:#22d3ee; --accent3:#fb7185;
      --grad: linear-gradient(90deg,#fde68a,#22d3ee);
    }
    body { background:var(--bg); color:var(--fg); font-family:"PingFang SC","Microsoft Yahei",Arial,sans-serif; }
    .ch-jump, .toc .t-ts, .qt-time, .ch-key .cap { cursor: default; }  /* 明确不可点击 */
    .ch-svg { background:#0b1220; border-radius:14px; padding:10px; border:1px solid var(--border); }
    .ch-svg svg { display:block; max-width:100%; height:auto; }
    .ch-key { position:relative; border-radius:14px; overflow:hidden; border:1px solid var(--border); background:var(--panel2); }
    .ch-key img { display:block; width:100%; height:auto; }
    .ch-key .cap { position:absolute; bottom:8px; left:8px; right:8px; background:rgba(11,18,32,0.85); padding:6px 10px; border-radius:8px; text-align:center; }
    section.chapter { break-inside: avoid; }
  </style>
</head>
<body>
  <header class="hero">
    <img class="cover" src="data:image/jpeg;base64,..." />
    <h1 class="title">…</h1>
    <div class="tags">…</div>
    <!-- 注意：没有 ▶ 在 B 站打开原视频 按钮 -->
  </header>

  <section class="toc">
    <h3>章节目录 · 时间点提示</h3>
    <ol>
      <li><a href="#ch1">…</a><span class="t-ts">00:00–00:15</span></li>
      <!-- 章节目录里的 #ch1 锚链接是同页内跳转，不算外部链接，可以保留 -->
      …
    </ol>
  </section>

  <article class="chapter" id="ch1">
    <header class="ch-head">
      <span class="ch-id">CH1</span>
      <h2>章节标题</h2>
      <span class="ch-jump">⏱ 00:00–00:15</span>  <!-- 纯文本时间戳，不是链接 -->
    </header>

    <div class="ch-grid">
      <div class="ch-svg"><!-- SVG inline --></div>
      <div class="ch-key">                              <!-- 不是 <a>，不可点击 -->
        <img src="data:image/jpeg;base64,..." />
        <span class="cap">关键帧 @ 00:00</span>
      </div>
    </div>

    <ul class="pts">…</ul>

    <blockquote class="quote">
      <span class="qt-time">00:08</span>                <!-- 纯文本，不是链接 -->
      <span class="qt-body">"</span>
    </blockquote>
  </article>
</body>
</html>
```

**检验脚本**（必跑）：

```bash
# 任何残留都要报警
test $(grep -c 'href="https://www.bilibili.com' note.html) -eq 0 && echo "✓ no bilibili links"
test $(grep -c '<audio' note.html) -eq 0 && echo "✓ no audio tag"
test $(grep -c '点击跳转' note.html) -eq 0 && echo "✓ no click-jump text"
```

## 环境经验（V5 实战补充）

| 问题 | 解决 |
|---|---|
| wkhtmltoimage 安装失败 | v5.2 已不需；早期版本用 weasyprint 优先 |
| weasyprint 中文 | 补中文字体到系统 |
| ffmpeg 抽帧偶发黑屏 | 抽 3 帧取代表（亮度+方差评分） |
| SVG 文字过浅不可见 | 强制最低 #94a3b8 |
| 金字塔底部被流程条压住 | 画布高度 +60~80px 或拆 SVG 外 |
| ffmpeg 拉流 SSL 错 | ffmpeg 自带 UA + Referer header |
| HuggingFace 直连失败 | hf-mirror 镜像 |
| **HTML 点击时间戳触发音频播放** | **v5.1 已修复：移除所有 `<a href>` 与 `<audio>`，时间戳降级为 `<span>`** |

## 完成校验清单（v5.2）

- [ ] 元数据齐全（标题/UP/BV/时长/封面/标签）
- [ ] ffprobe vs 元数据时长差 <5%
- [ ] 转写末段时间匹配
- [ ] 每章 SVG 类型贴合内容
- [ ] 每章 ≥1 张关键帧 base64 内嵌
- [ ] SVG 内文字最低亮度 #94a3b8
- [ ] SVG 画布无元素溢出或重叠
- [ ] **HTML 中 bilibili.com 外部链接数 = 0**
- [ ] **HTML 中 `<audio>` / 音频 src 引用数 = 0**
- [ ] **HTML 中"点击跳转"文案数 = 0**
- [ ] **时间戳全部为 `<span>` 文本，不是 `<a>` 链接**
- [ ] **无 PDF 生成动作（不调用 weasyprint / playwright / wkhtmltoimage）**
- [ ] 交付物已放 `<workspace>/`：`notes_<BVID>_v5.html`

## 输出文件命名

- 中间产物（工作目录）：`meta.json` / `cover.jpg` / `video_raw.mp4` / `audio.mp3` / `transcript.json` / `keys/<chid>.jpg` / `chapters.json`
- 最终交付（`<workspace>/`）：
  - **`notes_<BVID>_v5.html`** — 单页 HTML（封面 + 章节 + SVG + 关键帧全部 base64 内嵌，可离线打开）
  - 注意：**不再生成 PDF，不再生成切片 PNG**

## 已知差异 vs 旧版

| 维度 | 旧版（v1/V5.0）| 新版（v3/V5.2）|
|---|---|---|
| 主要交付 | HTML + PDF + 切片 PNG | **仅 HTML** |
| 时间戳 | `<a href="…?t=N">`（点击跳 B 站）| `<span>` 纯文本 |
| 章节目录 | 同上 | `#ch1` 锚链接（同页内，正常）|
| 关键帧图 | 点击触发跳转 | **不可点击**，仅展示 |
| 音频 | 在 HTML 外供下载 | 在 HTML 外存在；**HTML 内不引用任何音频** |
| 渲染引擎 | weasyprint/playwright/wkhtmltoimage | **不需要**（HTML 自带） |

如果用户在某些场景重新想要 PDF，请告知后单独用 `weasyprint notes_*.html notes_*.pdf` 渲染即可，pipeline 不变。
