---
name: podcast-obsidian-write
description: Obsidian安全写入模块。使用obsidian-cli将格式化内容写入对应的Obsidian文档，含写入前校验和写入后验证。
---

# 模块4：Obsidian 安全写入

## 模块定位

本模块负责将模块3输出的格式化内容安全地写入Obsidian文档。使用 `obsidian` CLI 进行操作，执行前后均进行校验。

---

## 前提条件

- Obsidian 桌面客户端已启动
- config.json 中的 `vault_name` 和文档映射关系已配置正确
- 模块3的输出文件就绪

---

## 执行流程

### Step 1：写入前校验

**1.1 验证 Obsidian CLI 连通性**

```bash
obsidian search query="test" vault="<vault_name>"
```

确认 Obsidian 正常响应。

**1.2 验证目标文档存在性**

对三个目标文档逐一检查：

```bash
obsidian read file="<obsidian_file>" vault="<vault_name>"
```

其中 `obsidian_file` 为 config.json 中 `document_mapping` 各分类的 `obsidian_file` 字段值。

- ✅ 文档存在 → 继续
- ❌ 文档不存在 → 自动创建：
  ```bash
  obsidian create name="<文件名（不含路径）>" content="#" vault="<vault_name>" silent
  ```
  注意：`obsidian create` 只能在 vault 根目录创建。如果文档在子目录中，需要用 `eval` 命令或改用直接文件写入作为后备方案。

**1.3 检查月份标题是否已存在**

读取目标文档内容，检查当前月份标题（如 `# 2026/5`）是否已存在：
- 已存在 → 通知模块3无需重复追加月份标题
- 不存在 → 需要在内容前追加月份标题

### Step 2：安全写入

对每个有内容的分类，执行以下写入操作：

**2.1 构造写入内容**

从模块3的输出文件（如 `modules/temp/ai_tech_output.md`）读取格式化后的Markdown文本。

**2.2 通过 obsidian-cli append 追加**

```bash
obsidian append file="<obsidian_file>" content="<markdown_content>" vault="<vault_name>" silent
```

> 注意事项：
> - `content` 参数中用 `\n` 表示换行
> - 使用 `silent` 防止自动打开文件
> - 过大的内容可能需要分段写入

**2.3 分段写入策略**

如果单次 append 内容过长（超过 obsidian-cli 参数限制），按以下方式分段：
1. 将内容按新闻条目拆分
2. 每次 append 一个日期段的完整内容
3. 逐段写入，每段写入后校验

### Step 3：写入后验证

**3.1 内容验证**

```bash
obsidian read file="<obsidian_file>" vault="<vault_name>"
```

确认最新追加的内容出现在文档中。

**3.2 完整性检查**

- 确认日期标题已写入
- 确认新闻条目数量与预期一致
- 确认月份标题未重复（当月第一天场景）

---

## 错误处理

| 错误场景 | 处理方式 |
|----------|----------|
| Obsidian 未启动 | 提示用户先打开 Obsidian，然后重试 |
| append 失败 | 重试1次，仍失败则尝试使用 `obsidian eval` 通过 JS API 写入 |
| 文件被外部修改 | 读取最新内容，检查冲突，提示用户确认后重试 |
| 内容过长导致写入截断 | 改用分段写入策略 |
| vault 名称不匹配 | 从错误信息中提取可用 vault 名称，提示用户确认 |

---

## obsidian-cli 备用方案

当 `obsidian append` 命令遇到问题时，可使用 `obsidian eval` 通过 JavaScript API 写入：

```bash
obsidian eval code="app.vault.adapter.write('<relative_path>', '<content>')" vault="<vault_name>"
```

> ⚠️ eval 方案绕过 Obsidian 的编辑历史，仅在 append 失败时使用。

---

## 典型执行示例

```
🔍 写入前校验:
   Obsidian CLI 连通性 ✅
   AI与科技文档: 播客记录/AI与科技Laifu每日新闻博客 ✅ (exists)
   金融商业文档: 播客记录/金融与商业Laifu每日新闻博客 ✅ (exists)
   政策文档: 播客记录/国内外政策Laifu每日新闻博客 ✅ (exists)
   月份标题 "# 2026/5": 已存在 ⚠️（跳过追加）

💾 写入:
   AI与科技 → 9 条新闻 append ✅
   金融商业 → 0 条（无内容，跳过）
   国内外政策 → 0 条（无内容，跳过）

🔍 写入后验证:
   AI与科技文档: 最新内容包含 "## Fri.1" ✅ (9 items)
   金融商业文档: 无变更 ✅
   政策文档: 无变更 ✅

📊 写入完成: 1/3 文档已更新
```
