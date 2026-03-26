# longform-to-brief

每天早上 8 点，邮箱里自动出现一篇中文深度文章——把昨天最值得看的 AI / 创业 YouTube 长视频，蒸馏成一杯 5 分钟能读完的浓缩咖啡。

---

## 它在解决什么问题

优质内容从来不缺，缺的是时间。

Lex Fridman 的访谈动辄 3 小时，Andrej Karpathy 的技术分享 1 小时起跳，YouTube 上每天还有大量值得看的 AI / 创业内容在涌现。你知道它们很重要，但根本没时间看完，更没精力主动去找。

这个项目的答案是：**别让用户去找内容，让内容自己来找用户。**

每天自动完成：选视频 → 提取字幕 → AI 写文章 → 发到邮箱。你只需要早上打开邮件。

---

## 效果

- 1-3 小时的视频 → 约 1500 字中文文章，5 分钟读完
- 文章风格：中文杂志腔，不是翻译腔，读起来像人写的
- 邮件正文包含：金句 / 共鸣故事 / 一句话精华 / 评论区热议话题（含原文引用 + 中文翻译）
- 附件是 EPUB，可直接导入 Apple Books
- 本地同时保存 PDF（带视频封面）

---

## 工作流程

```
YouTube API
  → 扫描 5 个指定频道（最近 30 天）
  → 用 4 组关键词在全 YouTube 搜索
  → 合并去重，过滤掉 20 分钟以下的短视频
  → 评分算法选出最优 1 条（互动量 + 关键词 + Builder 内容加分 - 流量内容扣分）

youtube-transcript-api
  → 提取英文字幕

Gemini 2.5 Pro（思考模式）
  → 生成中文杂志文章（含英文原文引用 + 中文翻译）
  → 生成邮件素材（金句 / 故事 / 一句话 / 评论话题）
  → 生成中文文件名

文件输出
  → PDF（Playwright 渲染，带视频封面图）
  → EPUB（作为邮件附件）

Gmail SMTP
  → 每天 08:00 自动发送

macOS launchd
  → 定时触发整个流程
```

---

## 视频评分逻辑

```
分数 = log10(点赞数 + 1) × 40
     + log10(播放量 + 1) × 10
     + 每匹配一个关键词 × 15
     + Builder 内容信号 × 20（"how I built"、"deep dive"、"architecture"…）
     - 流量内容信号 × 25（"top 10"、"you need to know"、"react to"…）
     + 时效性加分（最多 3 分，权重极低）
```

用对数压缩原始数字，避免大频道永远碾压小频道；Builder 内容加分、流量内容扣分，让真正有深度的视频浮出来。

---

## 快速开始

### 1. 安装依赖

```bash
pip install google-api-python-client youtube-transcript-api google-genai \
            python-dotenv playwright ebooklib python-docx
playwright install chromium
```

### 2. 配置 `.env`

复制 `.env.example`，填入你自己的 key：

```bash
cp .env.example .env
```

```ini
YOUTUBE_API_KEY=你的 YouTube Data API v3 Key
GEMINI_API_KEY=你的 Gemini API Key（需开启计费）
GMAIL_USER=你的 Gmail 地址
GMAIL_PASSWORD=Gmail 应用专用密码（16位）
RECIPIENT_EMAIL=接收邮件的地址
OUTPUT_DIR=/你想存文章的路径
```

### 3. 手动运行

```bash
python distilled.py
```

### 4. 配置每天自动运行（macOS）

编辑 `com.distilled.daily.plist`，把路径替换成你自己的，然后：

```bash
cp com.distilled.daily.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.distilled.daily.plist
```

---

## 监控的频道

| 频道 | 内容方向 |
|------|---------|
| Lenny's Podcast | 产品 / 增长 |
| Andrej Karpathy | AI / 技术深度 |
| Lex Fridman | 科技访谈 |
| Y Combinator | 创业 |
| Every | AI 时代的工作与思考 |

加上 4 组全站关键词搜索，覆盖 YouTube 上冒出来的高质量 AI / 创业内容。

---

## 文件说明

```
distilled.py          # 主脚本（全部逻辑）
.env                  # 密钥配置（不进 Git）
.env.example          # 配置模板
RULES.md              # 项目规则
FORsherry.md          # 项目复盘笔记（技术决策 + 踩坑记录）
processed.json        # 已处理视频 ID，防止重复（不进 Git）
```

---

## 注意事项

- Gemini 2.5 Pro 对新账号**没有免费额度**，必须先开启 Google Cloud 计费
- Gmail 需要开启两步验证，使用「应用专用密码」而不是账号密码
- PDF 生成依赖 Playwright，首次运行需要 `playwright install chromium`
