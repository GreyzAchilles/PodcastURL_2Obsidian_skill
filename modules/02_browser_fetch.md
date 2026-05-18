---
name: podcast-browser-fetch
description: 浏览器内容获取模块。优先启动用户设备中的浏览器解析URL，未检测到浏览器时提供手动指定路径或WebFetch备选方案。
---

# 模块2：浏览器内容获取

## 模块定位

本模块负责从播客 URL 中获取原始内容。采用**分级降级策略**：优先使用本地浏览器自动化抓取，不可用时提供备选方案。

---

## 浏览器检测与选择逻辑

### 自动检测（browser_preference=auto）

按以下优先级逐一检测，使用第一个检测到的浏览器：

| 优先级 | 浏览器 | 检测路径 |
|--------|--------|----------|
| 1 | Microsoft Edge (32位) | `C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe` |
| 2 | Microsoft Edge (64位) | `C:\Program Files\Microsoft\Edge\Application\msedge.exe` |
| 3 | Google Chrome | `C:\Program Files\Google\Chrome\Application\chrome.exe` |
| 4 | Mozilla Firefox | `C:\Program Files\Mozilla Firefox\firefox.exe` |

### 降级策略

```
浏览器检测
  ├─ 检测到浏览器 → 启动浏览器自动化抓取（方案A）
  └─ 未检测到任何浏览器 → 向用户提问选择：
       [1] 手动提供浏览器可执行文件路径
       [2] 使用 WebFetch 工具直接抓取页面内容
```

---

## 方案A：浏览器自动化抓取

### 前提条件
- `browser_preference` 为 `auto`、`edge`、`chrome`、`firefox` 或 `manual`
- 对应浏览器的可执行文件存在

### 执行步骤

**Step 1：启动浏览器**

```
启动参数:
- 窗口大小: 1200×800 像素
- JavaScript: 启用
- 超时时间: config 中的 edge_timeout（默认30秒）
```

**Step 2：导航并加载**

```
操作: 使用 browser 工具打开 podcast_url
等待条件: 页面主体内容加载完成（EC.presence_of_element_located）
重试策略: 最多 max_retries 次（默认3次），每次间隔递增
```

**Step 3：内容提取**

从页面 DOM 中提取所有新闻条目，清洗 HTML 标签，提取以下字段：

```json
{
  "title": "新闻标题",
  "content": "新闻正文（纯文本，已清理HTML标签）",
  "timestamp": "发布时��（如有）"
}
```

输出结构：新闻条目数组 `[{title, content, timestamp}, ...]`

---

## 方案B：WebFetch 降级抓取

### 前提条件
- `browser_preference` 为 `webfetch`，或用户选择方案B

### 执行步骤

**Step 1：调用 WebFetch**

```
使用 WebFetch 工具，传入:
- url: podcast_url
- prompt: "提取页面中所有播客新闻条目，每个条目包含标题、正文内容、发布时间，以JSON数组格式输出"
```

**Step 2：解析与校验**

从 WebFetch 返回的结果中解析出新闻条目数组。如果页面内容无法有效提取（例如返回登录页、反爬虫页面等），提示用户提供浏览器路径或手动复制内容。

---

## 输出规范

无论采用哪种方案，输出结构统一为：

```json
{
  "source_url": "https://...",
  "fetch_method": "browser | webfetch",
  "extract_time": "YYYY-MM-DD HH:mm:ss",
  "news_count": 18,
  "news_items": [
    {
      "title": "新闻标题",
      "content": "新闻正文内容",
      "timestamp": "2026-05-14 10:30"
    }
  ]
}
```

将输出保存为临时文件，供下游模块使用。

---

## 错误处理

| 错误场景 | 处理方式 |
|----------|----------|
| 浏览器启动失败 | 提示用户检查浏览器安装，建议切换至 WebFetch |
| 页面加载超时 | 重试（递增超时时间），仍失败则降级为 WebFetch |
| 登录页/反爬虫 | 提示用户提供已登录的 Cookie，或切换至手动提供内容 |
| DOM 结构异常 | 记录实际 DOM 结构，尝试适应性提取 |
| WebFetch 返回空内容 | 提示用户手动复制粘贴播客内容 |

---

## 典型执行示例

```
🔍 浏览器检测:
   检测到 Microsoft Edge → C:\Program Files (x86)\Microsoft\Edge\Application/msedge.exe ✅

🚀 启动浏览器自动化:
   窗口大小: 1200×800 ✅
   JavaScript: 启用 ✅
   超时: 30秒 ✅

📡 页面加载:
   URL: https://prod-fmshare.laifufm.com/...
   状态: 加载成功 ✅ (耗时 4.2秒)

📋 内容提取:
   检测到 11 条新闻条目 ✅
   数据标准化完成 ✅

💾 输出: modules/temp/raw_news.json (11 items)
```
