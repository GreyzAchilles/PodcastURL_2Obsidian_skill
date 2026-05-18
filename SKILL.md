---
name: podcast-processor
description: 自动化处理播客内容并分类写入Obsidian文档，按AI科技、金融商业、国内外政策三个维度进行新闻分析和归档。采用模块化设计，通过总控文档顺序调用各子模块。
---

# 播客内容分析与分类处理器（模块化版 v2）

## 功能概述

将播客 URL 中的新闻内容自动获取、智能分类，并写入对应的 Obsidian 文档。采用模块化架构，每个步骤独立为一个子模块，通过本文档定义的执行顺序进行调度。

## 核心能力

- **三维度新闻分类**：AI与科技 / 金融与商业 / 国内外政策
- **浏览器自动获取**：优先本地浏览器自动化，降级 WebFetch
- **智能格式化**：自动日期标题 + 月份标题管理
- **Obsidian CLI 写入**：安全追加，写入前后双重校验

## 环境与依赖

| 依赖项 | 要求 | 检查方式 |
|--------|------|----------|
| Obsidian 桌面客户端 | 必须运行中 | `obsidian search query="test"` |
| obsidian CLI | 随 Obsidian 安装 | `obsidian help` |
| 浏览器（可选） | Edge/Chrome/Firefox 至少一个 | 自动检测，缺失时降级 WebFetch |
| WebFetch 工具 | 内置工具，浏览器不可用时作为备选 | — |

## 目标文档

Vault: `PsykoPath`（路径：`D:\Obsidian\Base\PsykoPath`）

| 类别 | 文档路径（vault 内） | obsidian-cli file= 参数 |
|------|---------------------|------------------------|
| AI与科技 | `播客记录/AI与科技Laifu每日新闻博客.md` | `播客记录/AI与科技Laifu每日新闻博客` |
| 金融与商业 | `播客记录/金融与商业Laifu每日新闻博客.md` | `播客记录/金融与商业Laifu每日新闻博客` |
| 国内外政策 | `播客记录/国内外政策Laifu每日新闻博客.md` | `播客记录/国内外政策Laifu每日新闻博客` |

## 模块架构

```
PodcastURL_2Obsidian_skill/
├── SKILL.md                    ← 本文件（总控入口）
├── modules/
│   ├── 01_config.md            ← 配置验证与用户确认模块
│   ├── 02_browser_fetch.md     ← 浏览器内容获取模块
│   ├── 03_classify_format.md   ← 智能分类与格式化模块
│   ├── 04_obsidian_write.md    ← Obsidian 安全写入模块
│   ├── config.json             ← 持久化配置文件（首次运行后生成）
│   └── temp/                   ← 临时数据目录
│       ├── raw_news.json        ← 模块2输出：原始新闻数据
│       ├── ai_tech_output.md    ← 模块3输出：AI科技格式化内容
│       ├── financial_output.md  ← 模块3输出：金融商业格式化内容
│       ├── policy_output.md     ← 模块3输出：政策格式化内容
│       └── classification_stats.json ← 模块3输出：分类统计
└── PodcastURL_2Obsidian_skill_example：.md  ← 用例参考
```

## 执行流程

### 完整流程（首次运行 / 全量执行）

```
┌─────────────────────────────────────────────────────────┐
│ 开始                                                     │
│  ↓                                                       │
│ [前置检查] obsidian help 验证 Obsidian CLI 可用           │
│  ↓                                                       │
│ [前置检查] modules/config.json 是否存在？                  │
│  ├─ 不存在 → 执行模块1（配置验证与用户确认）               │
│  └─ 存在   → 读取 config.json，验证路径可访问性            │
│              ├─ 通过 → 跳过模块1，继续                      │
│              └─ 失败 → 执行模块1，重新配置                  │
│  ↓                                                       │
│ 向用户确认播客 URL 和目标日期（可选）                       │
│  ↓                                                       │
│ [模块2] 浏览器内容获取                                     │
│  ├─ 检测本地浏览器 → 自动化抓取                            │
│  └─ 未检测到 → 提示用户选择手动路径 或 降级 WebFetch       │
│  ↓                                                       │
│ [模块3] 智能分类与格式化                                   │
│  ├─ Part A: 关键词分类 + 置信度计算                        │
│  └─ Part B: Markdown 格式化 + 月份标题判断                 │
│  ↓                                                       │
│ [预览] 向用户展示格式化结果，请求确认                       │
│  ↓                                                       │
│ [模块4] Obsidian 安全写入                                  │
│  ├─ 写入前校验                                            │
│  ├─ 安全追加                                              │
│  └─ 写入后验证                                            │
│  ↓                                                       │
│ [输出报告] 展示处理统计和写入结果                           │
│  ↓                                                       │
│ 结束                                                     │
└─────────────────────────────────────────────────────────┘
```

### 后续运行（非首次，配置已存在）

```
┌─────────────────────────────────────────────────────────┐
│ 开始                                                     │
│  ↓                                                       │
│ [快速检查] obsidian CLI + config.json 路径验证            │
│  ↓ （仅在异常时触发模块1）                                  │
│ 向用户确认播客 URL（和目标日期）                            │
│  ↓                                                       │
│ [模块2] → [模块3] → [预览确认] → [模块4]                  │
│  ↓                                                       │
│ 结束                                                     │
└─────────────────────────────────────────────────────────┘
```

---

## 模块调用协议

### 模块1：配置验证与用户确认

**触发条件**（满足任一）：
- `modules/config.json` 不存在
- 现有配置中的路径验证失败
- 用户主动要求修改配置

**读取文档**：`modules/01_config.md`

**执行完毕后**：确保 `modules/config.json` 存在且有效

---

### 模块2：浏览器内容获取

**触发条件**：每次处理播客 URL 时

**读取文档**：`modules/02_browser_fetch.md`

**输入**：
- `podcast_url`（用户提供）
- `browser_preference`（config.json）
- `edge_timeout` / `max_retries`（config.json）

**输出**：`modules/temp/raw_news.json`

---

### 模块3：智能分类与格式化

**触发条件**：模块2完成后

**读取文档**：`modules/03_classify_format.md`

**输入**：
- `modules/temp/raw_news.json`
- `target_date`（用户提供，或默认今天）
- `classification_threshold`（config.json）
- 文档映射关系（config.json）

**输出**：
- `modules/temp/ai_tech_output.md`
- `modules/temp/financial_output.md`
- `modules/temp/policy_output.md`
- `modules/temp/classification_stats.json`

**预览环节**：展示分类统计和格式化结果，请求用户确认后再执行模块4

---

### 模块4：Obsidian 安全写入

**触发条件**：用户对模块3预览结果确认后

**读取文档**：`modules/04_obsidian_write.md`

**输入**：
- 模块3的输出文件
- config.json 中的 vault_name 和文档映射

**输出**：内容追加到 Obsidian 目标文档中

---

## 全局错误处理

| 级别 | 场景 | 处理方式 |
|------|------|----------|
| L1 致命 | Obsidian 未启动 | 提示用户打开 Obsidian 后重试 |
| L1 致命 | Vault 不存在 | 触发模块1重新配置 |
| L2 可恢复 | 浏览器不可用 | 降级为 WebFetch 或提示手动提供内容 |
| L2 可恢复 | 页面加载失败 | 重试后降级处理 |
| L3 警告 | 分类置信度低 | 标记"待审核"，继续执行 |
| L3 警告 | 部分文档写入失败 | 重试，记录已成功写入的分类 |

---

## 分类规则速查

### AI与科技
- 关键词：AI、人工智能、机器学习、神经网络、算法、大数据、云计算、区块链、元宇宙、VR/AR、深度学习、技术突破、版本更新
- 策略：聚焦技术创新和应用场景

### 金融与商业
- 关键词：股票、债券、基金、银行、保险、数字货币、财报、并购、IPO、投融资
- 策略：聚焦市场数据和交易信息

### 国内外政策
- 关键词：法律、法规、政策、条例、政府决策、外交、贸易、合作、协议、监管
- 策略：聚焦政策法规制定和国际关系

---

## 格式规范

### 日期与非当月第一天
```markdown
## Fri.1
- 摘要内容1
- 摘要内容2
```

### 当月第一天（日=1）
```markdown
# 2026/5
## Fri.1
- 摘要内容1
- 摘要内容2
```

> 如果月份标题已存在于文档中，不重复追加。

---

## 典型交互流程

```
[前置检查]
✓ Obsidian CLI 连接正常
✓ 配置文件 modules/config.json 有效
✓ Vault "PsykoPath" 可访问
✓ 三个目标文档均存在

📋 参数确认:
   播客URL: https://podcast.example.com/episode-302
   目标日期: 2026/05/14 (用户指定)
   
? 确认开始处理？ [Y/n]: y

🚀 [模块2] 内容获取:
   浏览器: Microsoft Edge ✅
   页面加载完成 ✅ (耗时 4.2s)
   提取 11 条新闻 ✅

🤖 [模块3] 分类与格式化:
   AI与科技: 11 条 (confirmed: 9, pending_review: 2)
   金融商业: 0 条
   国内外政策: 0 条
   日期标题: ## Wed.14 (非当月第一天，不追加月份标题)

📄 预览:
   以下 AI与科技 内容将被写入:
   
   ## Wed.14
   - 小红书组织升级：AI与出海成新增长
   - DeepSeek V4发布，国产算力生态崛起
   - 苹果Vision Pro团队解散，AI成新重心
   ... (11 items total)

? 确认写入？ [Y/n]: y

💾 [模块4] Obsidian 写入:
   播客记录/AI与科技Laifu每日新闻博客 ✅ (11 items)
   播客记录/金融与商业Laifu每日新闻博客 — (无内容)
   播客记录/国内外政策Laifu每日新闻博客 — (无内容)

📊 处理报告:
   ✅ 成功处理 11 条新闻
   📈 分类: AI科技(100%)
   📝 更新文档: 1/3
   
🎉 处理完成！
```
