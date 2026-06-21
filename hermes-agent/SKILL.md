---
name: ninchat
description: |
  实时信息检索系统，AI时代的搜索基础设施。
  为 Hermes Agent 提供实时新闻、深度内容、事件追踪、AI热评、热搜统计等能力。
category: Search & Information
author: NineInfra
version: 1.2.0
---

# ninchat Search API — Hermes Agent 专用

实时信息检索系统，AI时代的搜索基础设施。

## 主要功能

- **搜索 API**: 基于 Meilisearch 的全文检索，支持 50+ 主流媒体
- **热点新闻**: 基于爬取内容聚合，多源相似话题合并
- **AI 热评**: 基于热点新闻，由 AI 生成的深度评论
- **热搜统计**: 基于用户搜索行为的热搜词统计

## 配置要求

使用 ninchat 搜索功能前，需要配置以下环境变量：

| 变量名 | 说明 | 示例值 | 必需 |
|--------|------|--------|------|
| `ninchat_base_url` | ninchat 后端 API 地址 | `https://www.ninchat.cn` | 否 |
| `ninchat_api_key` | API Key，以 `sk_ninso_` 开头 | `sk_ninso_xxxxx` | 是 |

### 获取 API Key

1. 访问 ninchat Web 界面 https://www.ninchat.cn
2. 登录后进入 **系统设置 → API Key**
3. 点击 **创建新密钥**，复制生成的 `full_api_key`

### 在 Hermes Agent 中配置

**方式一：通过 .env 文件（推荐）**

在 Hermes Agent 的 `.env` 文件中添加：
```bash
# ninchat 搜索 API 配置
ninchat_base_url=https://www.ninchat.cn
ninchat_api_key=sk_ninso_xxxxx
```

**方式二：通过 Hermes UI 配置**

1. 打开 Hermes Dashboard（默认 http://localhost:9119）
2. 进入 **Settings → Skills → ninchat**
3. 填入 `Base URL` 和 `API Key`
4. 保存配置

## 连接信息

- **Base URL**: `ninchat_base_url`（默认 `https://www.ninchat.cn`）
- **认证方式**: API Key 认证（`sk_ninso_` 开头）
- **调用方式**: Python `requests` / `httpx`，或 Hermes 的 `execute_code` 工具
- **数据格式**: JSON
- **默认超时**: 30秒

## Search Endpoint — POST /api/search

**Method**: POST
**Content-Type**: `application/json`

### 请求参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `query` | string | 是 | - | 搜索关键词 |
| `detail` | boolean | 否 | false | 是否返回完整内容 |
| `match_mode` | string | 否 | "exact" | 匹配模式：`exact`（精确短语匹配）、`all`（全词匹配）、`fuzzy`（模糊匹配） |
| `limit` | integer | 否 | 50 | 返回结果数 |
| `api_key` | string | 是 | - | API 密钥 |

> **搜索结果数限制**：实际返回结果数取决于用户角色 —
> - **未登录用户**：最多 100 条
> - **已登录普通用户**：最多 1000 条
> - **管理员用户**：最多 10000 条

### Request Body（基础搜索）
```json
{
  "query": "搜索关键词",
  "api_key": "sk_ninso_xxxxx"
}
```

### Request Body（带完整内容）
```json
{
  "query": "搜索关键词",
  "detail": true,
  "limit": 10,
  "api_key": "sk_ninso_xxxxx"
}
```

### Hermes Agent 专用代码示例

#### 简单搜索（基础版）
```python
"""
在 Hermes Agent 中调用 ninchat 搜索 API
返回摘要信息用于快速预览
"""
import requests
import os
from typing import Dict, List

def search_ninchat(query: str, limit: int = 10) -> Dict:
    """
    使用 ninchat API 搜索实时信息
    
    Args:
        query: 搜索关键词
        limit: 返回结果数量
    
    Returns:
        搜索结果字典
    """
    NINCHAT_BASE_URL = os.getenv("ninchat_base_url", "https://www.ninchat.cn")
    NINCHAT_API_KEY = os.getenv("ninchat_api_key", "")
    
    if not NINCHAT_API_KEY:
        raise ValueError("请先配置 ninchat_api_key 环境变量")
    
    try:
        resp = requests.post(
            f"{NINCHAT_BASE_URL}/api/search",
            json={
                "query": query,
                "limit": limit,
                "api_key": NINCHAT_API_KEY
            },
            timeout=30
        )
        resp.raise_for_status()
        return resp.json()
    except Exception as e:
        print(f"搜索失败: {e}")
        return {"status": "error", "message": str(e)}

# 调用示例
result = search_ninchat("最新科技新闻", limit=5)
print(result)
```

#### 深度搜索（带完整内容）
```python
"""
获取完整文章内容用于深入分析或摘要生成
"""
import requests
import os
from typing import List

def search_ninchat_detail(query: str, limit: int = 5) -> List[Dict]:
    """
    搜索并获取完整文章内容
    
    Args:
        query: 搜索关键词
        limit: 返回结果数量
    
    Returns:
        包含完整内容的文章列表
    """
    NINCHAT_BASE_URL = os.getenv("ninchat_base_url", "https://www.ninchat.cn")
    NINCHAT_API_KEY = os.getenv("ninchat_api_key", "")
    
    if not NINCHAT_API_KEY:
        return []
    
    try:
        resp = requests.post(
            f"{NINCHAT_BASE_URL}/api/search",
            json={
                "query": query,
                "detail": True,
                "limit": limit,
                "api_key": NINCHAT_API_KEY
            },
            timeout=30
        )
        data = resp.json()
        if data.get("status") == "ok":
            return data.get("results", [])
        return []
    except Exception as e:
        print(f"搜索失败: {e}")
        return []

# 使用示例
articles = search_ninchat_detail("人工智能最新进展", limit=3)
for i, article in enumerate(articles, 1):
    print(f"\n=== 第 {i} 篇文章 ===")
    print(f"标题: {article['title']}")
    print(f"链接: {article['url']}")
    print(f"内容摘要: {article['content'][:400]}...")
```

#### 结合 Hermes Agent 的完整对话示例
```python
"""
在 Hermes Agent 中实现问答 + 搜索的完整流程
"""
import requests
import os

def get_real_time_info(question: str) -> str:
    """
    基于用户问题搜索实时信息并返回摘要
    
    Args:
        question: 用户的问题
    
    Returns:
        格式化的搜索结果摘要
    """
    NINCHAT_BASE_URL = os.getenv("ninchat_base_url", "https://www.ninchat.cn")
    NINCHAT_API_KEY = os.getenv("ninchat_api_key", "")
    
    if not NINCHAT_API_KEY:
        return "请先配置 ninchat API Key"
    
    try:
        # 搜索相关信息
        resp = requests.post(
            f"{NINCHAT_BASE_URL}/api/search",
            json={
                "query": question,
                "detail": True,
                "limit": 3,
                "api_key": NINCHAT_API_KEY
            },
            timeout=30
        )
        data = resp.json()
        
        if data.get("status") != "ok":
            return "搜索失败，请稍后重试"
        
        results = data.get("results", [])
        if not results:
            return f"未找到关于 '{question}' 的最新信息"
        
        # 格式化输出
        output = f"📰 关于 '{question}' 的最新信息：\n\n"
        for i, item in enumerate(results[:3], 1):
            output += f"【{i}】{item['title']}\n"
            output += f"   🔗 {item['url']}\n"
            output += f"   📝 {item['content'][:300]}...\n\n"
        
        return output
    except Exception as e:
        return f"搜索时出错: {str(e)}"

# 调用示例
print(get_real_time_info("今天有什么重要新闻？"))
```

### curl 示例
```bash
curl -s https://www.ninchat.cn/api/search \
  -H "Content-Type: application/json" \
  -d '{"query": "最新新闻", "detail": true, "api_key": "sk_ninso_xxxxx"}'
```

### Response 格式

**detail=false（默认）**:
```json
{
  "status": "ok",
  "query": "搜索关键词",
  "detail": false,
  "total": 50,
  "results": [
    {
      "url": "https://...",
      "title": "文章标题",
      "snippet": "正文片段（前150字符）",
      "score": 1
    }
  ]
}
```

**detail=true（含完整内容）**:
```json
{
  "status": "ok",
  "query": "搜索关键词",
  "detail": true,
  "total": 50,
  "results": [
    {
      "url": "https://...",
      "title": "文章标题",
      "content": "完整正文内容（最长50000字符）",
      "crawled_date": "20260520"
    }
  ]
}
```

## Index Status — GET /api/indexed

**Method**: GET

**Response**:
```json
{
  "status": "ok",
  "total": 12345,
  "stats": {
    "numberOfDocuments": 12345,
    "isIndexing": false
  }
}
```

## 热点新闻 — GET /api/hot-news

获取基于爬取内容聚合的热点新闻，涵盖多个新闻源的相同/相似话题。

**Method**: GET

### 请求参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `limit` | integer | 否 | 20 | 返回新闻数量（1-50） |
| `days` | integer | 否 | 7 | 统计最近几天的新闻（1-30） |
| `include_content` | boolean | 否 | false | 是否包含完整内容 |

### Request 示例

```bash
curl -s "https://www.ninchat.cn/api/hot-news?limit=10&days=7"
```

### Response 格式

```json
{
  "status": "ok",
  "news": [
    {
      "title": "热点新闻标题",
      "source_count": 5,
      "sources": [
        {"name": "来源1", "url": "https://..."},
        {"name": "来源2", "url": "https://..."}
      ],
      "updated_at": "2026-06-02T10:00:00"
    }
  ],
  "count": 10,
  "days": 7
}
```

### Hermes Agent 代码示例

```python
"""
获取热点新闻并生成摘要
"""
import requests
import os
from typing import Dict, List

def get_hot_news(limit: int = 10, days: int = 7) -> List[Dict]:
    """
    获取热点新闻
    
    Args:
        limit: 返回新闻数量
        days: 统计天数
    
    Returns:
        热点新闻列表
    """
    NINCHAT_BASE_URL = os.getenv("ninchat_base_url", "https://www.ninchat.cn")
    
    try:
        resp = requests.get(
            f"{NINCHAT_BASE_URL}/api/hot-news",
            params={"limit": limit, "days": days},
            timeout=15
        )
        data = resp.json()
        if data.get("status") == "ok":
            return data.get("news", [])
        return []
    except Exception as e:
        print(f"获取热点新闻失败: {e}")
        return []

def format_hot_news_summary() -> str:
    """格式化热点新闻摘要"""
    news = get_hot_news(limit=8, days=7)
    if not news:
        return "暂无热点新闻"
    
    output = "🔥 今日热点新闻：\n\n"
    for i, item in enumerate(news, 1):
        output += f"【{i}】{item['title']}\n"
        output += f"   📰 {item['source_count']} 个媒体报道\n"
        output += f"   ⏰ {item['updated_at'][:16]}\n\n"
    
    return output

print(format_hot_news_summary())
```

## AI 热评 — GET /api/ai-hot-comments

获取基于热点新闻的 AI 生成的深度评论。

**Method**: GET

### 请求参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `days` | integer | 否 | 7 | 统计最近几天的新闻（1-30） |
| `limit` | integer | 否 | 10 | 返回评论数量（1-50） |
| `auto_generate` | boolean | 否 | true | 是否自动生成评论 |

### Request 示例

```bash
curl -s "https://www.ninchat.cn/api/ai-hot-comments?limit=10&days=7"
```

### Response 格式

```json
{
  "status": "ok",
  "comments": [
    {
      "title": "热点新闻标题",
      "source_count": 5,
      "sources": [{"name": "来源1", "url": "https://..."}],
      "hot_group_id": "xxx",
      "updated_at": "2026-06-02T10:00:00",
      "ai_comment": "AI 生成的深度评论内容...",
      "comment_status": "ready",
      "comment_generated_at": "2026-06-02T11:00:00"
    }
  ],
  "count": 10,
  "days": 7,
  "ai_enabled": true
}
```

### Hermes Agent 代码示例

```python
"""
获取 AI 热评并展示
"""
import requests
import os

def get_ai_hot_comments(limit: int = 10, days: int = 7):
    """
    获取 AI 热评列表
    
    Args:
        limit: 返回评论数量
        days: 统计天数
    
    Returns:
        (评论列表, AI 是否启用)
    """
    NINCHAT_BASE_URL = os.getenv("ninchat_base_url", "https://www.ninchat.cn")
    
    try:
        resp = requests.get(
            f"{NINCHAT_BASE_URL}/api/ai-hot-comments",
            params={"limit": limit, "days": days},
            timeout=30
        )
        data = resp.json()
        if data.get("status") == "ok":
            return data.get("comments", []), data.get("ai_enabled", False)
        return [], False
    except Exception as e:
        print(f"获取 AI 热评失败: {e}")
        return [], False

def format_ai_hot_comments() -> str:
    """格式化 AI 热评输出"""
    comments, ai_enabled = get_ai_hot_comments()
    if not comments:
        return "暂无 AI 热评"
    
    output = "🤖 AI 热评\n"
    output += "=" * 40 + "\n\n"
    
    if not ai_enabled:
        output += "⚠️  AI 功能未启用，无法生成热评\n\n"
    
    for i, item in enumerate(comments, 1):
        output += f"【{i}】{item['title']}\n"
        output += f"   📰 {item['source_count']} 个来源\n"
        
        if item.get('ai_comment'):
            output += f"\n   💡 AI 热评:\n"
            output += f"      {item['ai_comment']}\n"
        elif item.get('comment_status') == 'generating':
            output += f"\n   ⏳ AI 正在生成评论中...\n"
        elif item.get('comment_status') == 'failed':
            output += f"\n   ❌ AI 评论生成失败\n"
        
        output += "\n"
    
    return output

print(format_ai_hot_comments())
```

## 热搜词 — GET /api/hot-search

获取基于用户搜索统计的热门搜索词。

**Method**: GET

### 请求参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `limit` | integer | 否 | 20 | 返回热搜词数量（1-50） |
| `time_range` | string | 否 | "all" | 时间范围：today/week/month/all |

### Request 示例

```bash
curl -s "https://www.ninchat.cn/api/hot-search?limit=20&time_range=today"
```

### Response 格式

```json
{
  "status": "ok",
  "terms": [
    {
      "term": "搜索关键词",
      "count": 156,
      "rank": 1,
      "trend": "up",
      "lastSearched": "2026-06-02T10:00:00"
    }
  ],
  "count": 20,
  "time_range": "today"
}
```

### Hermes Agent 代码示例

```python
"""
获取热搜词并展示
"""
import requests
import os

def get_hot_search(limit: int = 20, time_range: str = "today") -> list:
    NINCHAT_BASE_URL = os.getenv("ninchat_base_url", "https://www.ninchat.cn")
    
    try:
        resp = requests.get(
            f"{NINCHAT_BASE_URL}/api/hot-search",
            params={"limit": limit, "time_range": time_range},
            timeout=15
        )
        data = resp.json()
        return data.get("terms", []) if data.get("status") == "ok" else []
    except Exception as e:
        return []

terms = get_hot_search(time_range="today")
print("🔍 今日热搜：")
for t in terms[:10]:
    trend_icon = "🔺" if t['trend'] == "up" else "🔻" if t['trend'] == "down" else "➡️"
    print(f"{t['rank']}. {t['term']} (搜索{t['count']}次) {trend_icon}")
```

## Hermes Agent 常见任务示例

### 1. AI热评 + 热搜综合助手
```python
"""
综合展示 AI 热评和热搜词
"""
import requests
import os

def hot_news_and_search_agent() -> str:
    """
    获取并展示 AI 热评和今日热搜
    
    Returns:
        格式化的综合信息
    """
    NINCHAT_BASE_URL = os.getenv("ninchat_base_url", "https://www.ninchat.cn")
    
    output = "📱 ninchat 热点看板\n"
    output += "=" * 40 + "\n\n"
    
    # 1. AI 热评
    try:
        ai_resp = requests.get(
            f"{NINCHAT_BASE_URL}/api/ai-hot-comments",
            params={"limit": 6, "days": 7},
            timeout=30
        )
        ai_data = ai_resp.json()
        if ai_data.get("status") == "ok":
            output += "🤖 AI 热评\n"
            for i, c in enumerate(ai_data["comments"][:4], 1):
                output += f"{i}. {c['title']}\n"
                if c.get('ai_comment'):
                    output += f"   💡 {c['ai_comment'][:100]}...\n\n"
    except Exception:
        pass
    
    # 2. 今日热搜
    try:
        search_resp = requests.get(
            f"{NINCHAT_BASE_URL}/api/hot-search",
            params={"limit": 10, "time_range": "today"},
            timeout=10
        )
        search_data = search_resp.json()
        if search_data.get("status") == "ok":
            output += "🔍 今日热搜\n"
            for t in search_data["terms"][:8]:
                trend = "↑" if t['trend'] == "up" else "↓" if t['trend'] == "down" else "→"
                output += f"{t['rank']}. {t['term']} ({t['count']}次) {trend}\n"
    except Exception:
        pass
    
    return output

print(hot_news_and_search_agent())
```

### 2. 实时新闻助手
```python
"""
实时新闻搜索与摘要生成
"""
import requests
import os

def realtime_news_agent(topic: str = "") -> str:
    """
    搜索实时新闻并生成摘要
    
    Args:
        topic: 可选主题，空则返回综合新闻
    
    Returns:
        新闻摘要
    """
    NINCHAT_BASE_URL = os.getenv("ninchat_base_url", "https://www.ninchat.cn")
    NINCHAT_API_KEY = os.getenv("ninchat_api_key", "")
    
    query = topic if topic else "最新重要新闻"
    
    try:
        resp = requests.post(
            f"{NINCHAT_BASE_URL}/api/search",
            json={
                "query": query,
                "detail": True,
                "limit": 5,
                "api_key": NINCHAT_API_KEY
            },
            timeout=30
        )
        data = resp.json()
        
        if data.get("status") != "ok":
            return "获取新闻失败"
        
        output = f"📰 {query if topic else '今日要闻'}：\n\n"
        for i, item in enumerate(data["results"], 1):
            output += f"{i}. {item['title']}\n"
            output += f"   {item['url']}\n"
            output += f"   摘要: {item['content'][:250]}...\n\n"
        
        return output
    except Exception as e:
        return f"出错: {str(e)}"

print(realtime_news_agent("人工智能"))
```

### 3. AI热评深度追踪
```python
"""
基于 AI 热评进行深入分析
"""
import requests
import os

def ai_comment_deep_dive() -> str:
    """
    获取 AI 热评并对第一条进行深入搜索
    
    Returns:
        深度分析报告
    """
    NINCHAT_BASE_URL = os.getenv("ninchat_base_url", "https://www.ninchat.cn")
    NINCHAT_API_KEY = os.getenv("ninchat_api_key", "")
    
    try:
        # 1. 获取 AI 热评
        ai_resp = requests.get(
            f"{NINCHAT_BASE_URL}/api/ai-hot-comments",
            params={"days": 7, "limit": 5},
            timeout=30
        )
        ai_data = ai_resp.json()
        
        if ai_data.get("status") != "ok" or not ai_data.get("comments"):
            return "暂无 AI 热评"
        
        # 2. 对第一个热点话题进行深度搜索
        hot_topic = ai_data["comments"][0]
        search_resp = requests.post(
            f"{NINCHAT_BASE_URL}/api/search",
            json={
                "query": hot_topic["title"],
                "detail": True,
                "limit": 5,
                "api_key": NINCHAT_API_KEY
            },
            timeout=30
        )
        search_data = search_resp.json()
        
        output = f"📊 AI热评深度分析：{hot_topic['title']}\n\n"
        output += f"📰 相关报道数：{hot_topic['source_count']} 篇\n\n"
        
        if hot_topic.get('ai_comment'):
            output += f"🤖 AI 热评:\n"
            output += f"   {hot_topic['ai_comment']}\n\n"
        
        if search_data.get("status") == "ok":
            output += "🔗 详细报道：\n"
            for i, item in enumerate(search_data["results"], 1):
                output += f"\n{i}. {item['title']}\n"
                output += f"   {item['url']}\n"
                output += f"   摘要: {item['content'][:300]}...\n"
        
        return output
    except Exception as e:
        return f"分析失败: {str(e)}"

print(ai_comment_deep_dive())
```

### 4. 热点事件追踪
```python
"""
追踪特定事件的最新报道
"""
import requests
import os
from datetime import datetime

def track_event(keyword: str, hours: int = 24) -> str:
    """
    追踪特定事件的最新报道
    
    Args:
        keyword: 事件关键词
        hours: 追溯时间（小时）
    
    Returns:
        事件追踪报告
    """
    NINCHAT_BASE_URL = os.getenv("ninchat_base_url", "https://www.ninchat.cn")
    NINCHAT_API_KEY = os.getenv("ninchat_api_key", "")
    
    try:
        resp = requests.post(
            f"{NINCHAT_BASE_URL}/api/search",
            json={
                "query": keyword,
                "detail": True,
                "limit": 10,
                "api_key": NINCHAT_API_KEY
            },
            timeout=30
        )
        data = resp.json()
        
        if data.get("status") != "ok":
            return "追踪失败"
        
        output = f"🔍 关于「{keyword}」的最新报道（过去{hours}小时）：\n\n"
        output += f"找到 {data['total']} 篇相关报道\n\n"
        
        for i, item in enumerate(data["results"][:8], 1):
            date_str = item.get("crawled_date", "")
            output += f"{i}. [{date_str}] {item['title']}\n"
            output += f"   {item['url']}\n\n"
        
        return output
    except Exception as e:
        return f"出错: {str(e)}"

print(track_event("科技发布会"))
```

### 5. 多主题批量搜索
```python
"""
批量搜索多个主题并汇总
"""
import requests
import os

def batch_search(topics: list) -> dict:
    """
    批量搜索多个主题
    
    Args:
        topics: 主题列表
    
    Returns:
        各主题结果汇总
    """
    NINCHAT_BASE_URL = os.getenv("ninchat_base_url", "https://www.ninchat.cn")
    NINCHAT_API_KEY = os.getenv("ninchat_api_key", "")
    
    results = {}
    for topic in topics:
        try:
            resp = requests.post(
                f"{NINCHAT_BASE_URL}/api/search",
                json={
                    "query": topic,
                    "limit": 3,
                    "api_key": NINCHAT_API_KEY
                },
                timeout=30
            )
            data = resp.json()
            if data.get("status") == "ok":
                results[topic] = {
                    "total": data["total"],
                    "top_results": data["results"]
                }
        except Exception as e:
            results[topic] = {"error": str(e)}
    
    return results

# 使用示例
topics = ["科技", "金融", "体育", "国际"]
summary = batch_search(topics)
for topic, info in summary.items():
    if "error" in info:
        print(f"{topic}: 搜索失败")
    else:
        print(f"{topic}: {info['total']} 条结果")
```

## 搜索配置说明

- **搜索引擎**: Meilisearch
- **匹配模式**:
  - `exact`（默认）— 短语搜索，关键词必须连续按序出现
  - `all` — 所有查询词都必须出现在文档中（`matchingStrategy: "all"`），可乱序可间隔
  - `fuzzy` — 至少一个词匹配，最宽松，有拼写容错
- **搜索字段**: `title` + `content`
- **排序规则**: `exactness > words > typo > proximity > attribute > sort`
- **默认结果数**: 50 条
- **最大内容长度**: 单文档 50,000 字符
- **覆盖范围**: 中央官媒、综合新闻、财经、科技、体育等 50+ 主流媒体

## 常见问题

**Q: API 请求超时怎么办？**
A: 建议设置 30 秒超时，如遇超时可重试 1-2 次。

**Q: 搜索不到想要的内容？**
A: 尝试更精确的关键词，或减少限定词。

**Q: 如何知道索引状态？**
A: 调用 /api/indexed 接口查看文档总数和索引状态。
