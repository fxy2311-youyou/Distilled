# Distilled — 完整项目复盘

> 写给未来的自己：这是一份完整的项目交接文档，记录了 Distilled 从零到运行的全过程、每一个技术决策背后的原因、踩过的每一个坑，以及值得带走的经验。

---

## 这个项目在做什么

每天早上 8 点，你的邮箱里自动出现一封邮件。里面是一篇约 1500 字的中文深度文章——从 5 个精选频道加上全 YouTube 关键词搜索结果中，自动筛出互动量最高、内容最相关的一个视频，提取字幕，用 Gemini 2.5 Pro 写成一篇 5 分钟能读完的好文。邮件里还有金句、故事、评论精选，附件是 EPUB，可以直接导入 Apple Books。本地同时保存 PDF（带视频封面）。

**一句话版本**：把每天最值得看的 AI / 创业长视频，自动蒸馏成一杯浓缩咖啡，送到你手边。

---

## 整体架构：一条流水线

```
YouTube API
  → 扫描 5 个指定频道（最近 30 天的视频）
  → 用 4 组关键词在全 YouTube 搜索相关视频
  → 合并去重，过滤掉 20 分钟以下的短视频
  → 评分算法选出最优 1 条（见下方评分逻辑）

youtube-transcript-api
  → 提取英文字幕
  → 如果第一名没有英文字幕，自动尝试第二名、第三名……

Gemini 2.5 Pro（思考模式）
  → 第一次调用：生成中文杂志文章（含英文原文引用 + 中文翻译）
  → 第二次调用：生成 18 字以内的中文文件名
  → 第三次调用：生成邮件素材（背景介绍 / 金句 / 故事 / 评论精选）

文件生成
  → Playwright (Chromium) 把 HTML 渲染成 PDF，存本地（带视频封面）
  → ebooklib 打包成 EPUB，用于邮件附件

Gmail SMTP
  → 发送邮件：正文包含原视频标题、发布时间、背景介绍、金句、故事、评论原文
  → 附件是 EPUB

macOS launchd
  → 每天 08:00 自动触发整个流程
  → Mac 需要开着（建议系统设置 → 电池 → 计划，07:55 自动唤醒）
```

所有密钥和配置放在 `.env` 文件里，代码通过 `python-dotenv` 读取，绝对不硬编码。已处理过的视频 ID 存在 `processed.json`，防止同一个视频被处理两次。

---

## 每个模块详解

### 1. 视频选择：双轨制 + 评分

**轨道一：5 个精选频道**（最近 30 天）
- Lenny's Podcast、Andrej Karpathy、Lex Fridman、Y Combinator、Every

**轨道二：全站关键词搜索**
```python
SEARCH_QUERIES = [
    'vibe coding AI',
    'AI tools startup founder',
    'AI product management',
    'LLM AGI startup',
]
```

两轨合并去重后统一评分，没有频道优先级，只看质量。

---

### 2. 评分算法

```
分数 = log10(点赞数 + 1) × 40      ← 主要权重（互动质量）
     + log10(播放量 + 1) × 10      ← 次要权重（传播广度）
     + 每匹配一个关键词 × 15        ← 内容相关性
     + Builder 信号每匹配 × 20      ← 加分（真正在做东西的人）
     - Influencer 信号每匹配 × 25   ← 扣分（流量内容）
     + 时效性加分（最多 3 分）       ← 权重极低，几乎忽略
```

**为什么用对数？**
原始点赞数会让 Lex Fridman（10 万赞）永远碾压小频道（500 赞）。用 log10 压缩之后，差距从 200:1 缩小到 2:1，关键词匹配和 Builder 信号就有机会帮好内容翻盘。

**Builder vs Influencer 是什么？**
- Builder 信号（加分）：`how i built`、`deep dive`、`architecture`、`tutorial`、`open source`…— 说明这个人真的在做东西
- Influencer 信号（扣分）：`top 10`、`you need to know`、`react to`、`i made $`、`X ways`…— 典型流量标题，内容往往空洞

**搜索窗口为什么是 30 天？**
48 小时太短，很多好视频发布几天后才积累出高互动量。30 天能捞到真正经过时间验证的好内容，同时 `processed.json` 防止重复处理。

---

### 3. 字幕提取与 Fallback

选出评分最高的视频后，依次尝试提取字幕。如果第一名没有英文字幕，自动跳到第二名，以此类推，直到找到一个可用的。

```python
for candidate in candidates:  # candidates 是按分数降序排列的列表
    try:
        transcript = get_transcript(candidate['id'])
        video = candidate
        break
    except RuntimeError:
        continue  # 跳过，试下一个
```

这个 fallback 逻辑是后来加的。早期版本只处理第一名，一旦没字幕就整个流程崩掉。

---

### 4. Gemini 生成文章

Prompt 写得很细，核心要求：
- 中文杂志腔，不是翻译腔
- 只提取"宝石级洞见"（反直觉观点、具体故事、可落地策略），不要励志废话
- 遇到特别精彩的表达，用 `> ` 引用英文原文，下一行 `> 译：` 给中文翻译
- 开头引言自然传达：这个人是谁、话题利害关系、为什么值得读

**Prompt 质量直接决定文章质量。** 这是整个项目最重要的部分，没有之一。

---

### 5. 邮件结构（最终版）

```
头部
  ├── 文章中文标题
  ├── 频道 · 发布于 YYYY-MM-DD
  ├── 原视频英文标题（斜体）
  ├── 👍 点赞数 · 👁 播放量
  └── ▶ 观看原视频

关于这期视频（Gemini 生成）
  ├── 第一段：主讲人/受访者是谁，为什么值得听
  └── 第二段：这个频道是什么定位，为什么值得关注

传播素材
  ├── 最有传播力的金句（3条）
  ├── 最有共鸣的故事（3个，含标题+描述）
  └── 评论精选（3-4条原文 + 中文翻译）

附件
  └── {文章标题}.epub（可导入 Apple Books / Kindle）
```

---

### 6. PDF / EPUB

- **PDF**：第一页是视频封面图（深色背景 + 半透明缩略图），翻页是正文。正文中的英文引用显示为金色左边框的 blockquote，下方有中文译文。
- **EPUB**：开头有中文标题 + 原视频英文标题 + 频道信息，正文同样包含 blockquote 引用。

---

## 为什么这么选技术

| 技术 | 原因 |
|------|------|
| **Gemini 2.5 Pro** | 思考模型，处理长字幕质量明显更好；同属 Google 生态，与 YouTube API 协同顺畅 |
| **Playwright** | 能精准控制 Chrome 渲染，PDF 效果远好于 WeasyPrint 等纯 Python 方案 |
| **ebooklib** | Python 生成 EPUB 的事实标准 |
| **launchd** | macOS 原生定时任务，零依赖 |
| **.env + dotenv** | 密钥和代码分离，上 GitHub 只需排除 `.env` |
| **GitHub** | 代码托管，版本控制，方便 handover |

---

## 踩过的所有坑

### 坑 1：youtube-transcript-api 的 API 改了

```python
# ❌ 旧用法（已废弃）
YouTubeTranscriptApi.list_transcripts(video_id)

# ✅ 新用法
api = YouTubeTranscriptApi()
t = api.fetch(video_id, languages=['en', 'en-US', 'en-GB'])
```

**教训**：用第三方库前先看 changelog，有 "deprecation" 字样的要特别注意。

---

### 坑 2：Gemini API 的"免费额度"是 0

`gemini-2.5-pro` 对新账号没有任何免费额度，必须先开启 Google Cloud 计费才能用。先后换了三个 key 才搞清楚这件事。

**教训**：Google 的"免费"经常带引号，新模型通常没有免费层。用之前先查清楚计费规则。

---

### 坑 3：Playwright 渲染 PDF 时 30 秒卡死

HTML 里有 `@import url('https://fonts.googleapis.com/...')` 这行 CSS，无头浏览器会去请求外部字体，网络慢时超时卡死。

```python
# 解法：渲染前删掉这行
html_pdf = re.sub(r"@import url\('https://fonts\.googleapis\.com[^']*'\);", '', html_content)
page.set_content(html_pdf, wait_until='domcontentloaded')
```

**教训**：离线渲染 HTML 时，所有外部资源（字体、图片、CDN）必须换成本地或 base64 内联。

---

### 坑 4：ebooklib 的"Document is empty"错误

```python
ch.content = xhtml_string   # ❌ 不行
ch.set_content(xhtml.encode('utf-8'))  # ✅ 要传 bytes，不是 str
```

**教训**：ebooklib 内部期望字节（bytes）。类型错误不一定会报明显的错，有时只是静默产生空文件。

---

### 坑 5：EPUB 封面图片重复

`book.set_cover()` 内部已经把图片加进书里了，如果再手动 `book.add_item()` 一次，会出现 "Duplicate name" 警告。

**教训**：用库的高级方法时，不要同时手动做同一件事。读文档，了解高级方法底层做了什么。

---

### 坑 6：Gemini 模型名字失效

`gemini-2.5-pro-exp-03-25` 这个带日期的名字在某个时间点消失了。

**教训**：用不带日期的稳定别名（`gemini-2.5-pro`），带日期的实验性名字随时会消失。

---

### 坑 7：字幕提取没有 fallback，一失败就全崩

早期版本 `get_transcript()` 失败后直接 `return`，整个当天流程结束。后来遇到印地语视频（没有英文字幕）就白跑一天。

**解法**：`select_best_video()` 改为返回按分数排序的候选列表，`main()` 里依次尝试，直到找到有英文字幕的视频。

**教训**：单点失败不应该让整个流程崩溃。凡是依赖外部数据的环节，都要考虑"这个数据不存在时怎么办"。

---

### 坑 8：GitHub Actions 无法抓取 YouTube 字幕

尝试把脚本部署到 GitHub Actions 云端自动运行时，发现所有视频字幕抓取都失败，报错：

> YouTube is blocking requests from your IP. IPs belonging to a cloud provider (AWS, GCP, Azure...) are blocked by YouTube.

GitHub Actions 的服务器 IP 属于 Microsoft Azure，YouTube 直接封锁了。这是 `youtube-transcript-api` 的已知限制，没有免费的绕过方法（代理需要付费）。

**最终决策**：继续用 macOS launchd 本地运行，Mac 设置 07:55 自动唤醒，08:00 触发脚本。本地家用 IP 不会被封，稳定可靠，且完全免费。

**教训**：在云端运行脚本之前，要先确认所有依赖的外部服务是否允许云服务器访问。YouTube 字幕抓取这类"非官方"接口，云端 IP 通常会被封。

---

### 坑 9：GitHub 敏感信息泄露

在对话中直接把 GitHub Personal Access Token 发给了 AI。Token 出现在任何文本里，等于公开了——任何人都可以用它操作你的 GitHub 账号。

**解法**：立即去 GitHub Settings → Developer settings → Personal access tokens → Delete 撤销该 token。

**教训**：密码、token、API key 这类东西永远不要发给任何人，包括 AI。需要认证时，自己在终端里输入，不要复制给别人看。

---

### 坑 10：git push 认证问题

用 HTTPS 推代码时需要 Personal Access Token（GitHub 已不再支持密码认证）。Token 用一次就删，下次还要重新生成，很麻烦。

**解法**：换成 SSH 认证。生成一次 SSH key，绑定到 GitHub，之后永久免密推送。

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
cat ~/.ssh/id_ed25519.pub  # 复制这个内容到 GitHub Settings → SSH keys
git remote set-url origin git@github.com:用户名/仓库名.git
git push  # 之后每次直接 push，不需要任何密码
```

**教训**：项目一开始就用 SSH，省去后续麻烦。

---

## 文件结构

```
~/Desktop/Distilled/
├── distilled.py          # 主脚本（全部逻辑在这一个文件里）
├── .env                  # 所有密钥（绝对不能上 GitHub）
├── .env.example          # 配置模板（可以上 GitHub，供他人参考）
├── .gitignore            # 排除 .env、PDF、EPUB 等不上传的文件
├── requirements.txt      # Python 依赖列表
├── RULES.md              # 项目规则文档
├── FORsherry.md          # 这份复盘笔记
├── README.md             # GitHub 项目介绍（英文）
├── processed.json        # 已处理视频 ID（不上 GitHub）
├── distilled.log         # 运行日志（自动生成）
└── {文章标题}_{日期}.pdf  # 每天生成的文章

~/Library/LaunchAgents/
└── com.distilled.daily.plist    # macOS 定时任务（每天 08:00 触发）
```

**GitHub 仓库**：https://github.com/fxy2311-youyou/Distilled

---

## 如果要重新跑起来（新机器 / 交接）

```bash
# 1. 克隆代码
git clone https://github.com/fxy2311-youyou/Distilled
cd Distilled

# 2. 安装依赖
pip install -r requirements.txt
playwright install chromium

# 3. 配置密钥
cp .env.example .env
# 编辑 .env，填入：
#   YOUTUBE_API_KEY  → Google Cloud Console 申请
#   GEMINI_API_KEY   → Google AI Studio，需开启计费
#   GMAIL_USER       → Gmail 地址
#   GMAIL_PASSWORD   → Gmail 应用专用密码（16位，不是账号密码）
#   RECIPIENT_EMAIL  → 收件地址
#   OUTPUT_DIR       → 文章存哪儿

# 4. 手动测试
python distilled.py

# 5. 配置自动运行（macOS）
cp com.distilled.daily.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.distilled.daily.plist
# 系统设置 → 电池 → 计划 → 设置 07:55 自动唤醒
```

---

## 值得带走的思维方式

**1. 先手动跑通，再自动化**
整个项目先手动跑通了第一个视频的全流程，确认文章质量满意之后才搭自动化。避免花几小时建好框架，结果发现输出根本不好用。

**2. 每个环节都要有 log**
`log.info()` 贯穿全流程，每步做了什么、选了哪个视频、得了多少分都有记录。出问题打开 `distilled.log` 一分钟内定位到是哪一步崩了。没有 log 的系统出了问题就是在黑暗里摸索。

**3. 把"变化的"和"不变的"分开**
- 变化的：API key、邮箱、路径 → 放 `.env`
- 稳定的：业务逻辑、Prompt → 放代码

换 API key 只改 `.env`，不动代码，不会引入新 bug。

**4. Prompt 是产品核心**
整个项目大约 40% 的时间花在调 Prompt 上。模型调用只是一行代码，但 Prompt 的质量决定了输出内容是否值得读。好 Prompt 的三要素：明确角色、明确格式、明确什么要 / 什么不要。

**5. 单点失败不能让整个流程崩**
任何依赖外部数据的环节（字幕、封面图、评论）都要有 fallback：
- 字幕抓不到 → 试下一个候选视频
- 封面图下载失败 → 跳过封面，继续生成
- 评论抓取失败 → 用空列表，继续生成

**6. 云端不等于万能**
把脚本搬到云上（GitHub Actions）看起来很美好，但要先确认所有依赖的外部接口是否允许云服务器访问。这次因为 YouTube 封锁云端 IP，最终还是回到了本地运行。有时候本地方案才是最稳定的。

**7. 密钥从一开始就要保护好**
- 永远不发给任何人（包括 AI）
- 用完即删（Token 用一次就撤销）
- 上 GitHub 前检查 `.gitignore` 是否把 `.env` 排除了
- 优先用 SSH 认证，避免反复处理 token

---

## 还可以继续做的事

- [ ] Mac 定时唤醒配置：系统设置 → 电池 → 计划 → 07:55 唤醒
- [ ] 如果某天没有新视频，发一封"今日暂无内容"的提醒邮件，而不是静默跳过
- [ ] 支持多语言字幕 fallback（目前只试英文，可以加繁体中文等）
- [ ] 把 README 里的 "Apple Books" 改成 "Apple Books / Kindle"（已更新）
