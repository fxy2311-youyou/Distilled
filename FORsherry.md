# Distilled — 项目复盘笔记

> 写给未来的自己：这份笔记记录了 Distilled 这个项目是怎么建起来的、为什么这么设计、踩过哪些坑，以及值得带走的经验。

---

## 这个项目在做什么

每天早上 8 点，你的邮箱里会出现一封邮件。里面是一篇中文杂志文章——从 Lenny's Podcast、Lex Fridman、Y Combinator、Andrej Karpathy、Every 这五个频道，**加上全 YouTube 范围内的关键词搜索结果**中，自动筛出当前互动量最高、内容最相关的那一个视频，提取字幕，用 Gemini 2.5 Pro 写成一篇你能在 5 分钟内读完的深度好文。本地同时存有 PDF 和 Word，邮件里附着 EPUB，可以直接导入 Apple Books。

**一句话版本**：把每天涌出来的播客内容，自动蒸馏成一杯浓缩咖啡，送到你手边。

---

## 整体架构：一条流水线

```
YouTube API
  → 查询 5 个指定频道过去 48 小时的最新视频
  → 用 4 组关键词在全 YouTube 搜索相关视频
  → 合并去重，过滤掉 20 分钟以下的短视频
  → 用评分算法（互动量 + 关键词）选出最优的 1 条

youtube-transcript-api
  → 提取英文字幕，拼成完整文本

Gemini 2.5 Pro（思考模式）
  → 第一次调用：生成中文杂志文章（约 1500 字）
  → 第二次调用：生成 18 字以内的中文文件名
  → 第三次调用：生成 5 个板块的传播素材

文件生成
  → Playwright (Chromium) 把 HTML 渲染成 PDF，存本地
  → ebooklib 打包成 EPUB，用于邮件附件
  → python-docx 生成 Word 传播素材库，存本地

Gmail SMTP
  → 发送邮件：正文嵌入传播素材内容，附件是 EPUB

macOS launchd
  → 每天 08:00 自动触发整个流程
```

所有配置（API key、邮箱密码、输出目录）放在 `.env` 文件里，代码通过 `python-dotenv` 读取，不硬编码在脚本里。已处理过的视频 ID 存在 `processed.json`，防止同一个视频被处理两次。

---

## 每个模块是怎么工作的

### 1. 视频来源：双轨制

**轨道一：指定频道**
查询 5 个高质量频道过去 48 小时发布的视频：
- Lenny's Podcast / Andrej Karpathy / Lex Fridman / Y Combinator / Every

**轨道二：全站关键词搜索**
用 4 组查询词在整个 YouTube 搜索（按相关度排序）：
```python
SEARCH_QUERIES = [
    'vibe coding AI',
    'AI tools startup founder',
    'AI product management',
    'LLM AGI startup',
]
```

两轨结果合并去重后，统一进入评分流程。这样既不漏掉订阅频道的内容，也能捞到全站冒出来的高质量相关视频。

---

### 2. 视频评分：互动量优先

```
分数 = log10(点赞数 + 1) × 40     ← 主要权重（互动质量）
     + log10(播放量 + 1) × 10     ← 次要权重（传播广度）
     + 每匹配一个关键词 × 15       ← 内容相关性
```

**为什么用对数？**
如果直接用原始点赞数，Lex Fridman（10 万赞）会永远碾压小频道（500 赞），哪怕小频道的内容更相关。用对数压缩后：
- 10 万赞 → 约 200 分
- 500 赞 → 约 108 分

差距从 200:1 缩小到 2:1，内容相关性（关键词匹配最多 +180 分）就有机会帮高质量的小频道翻盘。

**为什么不考虑时效性？**
48 小时内的视频，发布时间差别不大，互动量已经能充分反映内容质量。时效性作为权重意义不大。

**关于"收藏量"**：YouTube 在 2012 年就把公开收藏功能下线了，现在能拿到的公开数据只有点赞和播放量，所以用这两个替代。

---

### 3. Gemini 生成文章

用 `google.genai` 新版 SDK，模型是 `gemini-2.5-pro`，自带思考能力。整个流程调用 Gemini 三次：

| 调用 | 输入 | 输出 |
|------|------|------|
| 文章生成 | 字幕（最多 8 万字符）| 完整中文杂志文章 |
| 文件名 | 文章前 600 字 | 18 字以内中文概括 |
| 传播素材 | 文章前 8000 字 | 5 个板块的结构化内容 |

Prompt 写得很细：6 个步骤的写作指令，指定语气（中文杂志腔，不是翻译腔）、格式（# 标题、**引言段**、--- 分节符）、内容标准（只提"宝石级洞见"，不要励志废话）。**Prompt 质量直接决定文章质量。**

---

### 4. PDF 封面 + 正文

PDF 由 Playwright 驱动无头 Chromium 渲染。第一页是视频封面图（base64 嵌入，深色背景 + 半透明缩略图 + 渐变文字），翻页后是正文内容。

关键细节：不能用 `page.goto(file://...)` 打开 HTML 文件，必须用 `page.set_content()` 直接注入内容，并且在注入前把 CSS 里的 Google Fonts `@import` 删掉——否则无头浏览器会因为加载外部字体超时（30 秒）导致整个任务卡死。

---

### 5. 邮件结构

邮件**正文**包含：
- 文章标题 + 频道 + 原视频链接
- 5 个传播素材板块（金句 / 故事 / 洞见 / 标题模板 / 一句话精华）

**附件**只有 EPUB 一个。Word 文件不再通过邮件发送，只保存到本地。

传播素材由 Gemini 生成，原始格式是 `===金句===` `===故事===` 这样的区块结构，`word_content_to_html()` 函数负责解析并转换成带样式的 HTML 嵌入邮件正文。

---

## 为什么这么选技术

| 技术 | 为什么选它 |
|------|-----------|
| **Gemini 2.5 Pro** | 思考模型，处理长字幕时质量明显更好；同属 Google 生态，跟 YouTube API 协同顺畅 |
| **Playwright** | 能精准控制 Chrome 渲染，生成的 PDF 跟浏览器看到的完全一致，比纯 Python 方案（WeasyPrint 等）好看得多 |
| **ebooklib** | Python 里生成 EPUB 的事实标准 |
| **python-docx** | 操控 Word 格式的唯一靠谱选择 |
| **launchd** | macOS 原生定时任务，不需要额外安装任何东西 |
| **.env + dotenv** | 密钥和配置分离，上 GitHub 只需排除 `.env` 即可 |

---

## 踩过的坑（最重要的部分）

### 坑 1：youtube-transcript-api 的 API 改了

旧的用法：
```python
YouTubeTranscriptApi.list_transcripts(video_id)  # ❌ 不行了
```
新的用法：
```python
api = YouTubeTranscriptApi()
t = api.fetch(video_id, languages=['en'])  # ✅
```
**教训**：用第三方库之前先查 changelog，尤其是有 "deprecation" 字样的。

---

### 坑 2：Google Gemini API 的"免费额度"是假的

在 Google Cloud Console 创建的 API key，对于 `gemini-2.5-pro` 这类新模型，免费额度是 **0**——不是"有限额度"，是字面意义上的零。必须先开启计费（billing）才能使用。

我们先后换了三个 key 才搞明白这个问题。

**教训**：Google 的"免费"经常带引号，新模型通常没有免费层。

---

### 坑 3：Playwright 渲染 PDF 时 30 秒超时

根本原因：HTML 里有 `@import url('https://fonts.googleapis.com/...')` 这行 CSS，无头浏览器会去请求外部字体，网络慢时超时卡死。

解法：
```python
# 删掉 Google Fonts import
html_pdf = re.sub(r"@import url\('https://fonts\.googleapis\.com[^']*'\);", '', html_content)
# 直接注入内容，不通过 file:// URL
page.set_content(html_pdf, wait_until='domcontentloaded')
```

**教训**：凡是要离线渲染 HTML 的场景，所有外部资源（字体、图片、CDN）必须替换成本地或 base64 内联。

---

### 坑 4：ebooklib 的"Document is empty"错误

`ch.content = xhtml_string` 不行，要显式编码：
```python
ch.set_content(xhtml.encode('utf-8'))  # ✅
```
**教训**：ebooklib 内部期望字节（bytes），不是字符串（str）。

---

### 坑 5：EPUB 封面图片重复

`book.set_cover()` 内部已经把图片加进书里了，不能再手动 `book.add_item()` 一次，否则出现 "Duplicate name" 警告。

**教训**：用库提供的高级方法时，不要同时手动做同一件事。

---

### 坑 6：Gemini 模型名字对不上

`gemini-2.5-pro-exp-03-25` 这个名字在某个时间点失效了。解法：直接调 `client.models.list()` 列出当前所有可用模型名，找正确的用。

**教训**：优先用不带日期的稳定别名（如 `gemini-2.5-pro`），带日期的实验性名字随时会消失。

---

## 代码里还有几个已知问题（还没修）

1. **`from google.genai import types` 是孤儿 import** — 当初为 ThinkingConfig 准备的，后来发现不需要，忘删了。
2. **`get_transcript()` 没有错误处理** — 如果视频没有英文字幕，整个流程会崩。应加 try/except，出错时跳过当前视频选下一个候选。
3. **`main()` 里步骤 7 出现两次** — 纯编号错误，不影响运行。

---

## 值得带走的思维方式

**1. 先手动，再自动化**
整个项目先手动跑通了第一个视频的全流程，确认文章质量满意之后才搭自动化逻辑。避免花几小时建好框架，结果发现输出内容根本不好用。

**2. 每个环节都要有 log**
`log.info()` 贯穿全流程，每步做了什么、选了哪个视频、得了多少分都有记录。出问题打开 `distilled.log` 就能定位到是哪一步崩的。

**3. 把"变化的"和"不变的"分开**
- 变化的：API key、邮箱、输出目录 → 放 `.env`
- 稳定的：业务逻辑、Prompt、HTML 模板 → 放代码

以后换 API key、换邮箱，只改 `.env` 一个文件，不动代码。

**4. Prompt 是产品核心**
大约 40% 的时间花在调 Prompt 上。模型调用只是一行代码，但 Prompt 的质量决定了输出内容是否值得读。好的 Prompt 需要：明确角色、明确格式、明确什么要、什么不要。

**5. 外部依赖越少越好**
每多一个外部服务，就多一个可能出故障或改价格的地方。能合并的就合并（Google 生态 = YouTube + Gemini，少一个账号）。

---

## 文件结构

```
~/Desktop/Distilled/
├── distilled.py          # 主脚本（全部逻辑在这一个文件里）
├── .env                  # 所有密钥（绝对不能上 GitHub）
├── RULES.md              # 项目规则文档
├── FORsherry.md          # 这份复盘笔记
├── processed.json        # 已处理视频 ID（不需要上 GitHub）
├── distilled.log         # 运行日志（自动生成）
└── {文章标题}_{日期}.pdf/.docx   # 每天生成的文章

~/Library/LaunchAgents/
└── com.distilled.daily.plist    # macOS 定时任务配置
```

---

## 下一步

- [ ] 修复 3 个已知代码问题
- [ ] 创建 `.gitignore`，把代码推到 GitHub
- [ ] 配置 GitHub Actions（让脚本在云端运行，Mac 不需要开着）
- [ ] 给 `get_transcript()` 加错误处理和 fallback 逻辑
- [ ] Mac 定时唤醒配置（系统设置 → 电池 → 计划，07:55 唤醒）
