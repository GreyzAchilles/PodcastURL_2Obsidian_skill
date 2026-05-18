---
name: podcast-config
description: 配置验证与用户确认模块。首次使用或路径异常时自动调用，其余时候仅当用户主动要求修改配置时调用。
---

# 模块1：配置验证与用户确认

## 模块定位

本模块负责项目的所有环境配置工作。**非每次运行必执行**，仅在以下场景触发：

1. **首次使用**：用户第一次运行本项目，无任何配置文件存在
2. **路径异常**：配置验证环节检测到 Vault 路径或目标文档不可访问
3. **用户主动**：用户明确要求修改配置参数

正常情况下，配置以 `modules/config.json` 持久化存储，后续运行时直接读取复用，无需重复执行本模块。

---

## 执行流程

### 场景A：首次使用 / 路径异常（自动触发）

```
触发条件: 未检测到 config.json，或 config.json 中路径验证失败

操作:
1. 向用户展示当前检测到的环境信息（浏览器、Obsidian vault 等）
2. 请用户确认或修改下方默认配置参数
3. 验证配置可访问性
4. 保存配置到 modules/config.json
```

### 场景B：用户主动修改配置

```
触发条件: 用户明确要求修改配置

操作:
1. 读取当前 config.json，展示现有配置
2. 请用户指定需要修改的参数
3. 验证新配置可访问性
4. 更新 modules/config.json
```

---

## 默认配置参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `vault_path` | string | `D:\Obsidian\Base\PsykoPath` | Obsidian Vault 根目录路径 |
| `vault_name` | string | `PsykoPath` | Obsidian Vault 名称（用于 obsidian-cli vault= 参数） |
| `browser_preference` | string | `auto` | 浏览器偏好：`auto`=自动检测，`edge`=Edge，`chrome`=Chrome，`firefox`=Firefox，`manual`=手动指定路径，`webfetch`=使用 WebFetch 替代 |
| `browser_manual_path` | string | 空 | 用户手动指定的浏览器可执行文件路径（仅 `browser_preference=manual` 时使用） |
| `edge_timeout` | integer | 30 | 浏览器页面加载超时（秒） |
| `max_retries` | integer | 3 | 网络错误最大重试次数 |
| `classification_threshold` | number | 0.7 | 分类置信度阈值（≥此值为确定分类，<此值标记"待审核"） |

### 目标文档映射（相对于 vault_path）

| 类别 | 文件名 | obsidian-cli file= 参数 |
|------|--------|------------------------|
| AI与科技 | `AI与科技Laifu每日新闻博客.md` | `播客记录/AI与科技Laifu每日新闻博客` |
| 金融与商业 | `金融与商业Laifu每日新闻博客.md` | `播客记录/金融与商业Laifu每日新闻博客` |
| 国内外政策 | `国内外政策Laifu每日新闻博客.md` | `播客记录/国内外政策Laifu每日新闻博客` |

> ⚠️ 注意：实际文件位于 vault 内部的 `播客记录/` 子目录下。

---

## 配置验证清单

每次配置确认/修改后，必须逐项验证：

### 1. Obsidian CLI 连通性
```bash
obsidian search query="test" vault="<vault_name>"
```
- ✅ 成功返回结果 → Obsidian CLI 正常
- ❌ 失败 → 提示用户确认 Obsidian 已打开，并检查 vault 名称是否正确

### 2. Vault 路径可访问性
检查 `vault_path` 目录是否存在，三个目标文件所在子目录是否可写入。

### 3. 浏览器检测（仅 browser_preference=auto 时）
按以下优先级检测本地浏览器：
1. `C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe` — Edge（32位默认路径）
2. `C:\Program Files\Microsoft\Edge\Application\msedge.exe` — Edge（64位路径）
3. `C:\Program Files\Google\Chrome\Application\chrome.exe` — Chrome
4. `C:\Program Files\Mozilla Firefox\firefox.exe` — Firefox

选择第一个检测到的浏览器。如果全部未检测到，将 `browser_preference` 降级为 `webfetch`，并提示用户。

---

## config.json 格式示例

```json
{
  "vault_path": "D:\\Obsidian\\Base\\PsykoPath",
  "vault_name": "PsykoPath",
  "browser_preference": "edge",
  "browser_manual_path": "",
  "edge_timeout": 30,
  "max_retries": 3,
  "classification_threshold": 0.7,
  "document_mapping": {
    "ai_tech": {
      "filename": "AI与科技Laifu每日新闻博客.md",
      "obsidian_file": "播客记录/AI与科技Laifu每日新闻博客"
    },
    "financial": {
      "filename": "金融与商业Laifu每日新闻博客.md",
      "obsidian_file": "播客记录/金融与商业Laifu每日新闻博客"
    },
    "policy": {
      "filename": "国内外政策Laifu每日新闻博客.md",
      "obsidian_file": "播客记录/国内外政策Laifu每日新闻博客"
    }
  }
}
```

---

## 错误处理

| 错误场景 | 处理方式 |
|----------|----------|
| Obsidian 未启动 | 提示用户先打开 Obsidian，然后重试 |
| Vault 名称不匹配 | 列出可用的 vault 名称供用户选择 |
| Vault 路径不存在 | 询问用户是否创建目录，或重新指定路径 |
| 目标文件不存在 | 询问是否自动通过 obsidian create 创建 |
| 所有浏览器均未检测到 | 自动切换为 webfetch 方案并通知用户 |
