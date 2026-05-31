# 情报终端 · 部署说明

一个个人「数据库网站」：金融/AI 动态、个人投资记录、大厂招聘趋势。
静态前端放公开仓库，个人数据放私有仓库，密钥分两处存放，互不暴露。

---

## 文件去向

```
公开仓库 (public, 部署 GitHub Pages)
├── index.html                         ← 整个前端，零密钥
└── README.md                          ← 部署说明，零密钥（GitHub 会渲染在仓库首页）

私有仓库 (private, 可与你其它数据库共存)
├── .github/workflows/
│   └── investment-sync.yml            ← 定时同步任务
└── investment/                        ← 本项目独立文件夹
    ├── config.json                    ← 你来配置：RSS 源 + 追踪的公司
    ├── invest.json                    ← 个人投资数据（前端读写）
    ├── news.json                      ← 自动生成的动态（Actions 写）
    ├── jobs.json                      ← 自动生成的岗位趋势（Actions 写）
    └── scripts/
        ├── collect.mjs                ← 采集 + Haiku 摘要 + Sonnet 复核/研判
        └── package.json
```

> ⚠️ `.github/workflows/` 必须在私有仓库**根目录**，不能放进 `investment/`，否则 GitHub 不识别。其余全部收在 `investment/` 内，不影响你库里其它数据库。

---

## 密钥架构（重要）

| 密钥 | 放哪 | 用途 | 谁能看到 |
|---|---|---|---|
| Anthropic API Key | 私有仓库 → Settings → Secrets → Actions，名为 `ANTHROPIC_API_KEY` | 定时任务调 Haiku + Sonnet | 只有 Actions，**绝不进网页** |
| GitHub Fine-grained PAT | 你浏览器 localStorage（首次打开网站时粘贴） | 前端读写私有数据 | 只在你这台设备 |

公开仓库的 `index.html` 里没有任何密钥，别人打开只看到要求输入 token 的空壳。

---

## 部署步骤

### 1. 公开仓库
1. 新建一个 public 仓库，上传 `index.html`。
2. Settings → Pages → Source 选 `main` 分支根目录，保存。
3. 几分钟后得到网址 `https://<用户名>.github.io/<仓库名>/`。

### 2. 私有仓库
1. 在你现有的私有仓库里，按上面的结构放入 `investment/` 文件夹和 `.github/workflows/investment-sync.yml`。
2. Settings → Secrets and variables → Actions → New secret：
   - Name: `ANTHROPIC_API_KEY`，Value: 你的 Anthropic API key（在 https://console.anthropic.com 获取）。

### 3. 配置追踪源 `investment/config.json`
- **RSS 源**：已预置 **15 个英文源**（2026-05 逐个核对，标 ✅ 的确认有效），AI 与金融按约 40/60 混合。可增删。
  - 金融：WSJ Markets、CNBC（Top News / Investing / Market Insider）、Federal Reserve
  - AI：OpenAI 官方、arXiv cs.AI、Hugging Face、Google AI、VentureBeat AI、WIRED AI、MIT News AI
  - 交叉：Hacker News、TechCrunch
  - ⚠️ **Anthropic 官方已停 RSS**，预置项用 Google News 搜索接口抓二手报道（标 `聚合`），不是一手源；想要一手只能官方邮件订阅（无法进 RSS 管线）。
  - 每个源带 `tag`（归类）和 `verified`（核对状态）字段，脚本忽略它们，仅供你看。
- **公司岗位**：把占位项替换成真实公司。`ats` 填 `greenhouse` / `lever` / `ashby`，`slug` 是该公司在对应平台的标识。

  **怎么找 slug**：打开公司招聘页，看 URL：
  - `boards.greenhouse.io/【slug】` 或 `job-boards.greenhouse.io/【slug】`
  - `jobs.lever.co/【slug】`
  - `jobs.ashbyhq.com/【slug】`

  也可直接在浏览器访问接口验证返回 JSON：
  - `https://boards-api.greenhouse.io/v1/boards/【slug】/jobs`
  - `https://api.lever.co/v0/postings/【slug】?mode=json`
  - `https://api.ashbyhq.com/posting-api/job-board/【slug】`

  > 没配真实 slug 的占位项会被脚本自动跳过，不会报错。LinkedIn 不支持也不要爬（违反 ToS）。

### 4. 创建给浏览器用的 PAT
GitHub → Settings → Developer settings → Fine-grained tokens → Generate：
- Resource owner：你自己 / 组织
- Repository access：Only select repositories → 选你的私有仓库
- Permissions → Repository permissions → **Contents: Read and write**（其它都不需要）
- 生成后复制（只显示一次）。

### 5. 首次运行
- 私有仓库 → Actions → Investment Sync → Run workflow（手动跑一次，之后按计划自动跑）。
- 打开你的网站，填入 owner / 私有仓库名 / PAT，点连接。
  - 「动态」「趋势」会显示 Actions 生成的数据；
  - 「投资」可直接增删记录，点「保存到 GitHub」即写回私有仓库。

---

## 运行节奏与成本
- 定时任务默认 **每周一、周四 温哥华早 9:00** 各跑一次（cron `0 16 * * 1,4`）。BC 自 2026-03-08 起永久 UTC−7，换季无需再调时间。GitHub 定时器有时会延迟几分钟到几十分钟，属正常。
- **新闻分两步**：① Haiku 4.5（`claude-haiku-4-5-20251001`，$1/$5）逐条翻译+摘要+初评分（20 条全处理，不漏条）；② Sonnet 4.6（`claude-sonnet-4-6`，$3/$15）拿标题清单复核重要性、覆盖分数，确保重大新闻不被埋没。某步失败会自动回退，不影响运行。
- **岗位研判**用 Sonnet 4.6（需要更强推理）。
- 每次抓取上限 `news_max_per_run = 20`。

成本估算（每月约 8.7 次）：

| 项目 | 模型 | 每次 | 每月 |
|---|---|---|---|
| 新闻摘要 | Haiku | $0.023 | $0.20 |
| 新闻重要性复核 | Sonnet | $0.009 | $0.08 |
| 岗位研判 | Sonnet | $0.032 | $0.28 |
| **合计** | | **~$0.064** | **~$0.55** |

- 还没配公司（slug 是占位）时研判跳过 → 只剩新闻，约 **$0.28/月**。
- 含波动区间大致 **$0.4–$0.8/月**，一年约 $6–8。
- 不额外花钱：GitHub Actions（私有库免费 2000 分钟/月，月用量约 15 分钟）、静态前端、GitHub API 读写都是 $0。

### 省钱钮（都只改一两处）
- `config.json` 的 `news_max_per_run`（当前 20）调小 → 新闻成本同比下降。
- cron 改更稀疏，如每周一次 `0 16 * * 1`。
- 关掉新闻重要性复核：删 `collect.mjs` 里那段 Sonnet re-rank（省每次 ~$0.009，但重要性回到 Haiku 初评）。
- `collect.mjs` 里岗位研判历史窗口 `jobs.snapshots.slice(-40)` 缩到 `-15`。
- 若连研判也想省，把 `MODEL_ANALYSIS` 换成 Haiku（质量略降）。

---

## 关于「岗位判断趋势」这个方法
是有效的**方向性**信号，不是精确预测：
- 看的是某类岗位**随时间的相对变化**（趋势页会显示与上一快照的 ▲▼ 对比），以及 JD 主题迁移，而非绝对数量。
- 噪声来源：幽灵岗位、长期常青 req、重复发布。所以只当佐证，别单独下结论。
- 想增强信号：追踪更多同赛道公司、拉长时间线、在 `collect.mjs` 的 `CATS` 里细化你关心的关键词（如把 "agent / inference / RAG / safety" 单列）。

---

## 可选的下一步
- 持仓「现价」目前手填；可加一个行情 API（如有免费源）在 Actions 里自动刷新写入 invest.json 的 price 字段。
- 趋势页可加按类别的时间序列折线图。
- 动态可加关键词高亮 / 收藏。

需要我加其中任何一个就说。
