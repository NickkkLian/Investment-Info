# 情报终端 · Intelligence Desk

> 一个面向个人的金融与 AI 情报面板：自动聚合行业前沿动态，记录与分析个人投资，并从大厂公开招聘数据中解读行业方向。

打开即用的单页应用，托管于 GitHub Pages。前端不含任何密钥；所有个人数据存放在使用者自己的私有仓库中，由 GitHub Actions 定时抓取、用 Claude 模型摘要后回写。

---

## 功能

**1. 行业动态（金融 / AI）**
自动从十余个英文一手与权威来源（WSJ、CNBC、Federal Reserve、OpenAI、arXiv、Hugging Face、Google AI 等）抓取最新资讯，由 Claude 生成中文摘要、自动分类，并对重要性评分排序，重大消息优先呈现。

**2. 个人投资**
在浏览器内管理持仓、交易记录与投资计划/复盘，自动计算成本与浮动盈亏，一键同步回私有仓库。

**3. 招聘趋势**
从大厂的公开招聘接口（Greenhouse / Lever / Ashby）统计各类岗位数量随时间的变化，由 Claude 给出方向性研判——把招聘动向当作行业投入方向的前瞻信号。

---

## 工作原理

```
公开仓库 (GitHub Pages)            私有仓库 (使用者自有)
┌────────────────────┐           ┌──────────────────────────┐
│ index.html         │  读/写    │ investment/               │
│  · 纯静态前端       │ ───────▶  │   invest.json  个人投资   │
│  · 零密钥           │           │   news.json    动态(自动) │
│  · README.md        │           │   jobs.json    趋势(自动) │
└────────────────────┘           │   config.json  源/公司配置│
                                  │   scripts/collect.mjs     │
        ┌─────────────────────┐  │ .github/workflows/        │
        │ GitHub Actions      │─▶│   investment-sync.yml     │
        │  定时抓取 + Claude   │  └──────────────────────────┘
        │  摘要/研判，回写数据 │
        └─────────────────────┘
```

**安全模型（设计核心）：** 公开仓库里的前端不含任何密钥。两类凭据分开存放，互不暴露——

| 凭据 | 存放位置 | 用途 |
|---|---|---|
| Anthropic API Key | 私有仓库的 Actions Secrets | 定时任务调用 Claude，**绝不进入前端** |
| GitHub 细粒度 PAT | 使用者浏览器 localStorage | 前端读写自己的私有数据，仅存于本机 |

任何访问公开站点的人只会看到一个要求输入凭据的空壳，看不到也用不了使用者的数据。

---

## 自行部署

### 1. 公开仓库
1. 新建 public 仓库，放入 `index.html` 与本 `README.md`。
2. Settings → Pages → Source 选 `main` 分支根目录，保存。
3. 几分钟后得到 `https://<用户名>.github.io/<仓库名>/`。

### 2. 私有仓库
1. 放入 `investment/` 文件夹与 `.github/workflows/investment-sync.yml`（workflow 必须在仓库**根目录**才会被 GitHub 识别）。`investment/` 自成一体，可与仓库内其它内容共存。
2. Settings → Secrets and variables → Actions → 新建 secret `ANTHROPIC_API_KEY`（在 console.anthropic.com 获取）。

### 3. 配置 `investment/config.json`
- **RSS 源**：预置 15 个英文源（AI 与金融约 40/60），每项带 `tag` 归类与 `verified` 核对状态（脚本忽略，仅供阅读）。可自由增删；失效的源会被自动跳过。
  - 注：Anthropic 官方已停用 RSS，预置项通过 Google News 搜索接口抓取相关报道（标记为聚合源），非一手。
- **追踪公司**：把占位项替换为真实公司，`ats` 填 `greenhouse` / `lever` / `ashby`，`slug` 取自其招聘页 URL：
  - `boards.greenhouse.io/<slug>` · `jobs.lever.co/<slug>` · `jobs.ashbyhq.com/<slug>`
  - 可用对应公开接口验证：`boards-api.greenhouse.io/v1/boards/<slug>/jobs` 等。
  - 未配置真实 slug 的占位项会被自动跳过。仅使用官方公开接口，不爬取违反服务条款的站点。

### 4. 创建浏览器用的 PAT
GitHub → Settings → Developer settings → Fine-grained tokens：仅授予目标私有仓库的 **Contents: Read and write** 权限即可。

### 5. 首次运行
私有仓库 → Actions → Investment Sync → Run workflow 手动跑一次，之后按计划自动运行。打开站点，填入 owner / 私有仓库名 / PAT 即可连接。

---

## 运行节奏与成本

定时任务默认 **每周一、周四各运行一次**（cron `0 16 * * 1,4`，即温哥华上午 9:00；GitHub cron 使用 UTC）。

模型分工：新闻摘要用 **Claude Haiku 4.5**（逐条处理，不漏条），新闻重要性复核与招聘研判用 **Claude Sonnet 4.6**（更强判断力，确保重大新闻不被埋没）。某一步失败会自动回退，不影响整体运行。

| 项目 | 模型 | 每次 | 每月（约 8.7 次） |
|---|---|---|---|
| 新闻摘要 | Haiku | $0.023 | $0.20 |
| 重要性复核 | Sonnet | $0.009 | $0.08 |
| 招聘研判 | Sonnet | $0.032 | $0.28 |
| **合计** | | **~$0.064** | **~$0.55** |

未配置追踪公司时招聘研判跳过，仅新闻成本约 $0.28/月；含波动区间大致 $0.4–$0.8/月。GitHub Actions（私有库免费额度内）、静态前端与数据读写均不额外计费。

可调项：`news_max_per_run`（当前 20）、cron 频率、是否启用重要性复核、研判历史窗口大小、各步所用模型。

---

## 关于招聘趋势这一方法

招聘数据是**方向性**信号，而非精确预测。值得关注的是某类岗位随时间的相对变化与 JD 主题迁移，而非绝对数量。数据存在噪声（幽灵岗位、长期常青职位、重复发布），宜作佐证而非单独结论。增强信号的方式：追踪更多同赛道公司、拉长时间线、在 `collect.mjs` 中细化关键词分类。

---

## 技术栈

纯静态前端（原生 HTML/CSS/JS，无构建步骤）· GitHub Pages 托管 · GitHub Actions 定时任务（Node.js）· Anthropic Claude API（Haiku 4.5 + Sonnet 4.6）· 数据以 JSON 存于 GitHub 仓库，无需数据库服务器。

---

## 路线图

- 持仓现价自动刷新（接入行情数据源）
- 招聘趋势按类别的时间序列图表
- 浏览器内交互式分析（基于当前持仓与当日动态的即时问答）
- 动态关键词高亮与收藏
