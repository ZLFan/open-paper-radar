# 量子论文雷达（paper-radar）

每天早上把 **Scirate 量子热榜**、**arXiv quant-ph 新论文**、**DeepSeek 搜索** 三个来源汇总，用 DeepSeek 模型逐篇精读打分；对最值得读的论文（默认 **QRC 3 篇 + 量子算法 5 篇**）**抓 arXiv 全文**，生成「问题 / 方法 / 结果」三段式中文简报（含作者机构、摘要翻译、公式），最后推到**微信**并把完整版存进仓库。

> 面向两个并重方向：**量子储备池计算（QRC）/ 量子机器学习**，以及**量子算法**（重点是近最优量子算法 near-optimal quantum algorithms，以及其他值得关注的量子算法任务：量子模拟、量子搜索、QSVT、量子线性代数、量子优化、量子行走等）。改 `config/topics.json` 就能换方向。

## 效果

每天 09:30（北京时间）收到一条微信消息：

```
【量子论文雷达】10-20 周二

🔥 Scirate 热榜
1. `量子算法` Exact Quantum Query Complexity of the Five-Bit Symmetric Boolean Function（⬆12）
   💡 精确确定五比特对称布尔函数的量子查询复杂度，属于紧确界结果，是近最优量子算法方向的直接进展
   https://arxiv.org/abs/2610.26228
...
🎯 今日精选 3 篇
...
```

仓库里同时会提交完整版 `reports/2026-10-20.md`，可以当历史存档或挂到 GitHub Pages。

重点论文还会生成**结构化深度解读**（完整格式见 [docs/example-report.md](docs/example-report.md)）：

```markdown
### 1. Exact Quantum Query Complexity of the Five-Bit Symmetric Boolean Function

**中文标题**：五比特对称布尔函数的精确量子查询复杂度

**作者与机构**

| 作者 | 论文署名机构 |
| --- | --- |
| Songyao Xue | School of Software；Luoyang Normal University |

**链接**：SciRate · arXiv · 全文 · arXiv DOI

**摘要中文翻译**

设 f 为恰好在海明重量 3 或 4 的输入上取 1 的布尔函数……

**1. 主要解决的问题**

此前只有数值证据指向「三次查询可能够用」，但既没有算法也没有排除证明……

**2. 核心方法或创新**

① **Fourier–Gram 层分解**：把任意三查询算法按层展开……
② **有理对偶证书**：构造固定有理证书排除三查询……

**3. 值得关注的结果或意义**

$Q_E(\text{EXACT}^{5}_{3,4}) = 4$，终结了一个此前只有数值证据的开放问题……
```

---

### 微信里的排版

个人微信不渲染 LaTeX 和 Markdown 表格，所以推送版做了两处适配：

- **改用 HTML 模板**（PushPlus 的 `html` template）：换行用 `<br>` 控制，链接可点击，不再出现「内容挤成一坨」
- **公式转 Unicode**：`$\mathcal{D}_q^{\otimes n}$` → `D_q^(⊗ n)`、`$O(\sqrt{N})$` → `O(√N)`，保证手机上可读

完整版（含 LaTeX 公式、作者机构表格）仍然保存在仓库 `reports/` 里。

## 仓库拆分：私有运行 + 开源模板

本项目采用两个仓库：

| 仓库 | 可见性 | 作用 |
| --- | --- | --- |
| `open-paper-radar` | Public | 开源模板（代码 + 示例数据），同时托管每日 PDF 日报，通过 GitHub Pages 提供下载 |

**为什么拆开**：GitHub 免费账号的 private 仓库不支持 Pages，而微信推送需要 PDF 下载链接。折中方案是
PDF 生成在私有仓库、同步到公开仓库托管。每日 PDF 链接形如：

```
https://<你的用户名>.github.io/open-paper-radar/reports/YYYY-MM-DD.pdf
```

**需要额外配置一个 Secret**：`PUBLIC_REPO_TOKEN`，用于把 PDF 推送到公开仓库。创建方式：
GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens → 新建，
只授权 `open-paper-radar` 这一个仓库的 `Contents: Read and write` 权限，

```bash
gh secret set PUBLIC_REPO_TOKEN -b "github_pat_xxx" -R <你的用户名>/paper-radar
```

没有配置这个 Secret 时，PDF 仍会生成但只留在私有仓库，微信推送里不会显示下载链接。

## 一、架构

```
                    ┌──────────────────────────────┐
                    │  GitHub Actions（每天 09:30） │
                    └──────────────┬───────────────┘
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        ▼                          ▼                          ▼
┌───────────────┐        ┌─────────────────┐        ┌──────────────────┐
│ Scirate 热榜   │        │ arXiv quant-ph  │        │ DeepSeek Search  │
│ 按 scites 降序 │        │ 官方 API / 列表页 │        │ 跨源发现（博客、  │
│ 取前 10        │        │ 近 72h 新论文     │        │ 会议、GitHub）    │
└───────┬───────┘        └────────┬────────┘        └────────┬─────────┘
        │ 直连被 Cloudflare 拦时   │                          │
        │ 自动改用 DeepSeek Fetch  │                          │
        └──────────────────────────┼──────────────────────────┘
                                   ▼
                     合并去重（arXiv ID 优先，标题兜底）
                                   ▼
                    过滤已推送（data/state/seen.json）
                                   ▼
                规则预筛 → DeepSeek Chat 逐篇打分 + 写理由
                                   ▼
               Top N 论文 → 抓 arXiv 全文 → DeepSeek 生成三段式深度解读
                                   ▼
                    渲染日报（Markdown + 微信精简版）
                                   ▼
              PushPlus 微信推送  +  提交 reports/ 和 state 回仓库
```

关键设计：

| 问题 | 做法 |
| --- | --- |
| Scirate 有 Cloudflare，无法直接抓 | 三条路径依次降级：① 直连 → ② DeepSeek Fetch API 服务端抓取 → ③ **用 arXiv 新论文 + 模型评分生成「AI 热榜」**，替代 Scirate 前 10，日报里明确标注来源 |
| arXiv API 会 429 限流 | 自动降级到 `arxiv.org/list/quant-ph/new` 列表页 HTML 解析 |
| GitHub Actions 无状态，会重复推送 | `data/state/seen.json` 记录已推送的 arXiv ID，workflow 每次跑完自动提交回仓库 |
| 每天论文太多，LLM 太贵 | 先按关键词/Scirate 排名/时效做规则预筛，只把 top 30 送 LLM 打分；深度解读按方向配额（QRC 3 + 量子算法 5） |
| 深度解读需要全文和作者机构 | 直接解析 `arxiv.org/html/<id>`（MathML 的 alttext 里就是 LaTeX），抓不到时依次降级 ar5iv、DeepSeek Fetch，再不行就基于摘要生成并明确标注 |
| 推送失败会丢内容 | 推送失败时不写去重状态，第二天自动重试；日报文件始终落盘 |

---

## 二、部署到 GitHub Actions（约 15 分钟）

### 步骤 1：创建 GitHub 仓库并推送

```bash
cd paper-radar
git init -b main
git add -A
git commit -m "feat: 每日量子论文雷达"
# 在 GitHub 上新建一个空仓库（建议 Public, Private 需要 Github 会员），然后：
git remote add origin git@github.com:<你的用户名>/paper-radar.git
git push -u origin main
```

### 步骤 2：申请 DeepSeek API Key

1. 打开 <https://platform.deepseek.com/>，注册并登录
2. 左侧「充值」充入最低金额（10 元就够用很久，见下方成本估算）
3. 左侧「API Keys」→「创建 API Key」，复制 `sk-` 开头的字符串

> Search / Fetch 接口和模型接口共用同一个 Key，不需要单独开通。

### 步骤 3：申请 PushPlus 微信推送

1. 手机微信扫码登录 <https://www.pushplus.plus/>
2. 进入「一对一推送」，复制页面上的 **token**
3. 关注它提示的微信公众号，之后消息会出现在这个服务号里

> 免费版每天 200 条，本项目每天只发 1 条，完全够用。
> 如果更习惯 Server酱：<https://sct.ftqq.com/> 微信扫码后复制 SendKey，同样填进 Secrets 即可。

### 步骤 4：把密钥写进 GitHub Secrets

仓库页面 → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**，添加：

| Name | Value |
| --- | --- |
| `DEEPSEEK_API_KEY` | 步骤 2 的 `sk-...` |
| `PUSHPLUS_TOKEN` | 步骤 3 的 token |

（可选，非必填）在 **Variables** 里可以加：

| Name | Value |
| --- | --- |
| `NOTIFY_PROVIDER` | `pushplus` / `serverchan` / `wxpusher` / `wecom` / `email`，默认自动判断 |
| `DEEPSEEK_BASE_URL` | 使用中转服务时才需要 |

### 步骤 5：允许 workflow 写回仓库

仓库页面 → **Settings** → **Actions** → **General** → 拉到最下面 **Workflow permissions** → 选中 **Read and write permissions** → Save。

> 这一步是必须的：每天跑完要把 `reports/` 和 `data/state/` 提交回仓库，否则去重状态无法跨天保留，会重复推送。

### 步骤 6：手动触发第一次运行

仓库页面 → **Actions** → 左侧「每日论文雷达」→ 右上 **Run workflow** → 勾选 `dry_run = true` 先试跑（只生成日报、不推送）→ Run。

跑完后仓库里会出现 `reports/2026-10-20.md`，点开检查内容。确认没问题后，再 **Run workflow** 一次并把 `dry_run` 设为 `false`，微信就能收到消息。

之后每天北京时间 **09:30** 自动运行。

---

## 三、本地运行（调试用）

```bash
cd paper-radar
cp .env.example .env
# 编辑 .env，至少填 DEEPSEEK_API_KEY 和 PUSHPLUS_TOKEN

# 只生成日报、不推送、不写状态（推荐第一次用这个）
python3 scripts/run.py --dry-run

# 不调用 LLM，验证抓取和渲染链路
python3 scripts/run.py --dry-run --no-llm

# 正式跑一次（会推送并写状态）
python3 scripts/run.py

# 强制重跑某天（补日报）
python3 scripts/run.py --date 2026-10-19 --no-notify
```

要求 Python 3.11+，**零第三方依赖**，不需要 `pip install` 任何东西。

### 配置完先做一次体检

```bash
python3 scripts/check.py
```

它会逐项检查：DeepSeek Key 是否有效、Search API 是否可用、**Scirate 走哪条抓取通道**、arXiv 是否可达、推送渠道是否配好。输出类似：

```
[1/5] DeepSeek API Key
  ✅ 已配置（sk-abc…wxyz）
  ✅ Search API 可用，返回 3 条结果
[2/5] Scirate 抓取（三条路径依次降级）
  ✅ 抓取成功（通道：DeepSeek Fetch），拿到 10 篇，第 1 名：The Fundamental Memory Ceiling…
[3/5] arXiv 数据源
  ✅ 可用，近 72h 内抓到 100 篇
[4/5] 推送渠道
  ✅ 已配置：pushplus
[5/5] 去重状态
  ⚠️  尚未创建（首次正式运行后生成）
```

重点看第 2 项：显示 `通道：DeepSeek Fetch` 就说明 Scirate 热榜能稳定拿到；如果显示 ❌，见下方「已知限制」。

常用参数：

| 参数 | 作用 |
| --- | --- |
| `--dry-run` | 只渲染并在终端打印结果，不推送、不写状态 |
| `--no-llm` | 跳过 LLM 精读，只用规则排序 |
| `--no-notify` | 生成日报但不推送 |
| `--no-state` | 忽略去重状态（把抓到的全部当新论文） |
| `--no-deep-dive` | 跳过全文抓取与深度解读（调试用，省钱） |
| `--date YYYY-MM-DD` | 指定运行日期，用于补跑 |
| `--verbose` | 打印调试日志 |

跑单元测试：

```bash
PYTHONPATH=src python3 -m unittest discover -s tests -v
```

---

## 四、配置说明（`config/topics.json`）

| 字段 | 说明 |
| --- | --- |
| `focus.description` | 研究方向描述，直接进入 LLM 打分 prompt。**换方向优先改这里** |
| `directions` | 日报里给每篇论文打的方向标签（默认 `QRC` / `量子算法`），按关键词归类 |
| `report.scirate_top_n` | Scirate 热榜取前几篇，默认 10 |
| `report.max_items` | 日报最多展示多少篇（含热榜） |
| `scirate.feeds` | Scirate 板块，`quant-ph` 就是量子物理；也可填 `quant-ph` 之外的 feed |
| `arxiv.categories` | arXiv 分类，可加 `cs.LG`、`cond-mat.stat-mech` 等 |
| `arxiv.lookback_hours` | 只保留最近多少小时的论文，默认 72 |
| `search.queries` | DeepSeek Search 的查询列表，支持 `{today}`、`{date_minus_N}` 日期占位符和 `site:`、`after:`、`lang:` 语法 |
| `keywords.priority_any` | 最高优先关键词（如近最优量子算法）。只放高特异性词，避免 `quantum advantage` 这类宣传性热词把无关论文顶上来 |
| `keywords.core_any` | 具体研究方向 / 算法任务关键词（强相关） |
| `keywords.adjacent_any` | 泛化邻近关键词（权重最低） |
| `deep_dive.enabled` | 是否生成全文深度解读 |
| `deep_dive.per_direction` | **按方向分配名额**：`{"QRC": {"top_n": 3, "min_llm_score": 6.5}, "量子算法": {"top_n": 5, "min_llm_score": 7.0}}`，某方向候选不足时用未标方向但分数达标的论文补位 |
| `deep_dive.max_fulltext_chars` | 送进 LLM 的全文长度上限，默认 5 万字符 |
| `llm.max_candidates` | 每天送 LLM 精读的候选上限，控制成本 |
| `llm.batch_size` | 每次请求评估几篇，默认 8 |
| `notify.provider` | 推送渠道；环境变量 `NOTIFY_PROVIDER` 会覆盖它 |

### 修改搜索主题或关键词

> 只需要改 `config/topics.json` 一个文件，改完 `git push`，下次定时运行自动生效。
> 本地验证：`python3 scripts/run.py --dry-run`（不推送、不写状态）。

#### 1. 四个要一起改的地方

改方向不是只改关键词，四处联动才能生效：

| 配置位置 | 负责什么 | 只改关键词不改它的后果 |
| --- | --- | --- |
| `focus.description` | 告诉 LLM「我在关注什么」，直接拼进打分 prompt | 规则预筛选出了新方向的论文，LLM 仍按旧标准打分 |
| `keywords.priority_any / core_any / adjacent_any` | 规则预筛，决定哪些论文能进 LLM 精读（每天只送 30 篇） | 新方向论文在预筛阶段就被刷掉，后面全看不到 |
| `directions` | 日报里的方向标签、深度解读的分组 | 论文没有方向标签，不会进入深度解读配额 |
| `search.queries` | DeepSeek 联网搜索，负责发现 arXiv 之外的内容 | 只能靠 arXiv 分类兜底，漏掉期刊/博客/会议内容 |

如果还要做深度解读，再加一处：`deep_dive.per_direction` 里给新方向分配名额。

#### 2. 三层关键词怎么分工

`relevance_score()` 会扫标题+摘要（转小写），分三档加权：

| 层级 | 权重 | 放什么 | 例子 |
| --- | --- | --- | --- |
| `priority_any` | 最高（命中即 3 分起） | 你**最想盯死**的高特异性表达 | `quantum query complexity`、`near-optimal quantum`、`matching lower bound` |
| `core_any` | 中（2 分起） | 具体研究方向 / 算法任务名 | `quantum reservoir`、`hamiltonian simulation`、`quantum optimization` |
| `adjacent_any` | 低（每个 0.35 分，封顶 1.2） | 宽泛邻近词，用来捞边缘相关 | `quantum information`、`entanglement`、`quantum algorithm` |

**选词原则**：

- 关键词写**英文小写**（论文标题摘要都是英文；匹配时会 `lower()`，所以配置里写小写最直观）
- `priority_any` 只放**高特异性**词。反例：`quantum advantage`、`quantum speedup` 这种宣传性热词几乎每篇都提，放进 priority 会把「云量参数化的量子神经网络」这类论文顶到第一（这是实测踩过的坑）
- 宁可 `core_any` 写多条具体任务词，也不要把一个泛词放到 priority

#### 3. 场景 A：微调现有方向的关键词

比如量子算法方向想多盯「量子纠错」（片段示例，`...` 表示保留原有词条）：

```json
"keywords": {
  "priority_any": [ ..., "quantum error correction", "surface code" ],
  "core_any": [ ..., "fault-tolerant", "logical qubit", "syndrome extraction" ],
  "adjacent_any": [ ... ]
}
```

改完跑一次 dry-run，日志里看 `规则预筛：N 篇候选进入 LLM 精读`，再打开 `reports/日期.md` 检查有没有抓到目标论文。

#### 4. 场景 B：新增一个关注方向（以「量子纠错」为例）

四处联动改（片段示例，`...` 表示保留原有配置）：

```json
{
  "focus": {
    "description": "（在原有描述末尾追加）此外关注量子纠错：表面码、逻辑量子比特、容错阈值、错误缓解与纠错码设计。"
  },
  "directions": [
    { "name": "QRC", "any": ["quantum reservoir", "reservoir computing"] },
    { "name": "量子算法", "any": ["quantum algorithm", "query complexity"] },
    { "name": "量子纠错", "any": ["quantum error correction", "surface code", "logical qubit", "fault-tolerant"] }
  ],
  "keywords": {
    "priority_any": [ ..., "quantum error correction code" ],
    "core_any": [ ..., "surface code", "logical qubit", "syndrome", "decoder" ],
    "adjacent_any": [ ..., "error mitigation", "noise characterization" ]
  },
  "search": {
    "queries": [
      { "name": "量子纠错", "query": "\"quantum error correction\" surface code after:{date_minus_5}", "count": 10 }
    ]
  },
  "deep_dive": {
    "per_direction": {
      "QRC": { "top_n": 3, "min_llm_score": 6.5 },
      "量子算法": { "top_n": 5, "min_llm_score": 7.0 },
      "量子纠错": { "top_n": 3, "min_llm_score": 7.0 }
    }
  }
}
```

注意三点：

- `directions[].name` 必须和 `deep_dive.per_direction` 的 key **完全一致**（含中文/英文），否则配额分配不到
- 新方向的论文要能被 `directions[].any` 命中，才会打上标签
- 深度解读总篇数会变成 3+5+3=11 篇，成本会上升，留意 `五、成本`

#### 5. 场景 C：换成完全不同的领域（以遥感为例）

```json
{
  "focus": { "description": "SAR 遥感、土壤湿度反演、变化检测、地球观测基础模型" },
  "directions": [
    { "name": "遥感", "any": ["remote sensing", "SAR", "soil moisture", "change detection"] }
  ],
  "keywords": {
    "priority_any": ["soil moisture retrieval"],
    "core_any": ["soil moisture", "synthetic aperture radar", "change detection", "InSAR"],
    "adjacent_any": ["remote sensing", "radar backscatter", "earth observation"]
  },
  "arxiv": { "categories": ["physics.geo-ph", "cs.CV"] },
  "search": {
    "queries": [
      { "name": "土壤湿度", "query": "\"soil moisture\" SAR after:{date_minus_7}", "count": 10 },
      { "name": "变化检测", "query": "\"change detection\" remote sensing after:{date_minus_7}", "count": 10 }
    ]
  },
  "deep_dive": { "per_direction": { "遥感": { "top_n": 5, "min_llm_score": 7.0 } } }
}
```

同时记得改 `report.subtitle`（日报副标题）和 `scirate.feeds`（如果不再用 `quant-ph`）。

#### 6. DeepSeek 搜索查询语法

`query` 支持官方搜索语法，可组合使用：

| 语法 | 示例 | 作用 |
| --- | --- | --- |
| `site:` | `quantum computing site:arxiv.org` | 限定站点；多个用 `site:a OR site:b` |
| `-site:` | `DeepSeek -site:example.com` | 排除站点 |
| `-词` | `reservoir -advertisement` | 排除包含该词的页面 |
| `"..."` | `"quantum reservoir computing"` | 精确短语匹配（长尾关键词必用） |
| `-"..."` | `- "release notes"` | 排除短语 |
| `lang:` | `lang:en` 或 `lang:zh,en` | 限定语言 |
| `after:` | `after:2026-10-15` | 限定发布日期之后 |
| `before:` | `before:2026-10-20` | 限定发布日期之前 |

日期占位符（运行时自动替换）：

| 占位符 | 替换成 |
| --- | --- |
| `{today}` | 运行当天 |
| `{yesterday}` | 昨天 |
| `{date_minus_N}` | N 天前（N 可写 0-30） |

其他限制：

- `query` 长度 **1~128 个 Unicode 字符**（超了会自动截断并告警）
- `count` 是返回条数（1~50），**不影响计费**（按请求次数计费）
- 每个查询都是一次计费请求，`search.queries` 条数直接决定每天成本

#### 7. 改完怎么验证

```bash
# 1. 先确认数据源和密钥都正常
python3 scripts/check.py

# 2. 跑完整流程但不推送（推荐加 --no-deep-dive 先快速验证抓取和排序）
python3 scripts/run.py --dry-run --no-deep-dive

# 3. 看终端日志里的关键行：
#    规则预筛：N 篇候选进入 LLM 精读
#    方向「量子纠错」深度解读候选不足：1/3 篇
# 4. 打开 reports/2026-XX-XX.md 检查热榜和精选里有没有目标论文
#    如果全是无关论文，说明关键词太泛（adjacent 权重过高）或 query 太宽
```

确认没问题后 `git push`，GitHub Actions 下次定时运行就会用新配置。

#### 8. 常见坑

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 新方向论文完全没出现 | 只改了 `focus` 或 `search`，没改 `keywords`，预筛阶段就被刷掉 | 三层关键词都要加 |
| 热榜全是综述/新闻 | `priority_any` 放了泛词 | 换成高特异性词，泛词降级到 `adjacent_any` |
| 深度解读数量不足 | `directions` 没给新方向打上标签 | 检查 `directions[].any` 的词是否能在标题/摘要里命中 |
| 方向标签全是「其他」 | `directions[].name` 拼写和 `per_direction` 的 key 不一致 | 复制粘贴保证一致 |
| 搜索结果一条都没有 | `after:` 时间窗太窄，或 `query` 里的短语加了引号但原文表述不同 | 放宽到 `{date_minus_7}`，去掉引号试试 |

---

## 五、成本

| 项目 | 用量 | 单价 | 每月成本 |
| --- | --- | --- | --- |
| DeepSeek Search | 每天 4 次查询 | 5 元 / 1000 次 | ≈ 0.6 元 |
| DeepSeek Fetch（抓 Scirate） | 每天 1 次 | 5 元 / 1000 次 | ≈ 0.15 元 |
| DeepSeek Chat（精读打分） | 每天 4 次请求，约 2 万 token | 约 2 元 / 百万 token | ≈ 0.6 元 |
| DeepSeek Chat（8 篇深度解读：QRC 3 + 算法 5） | 每篇约 2 万 token 输入 + 2.5k 输出 | 约 2 元 / 百万 token | ≈ 10 元 |
| GitHub Actions | 每天约 1 分钟 | 免费额度 2000 分钟/月 | 0 元 |
| PushPlus | 每天 1 条 | 免费 | 0 元 |
| **合计** | | | **约 13 元 / 月** |

> 搜索和 Fetch 只有「返回 200 且结果非空」才计费，搜不到不扣钱。
> 全文抓取优先直连 `arxiv.org/html`，这部分免费；只有直连失败才走 DeepSeek Fetch（每次 0.005 元）。
> 想省钱就把 `deep_dive.per_direction` 里的 `top_n` 调小（比如 QRC 1 + 算法 2），或干脆设 `enabled: false`。
> 8 篇深度解读的日报会比较长，微信里读不完的部分可以在仓库 `reports/` 看完整版。

---

## 六、已知限制与排查

### Scirate 抓不到（403 / Cloudflare）

Scirate 用 Cloudflare 的 managed challenge 保护。**实测结论：直连、真实浏览器（Playwright）、DeepSeek Fetch API 三条路全部被拦**，即使 GitHub Actions 的出口 IP 也一样。Search API 能索引到 scirate.com，但都是几个月前的旧快照，拿不到当天列表。

所以代码里做了完整的降级链：

1. **直连 Scirate**——某些网络环境可能可以；
2. **DeepSeek Fetch API**——服务端抓取，本轮实测返回 404 "failed to fetch URL"（被 Cloudflare 拦）；
3. **AI 热榜**——前两条都失败时，从 arXiv 新论文里按模型评分排序取前 10，作为「🔥 今日量子热榜（AI 排序）」展示。

第 3 条是当前实际生效的路径，所以每天的日报**仍然有 10 篇热榜**，只是排序依据从「社区投票」换成「模型对重要性/新颖性的判断」。深度解读候选也从中选取，保证 QRC 3 + 量子算法 5 的配额尽量填满。

想看当前走的是哪条路径，跑 `python3 scripts/run.py --dry-run --verbose`，日志里会打印 `Scirate 直连成功`、`Scirate 经 DeepSeek Fetch 成功` 或 `Scirate 不可达，已用 AI 评分生成今日热榜`。

> 如果你以后有能访问 Scirate 的网络环境（比如带浏览器的代理），代码里的直连路径会自动生效，热榜就恢复成真实的社区票数。

### arXiv API 429（Rate exceeded）

偶尔会被限流。代码会自动降级到 `arxiv.org/list/quant-ph/new` 列表页，不影响结果；如果两个源同时失败，日报「今日概览」表格里会显示 ⚠️。

### 微信没收到消息

1. 看 Actions 日志里 `推送成功：pushplus` 还是报错；
2. PushPlus 免费版需要**先关注服务号**才能收到；
3. token 是否填对、是否写在了 Secrets 而不是 Variables；
4. 消息可能被微信折叠在「服务通知」里。

### 重复推送 / 漏推

去重状态在 `data/state/seen.json`，workflow 每次跑完会提交它。如果发现某天重复推送，检查 Actions 日志的「保存日报与去重状态」步骤是否 push 成功（常见原因是步骤 5 没开 Write 权限）。

### 本地报 `CERTIFICATE_VERIFY_FAILED`

用 python.org 安装包或 Homebrew 安装的 Python 可能没有 CA 证书。代码里已自动回退到系统证书（`/etc/ssl/cert.pem` 等）；如果还报错，设置环境变量：

```bash
export SSL_CERT_FILE=/etc/ssl/cert.pem
```

---

## 七、目录结构

```
paper-radar/
├── config/topics.json          # 唯一需要改的配置文件
├── src/paper_radar/
│   ├── __main__.py             # 命令行入口与流程编排
│   ├── config.py               # 配置加载
│   ├── models.py               # Paper 数据模型与去重键
│   ├── http.py                 # HTTP 重试/证书回退
│   ├── deepseek.py             # Search / Fetch / Chat 客户端
│   ├── llm.py                  # 精读打分 prompt
│   ├── deep_dive.py            # 全文深度解读（问题/方法/结果三段式）
│   ├── rank.py                 # 合并去重、预筛、综合排序
│   ├── render.py               # Markdown / 微信版渲染
│   ├── notify.py               # 5 种推送渠道
│   ├── state.py                # 跨天去重状态
│   ├── sources/
│   │   ├── scirate.py          # Scirate 抓取（三条降级路径）
│   │   ├── arxiv.py            # arXiv API + 列表页解析
│   │   ├── fulltext.py         # arXiv HTML 全文解析（作者机构、公式、章节）
│   │   └── web_search.py       # DeepSeek Search 数据源
│   └── util.py
├── scripts/
│   ├── run.py                  # 本地运行入口（自动加载 .env）
│   └── check.py                # 环境体检：key / 数据源 / 推送渠道连通性
├── tests/                      # 40 个测试（含本地 mock 服务、全文解析夹具与端到端集成测试）
├── data/state/seen.json        # 去重状态（自动生成并提交）
├── reports/YYYY-MM-DD.md       # 每日日报（自动生成并提交）
└── .github/workflows/daily.yml # 定时任务
```

---

## 八、参考

- DeepSeek 搜索与抓取 API 文档：<https://api-docs.deepseek.com/zh-cn/guides/search/>
- Scirate 开源仓库（页面结构参考）：<https://github.com/scirate/scirate>
- arXiv API 文档：<https://info.arxiv.org/help/api/index.html>
- PushPlus 文档：<https://www.pushplus.plus/doc/>
