---
name: podcast-classify-format
description: 智能分类与格式化模块。合并了原步骤3（智能分类与分析）和步骤4（格式化与文档准备），包含月份标题判断逻辑。
---

# 模块3：智能分类与格式化

## 模块定位

本模块接收模块2输出的新闻条目数组，执行分类分析和Markdown格式化，输出可直接写入Obsidian文档的格式化文本。

---

## Part A：智能分类（原步骤3）

### 输入

模块2的输出 —— 新闻条目数组 `[{title, content, timestamp}, ...]`

### 分类规则

#### 分类维度与关键词

**AI与科技** (`ai_tech`)：
- 技术术语：AI、人工智能、机器学习、神经网络、算法、大数据、云计算、区块链、元宇宙、VR/AR、深度学习
- 产品发布：新产品、版本更新、技术突破、研发成果
- 行业趋势：数字化转型、智能化升级、技术革新
- 企业关键词：OpenAI、DeepSeek、英伟达、字节跳动、小米、特斯拉（科技相关条目）

**金融与商业** (`financial`)：
- 金融领域：股票、债券、基金、银行、保险、数字货币、支付、利率
- 市场动态：股价变动、成交量、指数表现、市场分析、财报
- 企业经营：并购重组、IPO上市、投融资、独角兽
- 行业关键词：金融、贸易、上市、融资、投资

**国内外政策** (`policy`)：
- 政策法规：法律、法规、条例、办法、规定
- 政府决策：政策、措施、方案、计划、规划、监管
- 国际事务：外交、贸易、合作、协议、协定
- 国际关系：制裁、冲突、战争（地缘政治类）

#### 分类算法

对每条新闻条目，按以下逻辑执行：

```
1. 将 title + content 合并为待分析文本
2. 对每个类别，统计匹配到的关键词数量
3. 计算置信度: confidence = matched_keywords_count / total_keywords_in_category
4. 取置信度最高的类别作为主分类
5. 如果最高置信度 ≥ classification_threshold（默认0.7），标记为"确定"
6. 如果最高置信度 < classification_threshold，标记为"待审核"
7. 如果多个类别置信度接近（差值 < 0.15），标记为"多分类"[主分类, 次分类]
```

#### 摘要生成

对每条新闻，从 `content` 中提取代表性摘要（1-2句，≤50字）：
- 取 content 的前两句，或
- 对较长内容，取前50字
- 去除末尾不完整的句子

### 分类输出结构

```json
{
  "classified_news": [
    {
      "news_id": 1,
      "title": "原始标题",
      "summary": "摘要",
      "primary_category": "ai_tech | financial | policy",
      "confidence": 0.85,
      "status": "confirmed | pending_review | multi_tag",
      "secondary_category": "financial (仅 multi_tag 时)"
    }
  ],
  "statistics": {
    "total_count": 18,
    "ai_tech_count": 9,
    "financial_count": 6,
    "policy_count": 3,
    "pending_review": 0
  }
}
```

---

## Part B：格式化与文档准备（原步骤4，含改进）

### 输入

- Part A 的分类结果
- 目标日期（用户指定，或默认今天）
- config.json 中的文档映射关系

### 日期处理

**输入格式**：`YYYY-MM-DD`（如 `2026-05-14`）

**日期标题格式**：`#Day.Month` — 如 `Fri.1`、`Sat.2`、`Thu.14`

映射规则：
- 星期：`Mon` / `Tue` / `Wed` / `Thu` / `Fri` / `Sat` / `Sun`
- 日期：去掉前导零（如 `01` → `1`，`14` → `14`）
- 最终格式：`#{Weekday}.{Day}`（如 `## Fri.1`）

### 月份标题判断（新增逻辑）

在生成日期标题前，**必须**执行以下判断：

```
IF target_date 的"日"为 1（即当月第一天）:
    在日期标题上方追加月份标题: "# {year}/{month}"
    示例: "# 2026/5"，然后换行，再接 "## Fri.1"

ELSE:
    不添加月份标题，直接生成日期标题
    示例: "## Thu.14"
```

**特殊情况**：如果目标文档中已存在当前月份的月份标题，不再重复追加。

### Markdown 生成规则

**排序**：AI与科技 → 金融与商业 → 国内外政策

**条目格式**：
```
- {摘要}
```

- 不使用 `【类别】` 前缀（因为内容已按分类写入不同的文档）
- 摘要简洁，≤50字
- 多分类条目同时在对应分类文档中写入

### 输出格式示例

**非当月第一天（如 2026-05-14）**：

在 `金融与商业Laifu每日新闻博客.md` 中追加：
```markdown
## Wed.14
- 美联储宣布维持利率不变，市场对此反应积极
- 软银千亿美元AI公司上市，布局算力与机器人赛道
```

**当月第一天（如 2026-05-01）**：

在 `AI与科技Laifu每日新闻博客.md` 中追加：
```markdown
# 2026/5
## Fri.1
- 小红书组织升级：AI与出海成新增长
- DeepSeek V4发布，国产算力生态崛起
```

> 注意：月份标题格式为 `# {year}/{month}`，其中 month 不带前导零。这参考了实际用例中的格式（如 `# 2026/4`、`# 2026/5`）。

### 完整性校验

格式化完成后执行以下校验：
1. 检查每个条目都有有效分类
2. 验证日期标题格式正确
3. 检查月份标题是否已存在（避免重复追加）
4. 生成预览供用户确认

### 输出规范

将三个分类的格式化内容分别输出：
- `modules/temp/ai_tech_output.md` — AI与科技格式化内容
- `modules/temp/financial_output.md` — 金融与商业格式化内容
- `modules/temp/policy_output.md` — 国内外政策格式化内容

同时将分类统计信息输出到 `modules/temp/classification_stats.json`。

---

## 典型执行示例

```
📊 分类分析:
   输入新闻: 11 条

🤖 AI与科技分类:
   关键词匹配: ai_tech(11), financial(3), policy(1)
   分类结果: 11 条 (confirmed: 9, pending_review: 2)

💰 金融商业分类:
   （已在上面步骤中排除 AI 科技条目后处理剩余条目）

📝 格式化:
   目标日期: 2026-05-01
   月份判断: 当月第一天 ✅ → 追加月份标题 "# 2026/5"
   日期标题: ## Fri.1
   非当月第一天: ❌

📄 输出:
   AI与科技 → modules/temp/ai_tech_output.md (9 items)
   金融商业 → modules/temp/financial_output.md (0 items)
   国内外政策 → modules/temp/policy_output.md (0 items)
   统计 → modules/temp/classification_stats.json
```
