---
name: ninchat
description: |
  实时信息检索系统，AI时代的搜索基础设施。
  为 OpenClaw 提供实时新闻、深度内容、事件追踪、热点新闻聚合、AI热评、热搜统计等能力，支持 CLI、Telegram、Discord、Slack、WhatsApp、Signal 等多平台。
version: 1.2.0
platforms: [cli, telegram, discord, slack, whatsapp, signal]
author: NineInfra
category: Search & Information
---

# ninchat Search API — OpenClaw 专用

实时信息检索系统，AI时代的搜索基础设施。

## 主要功能

- **搜索 API**: 基于 Meilisearch 的全文检索，支持 50+ 主流媒体
- **热点新闻**: 基于爬取内容聚合，多源相似话题合并
- **AI热评**: 基于热点新闻使用LLM生成深度评论
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

### 在 OpenClaw 中配置

**方式一：通过 .env 文件（推荐）**

在 OpenClaw 的 `.env` 文件中添加：
```bash
# ninchat 搜索 API 配置
ninchat_base_url=https://www.ninchat.cn
ninchat_api_key=sk_ninso_xxxxx
```

**方式二：通过 OpenClaw 配置面板**

1. 打开 OpenClaw 的配置界面
2. 找到 **Skills → ninchat**
3. 填入 `Base URL` 和 `API Key`
4. 保存并重启 OpenClaw

## 连接信息

- **Base URL**: `ninchat_base_url`（默认 `https://www.ninchat.cn`）
- **认证方式**: API Key 认证（`sk_ninso_` 开头）
- **调用方式**: Python `requests` / `httpx`，或 OpenClaw 的内置工具
- **数据格式**: JSON
- **默认超时**: 30秒
- **支持平台**: CLI、Telegram、Discord、Slack、WhatsApp、Signal

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

### OpenClaw 专用代码示例

#### 基础搜索函数
```python
"""
OpenClaw 平台通用搜索函数
支持多平台响应格式化
"""
import requests
import os
from typing import Dict, List, Optional

def search_ninchat(query: str, detail: bool = False, limit: int = 10) -> Dict:
    """
    使用 ninchat API 搜索实时信息
    
    Args:
        query: 搜索关键词
        detail: 是否返回完整内容
        limit: 返回结果数量
    
    Returns:
        搜索结果字典
    """
    NINCHAT_BASE_URL = os.getenv("ninchat_base_url", "https://www.ninchat.cn")
    NINCHAT_API_KEY = os.getenv("ninchat_api_key", "")
    
    if not NINCHAT_API_KEY:
        return {
            "status": "error",
            "message": "请先配置 ninchat_api_key 环境变量"
        }
    
    try:
        resp = requests.post(
            f"{NINCHAT_BASE_URL}/api/search",
            json={
                "query": query,
                "detail": detail,
                "limit": limit,
                "api_key": NINCHAT_API_KEY
            },
            timeout=30
        )
        resp.raise_for_status()
        return resp.json()
    except requests.exceptions.Timeout:
        return {
            "status": "error",
            "message": "请求超时，请稍后重试"
        }
    except requests.exceptions.RequestException as e:
        return {
            "status": "error",
            "message": f"请求失败: {str(e)}"
        }
    except Exception as e:
        return {
            "status": "error",
            "message": f"未知错误: {str(e)}"
        }
```

#### 多平台响应格式化
```python
"""
为不同平台格式化搜索结果
"""
from typing import Dict

def format_for_telegram(results: List[Dict]) -> str:
    """
    Telegram 平台格式化输出（Markdown）
    
    Args:
        results: 搜索结果列表
    
    Returns:
        格式化后的字符串
    """
    if not results:
        return "未找到相关结果"
    
    output = "📰 *搜索结果*：\n\n"
    for i, item in enumerate(results[:5], 1):
        output += f"*{i}*. [{item['title']}]({item['url']})\n"
        if 'content' in item:
            content = item['content'][:200]
            output += f"   {content}...\n\n"
        elif 'snippet' in item:
            output += f"   {item['snippet']}\n\n"
    
    return output

def format_for_discord(results: List[Dict]) -> str:
    """
    Discord 平台格式化输出（Markdown）
    
    Args:
        results: 搜索结果列表
    
    Returns:
        格式化后的字符串
    """
    if not results:
        return "未找到相关结果"
    
    output = "📰 **搜索结果**：\n\n"
    for i, item in enumerate(results[:5], 1):
        output += f"**{i}**. [{item['title']}]({item['url']})\n"
        if 'content' in item:
            content = item['content'][:200]
            output += f"   {content}...\n\n"
        elif 'snippet' in item:
            output += f"   {item['snippet']}\n\n"
    
    return output

def format_for_slack(results: List[Dict]) -> List[Dict]:
    """
    Slack 平台格式化输出（Block Kit）
    
    Args:
        results: 搜索结果列表
    
    Returns:
        Slack Block Kit 格式的列表
    """
    if not results:
        return [
            {
                "type": "section",
                "text": {
                    "type": "mrkdwn",
                    "text": "未找到相关结果"
                }
            }
        ]
    
    blocks = [
        {
            "type": "header",
            "text": {
                "type": "plain_text",
                "text": "📰 搜索结果",
                "emoji": True
            }
        }
    ]
    
    for i, item in enumerate(results[:5], 1):
        content = ""
        if 'content' in item:
            content = item['content'][:150] + "..."
        elif 'snippet' in item:
            content = item['snippet']
        
        blocks.extend([
            {
                "type": "section",
                "text": {
                    "type": "mrkdwn",
                    "text": f"*{i}*. <{item['url']}|{item['title']}>\n{content}"
                }
            },
            {
                "type": "divider"
            }
        ])
    
    if blocks:
        blocks.pop()  # 移除最后一个 divider
    
    return blocks

def format_for_cli(results: List[Dict]) -> str:
    """
    CLI 平台格式化输出（纯文本）
    
    Args:
        results: 搜索结果列表
    
    Returns:
        格式化后的字符串
    """
    if not results:
        return "未找到相关结果"
    
    output = "=== 搜索结果 ===\n\n"
    for i, item in enumerate(results[:5], 1):
        output += f"{i}. {item['title']}\n"
        output += f"   {item['url']}\n"
        if 'content' in item:
            output += f"   {item['content'][:250]}...\n"
        elif 'snippet' in item:
            output += f"   {item['snippet']}\n"
        output += "\n"
    
    return output
```

#### OpenClaw 完整集成示例
```python
"""
OpenClaw 技能集成完整示例
"""
import requests
import os

def ninchat_search_skill(query: str, platform: str = "cli") -> str:
    """
    OpenClaw 搜索技能主入口
    
    Args:
        query: 搜索关键词
        platform: 平台类型 (cli, telegram, discord, slack)
    
    Returns:
        格式化后的响应
    """
    NINCHAT_BASE_URL = os.getenv("ninchat_base_url", "https://www.ninchat.cn")
    NINCHAT_API_KEY = os.getenv("ninchat_api_key", "")
    
    if not NINCHAT_API_KEY:
        return "⚠️ 请先配置 ninchat API Key"
    
    try:
        # 搜索并获取完整内容
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
            return "❌ 搜索失败，请稍后重试"
        
        results = data.get("results", [])
        if not results:
            return f"🔍 未找到关于 '{query}' 的相关信息"
        
        # 根据平台选择格式化方式
        return format_response(results, query, platform)
    except Exception as e:
        return f"⚠️ 出错了: {str(e)}"

def format_response(results: list, query: str, platform: str) -> str:
    """
    根据不同平台格式化响应
    """
    if platform == "telegram":
        return format_for_telegram(results)
    elif platform == "discord":
        return format_for_discord(results)
    elif platform == "slack":
        # Slack 返回 JSON 字符串
        import json
        return json.dumps({"blocks": format_for_slack(results)})
    else:
        return format_for_cli(results)

# 调用示例
if __name__ == "__main__":
    # CLI 模式
    print(ninchat_search_skill("最新科技新闻", platform="cli"))
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

### OpenClaw 专用代码示例

#### 热点新闻基础函数
```python
"""
获取热点新闻的通用函数
"""
import requests
import os
from typing import Dict, List

def get_hot_news(limit: int = 10, days: int = 7) -> Dict:
    """
    获取热点新闻
    
    Args:
        limit: 返回新闻数量
        days: 统计天数
    
    Returns:
        热点新闻结果字典
    """
    NINCHAT_BASE_URL = os.getenv("ninchat_base_url", "https://www.ninchat.cn")
    
    try:
        resp = requests.get(
            f"{NINCHAT_BASE_URL}/api/hot-news",
            params={"limit": limit, "days": days},
            timeout=15
        )
        return resp.json()
    except Exception as e:
        return {"status": "error", "message": str(e)}
```

#### 热点新闻多平台格式化
```python
"""
热点新闻多平台响应格式化
"""
from typing import Dict, List

def format_hot_news_for_telegram(news: List[Dict]) -> str:
    """
    Telegram 平台格式化输出（Markdown）
    """
    if not news:
        return "🔍 暂无热点新闻"
    
    output = "🔥 *热点新闻*：\n\n"
    for i, item in enumerate(news[:8], 1):
        output += f"*{i}*. {item['title']}\n"
        output += f"   📰 {item['source_count']} 个来源\n"
        if item['sources']:
            sources = ", ".join([s['name'] for s in item['sources'][:3]])
            output += f"   📌 {sources}\n"
        output += "\n"
    
    return output

def format_hot_news_for_cli(news: List[Dict]) -> str:
    """
    CLI 平台格式化输出（纯文本）
    """
    if not news:
        return "暂无热点新闻"
    
    output = "=== 热点新闻 ===\n\n"
    for i, item in enumerate(news[:8], 1):
        output += f"{i}. {item['title']}\n"
        output += f"   来源数: {item['source_count']}\n"
        if item['sources']:
            sources = ", ".join([s['name'] for s in item['sources'][:3]])
            output += f"   来源: {sources}\n"
        output += "\n"
    
    return output

def format_hot_news_for_slack(news: List[Dict]) -> List[Dict]:
    """
    Slack 平台格式化输出（Block Kit）
    """
    if not news:
        return [{
            "type": "section",
            "text": {"type": "mrkdwn", "text": "🔍 暂无热点新闻"}
        }]
    
    blocks = [{
        "type": "header",
        "text": {"type": "plain_text", "text": "🔥 热点新闻", "emoji": True}
    }]
    
    for i, item in enumerate(news[:8], 1):
        sources_text = ""
        if item['sources']:
            sources = ", ".join([s['name'] for s in item['sources'][:3]])
            sources_text = f"\n📌 {sources}"
        
        blocks.extend([{
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": f"*{i}*. {item['title']}\n📰 {item['source_count']} 个来源{sources_text}"
            }
        }, {
            "type": "divider"
        }])
    
    if len(blocks) > 1:
        blocks.pop()
    
    return blocks
```

## AI热评 — GET /api/ai-hot-comments

获取基于热点新闻使用LLM生成的深度评论。

**Method**: GET

### 请求参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `limit` | integer | 否 | 10 | 返回AI热评数量（1-50） |
| `days` | integer | 否 | 7 | 统计最近几天的新闻（1-30） |

### Request 示例

```bash
curl -s "https://www.ninchat.cn/api/ai-hot-comments?limit=5&days=7"
```

### Response 格式

```json
{
  "status": "ok",
  "comments": [
    {
      "hot_group_id": 1,
      "title": "热点新闻标题",
      "source_count": 5,
      "comment": "LLM生成的深度评论内容",
      "status": "ready",
      "generated_at": "2026-06-02T10:00:00",
      "sources": [
        {"name": "来源1", "url": "https://..."},
        {"name": "来源2", "url": "https://..."}
      ]
    }
  ],
  "count": 5,
  "days": 7
}
```

### OpenClaw 专用代码示例

```python
"""
获取AI热评
"""
import requests
import os
from typing import Dict, List

def get_ai_hot_comments(limit: int = 10, days: int = 7) -> Dict:
    """
    获取AI热评
    
    Args:
        limit: 返回评论数量
        days: 统计天数
    
    Returns:
        AI热评结果字典
    """
    NINCHAT_BASE_URL = os.getenv("ninchat_base_url", "https://www.ninchat.cn")
    
    try:
        resp = requests.get(
            f"{NINCHAT_BASE_URL}/api/ai-hot-comments",
            params={"limit": limit, "days": days},
            timeout=15
        )
        return resp.json()
    except Exception as e:
        return {"status": "error", "message": str(e)}

def format_ai_comments_for_telegram(comments: List[Dict]) -> str:
    """格式化AI热评用于 Telegram"""
    if not comments:
        return "🤖 暂无AI热评"
    
    output = "🤖 *AI热评*：\n\n"
    for i, item in enumerate(comments[:5], 1):
        if item['status'] == 'ready':
            output += f"*{i}*. {item['title']}\n"
            output += f"   💬 {item['comment'][:300]}...\n\n"
        elif item['status'] == 'generating':
            output += f"*{i}*. {item['title']}\n"
            output += f"   ⏳ 评论生成中...\n\n"
        else:
            output += f"*{i}*. {item['title']}\n"
            output += f"   ❌ 评论生成失败\n\n"
    
    return output

def format_ai_comments_for_cli(comments: List[Dict]) -> str:
    """格式化AI热评用于 CLI"""
    if not comments:
        return "暂无AI热评"
    
    output = "=== AI热评 ===\n\n"
    for i, item in enumerate(comments[:5], 1):
        output += f"{i}. {item['title']}\n"
        if item['status'] == 'ready':
            output += f"   评论: {item['comment'][:400]}...\n"
        elif item['status'] == 'generating':
            output += f"   状态: 生成中...\n"
        else:
            output += f"   状态: 生成失败\n"
        output += f"   来源数: {item['source_count']}\n\n"
    
    return output

def format_ai_comments_for_slack(comments: List[Dict]) -> List[Dict]:
    """格式化AI热评用于 Slack"""
    if not comments:
        return [{
            "type": "section",
            "text": {"type": "mrkdwn", "text": "🤖 暂无AI热评"}
        }]
    
    blocks = [{
        "type": "header",
        "text": {"type": "plain_text", "text": "🤖 AI热评", "emoji": True}
    }]
    
    for i, item in enumerate(comments[:5], 1):
        comment_text = ""
        if item['status'] == 'ready':
            comment_text = f"💬 {item['comment'][:250]}..."
        elif item['status'] == 'generating':
            comment_text = "⏳ 评论生成中..."
        else:
            comment_text = "❌ 评论生成失败"
        
        blocks.extend([{
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": f"*{i}*. {item['title']}\n{comment_text}\n📰 {item['source_count']} 个来源"
            }
        }, {
            "type": "divider"
        }])
    
    if len(blocks) > 1:
        blocks.pop()
    
    return blocks
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

### OpenClaw 专用代码示例

```python
"""
获取热搜词
"""
import requests
import os
from typing import Dict, List

def get_hot_search(limit: int = 20, time_range: str = "today") -> Dict:
    """
    获取热搜词
    
    Args:
        limit: 返回词数量
        time_range: 时间范围
    
    Returns:
        热搜词结果字典
    """
    NINCHAT_BASE_URL = os.getenv("ninchat_base_url", "https://www.ninchat.cn")
    
    try:
        resp = requests.get(
            f"{NINCHAT_BASE_URL}/api/hot-search",
            params={"limit": limit, "time_range": time_range},
            timeout=15
        )
        return resp.json()
    except Exception as e:
        return {"status": "error", "message": str(e)}

def format_hot_search_for_telegram(terms: List[Dict]) -> str:
    """格式化热搜词用于 Telegram"""
    if not terms:
        return "🔍 暂无热搜词"
    
    output = "🔍 *今日热搜*：\n\n"
    for term in terms[:10]:
        trend_icon = "🔺" if term['trend'] == "up" else "🔻" if term['trend'] == "down" else "➡️"
        output += f"{term['rank']}. {term['term']} ({term['count']} 次) {trend_icon}\n"
    
    return output

def format_hot_search_for_cli(terms: List[Dict]) -> str:
    """格式化热搜词用于 CLI"""
    if not terms:
        return "暂无热搜词"
    
    output = "=== 今日热搜 ===\n\n"
    for term in terms[:10]:
        trend = "↑" if term['trend'] == "up" else "↓" if term['trend'] == "down" else "→"
        output += f"{term['rank']}. {term['term']} (搜索 {term['count']} 次) {trend}\n"
    
    return output
```

## OpenClaw 各平台示例

### 1. AI热评 + 热搜综合看板
```python
"""
OpenClaw 热点看板集成示例
同时展示AI热评、热点新闻和热搜词
"""
def ninchat_hotboard_skill(platform: str = "cli") -> str:
    """
    热点看板主入口
    """
    # 获取AI热评
    ai_comments_result = get_ai_hot_comments(limit=5, days=7)
    ai_comments = ai_comments_result.get("comments", []) if ai_comments_result.get("status") == "ok" else []
    
    # 获取热点新闻
    news_result = get_hot_news(limit=6, days=7)
    news = news_result.get("news", []) if news_result.get("status") == "ok" else []
    
    # 获取热搜词
    hot_search_result = get_hot_search(limit=10, time_range="today")
    hot_terms = hot_search_result.get("terms", []) if hot_search_result.get("status") == "ok" else []
    
    # 根据平台选择格式化
    if platform == "telegram":
        return format_hotboard_for_telegram(ai_comments, news, hot_terms)
    elif platform == "slack":
        return format_hotboard_for_slack(ai_comments, news, hot_terms)
    else:
        return format_hotboard_for_cli(ai_comments, news, hot_terms)

def format_hotboard_for_telegram(ai_comments, news, hot_terms):
    """Telegram 格式化"""
    output = "📱 *ninchat 热点看板*：\n"
    output += "=" * 40 + "\n\n"
    
    if ai_comments:
        output += "🤖 *AI热评*：\n"
        for i, c in enumerate(ai_comments[:3], 1):
            output += f"{i}. {c['title']}\n"
            if c['status'] == 'ready':
                output += f"   💬 {c['comment'][:200]}...\n"
            elif c['status'] == 'generating':
                output += f"   ⏳ 生成中...\n"
            else:
                output += f"   ❌ 生成失败\n"
        output += "\n"
    
    if news:
        output += "🔥 *热点新闻*：\n"
        for i, n in enumerate(news[:4], 1):
            output += f"{i}. {n['title']}\n   📰 {n['source_count']}个来源\n\n"
    
    if hot_terms:
        output += "🔍 *今日热搜*：\n"
        for t in hot_terms[:8]:
            trend = "🔺" if t['trend'] == "up" else "🔻" if t['trend'] == "down" else "➡️"
            output += f"{t['rank']}. {t['term']} ({t['count']}次) {trend}\n"
    
    return output

def format_hotboard_for_cli(ai_comments, news, hot_terms):
    """CLI 格式化"""
    output = "=== ninchat 热点看板 ===\n"
    output += "=" * 40 + "\n\n"
    
    if ai_comments:
        output += "AI热评：\n"
        for i, c in enumerate(ai_comments[:3], 1):
            output += f"  {i}. {c['title']}\n"
            if c['status'] == 'ready':
                output += f"     评论: {c['comment'][:250]}...\n"
            elif c['status'] == 'generating':
                output += f"     状态: 生成中...\n"
            else:
                output += f"     状态: 生成失败\n"
        output += "\n"
    
    if news:
        output += "热点新闻：\n"
        for i, n in enumerate(news[:4], 1):
            output += f"  {i}. {n['title']}\n      {n['source_count']}个来源\n\n"
    
    if hot_terms:
        output += "今日热搜：\n"
        for t in hot_terms[:8]:
            trend = "↑" if t['trend'] == "up" else "↓" if t['trend'] == "down" else "→"
            output += f"  {t['rank']}. {t['term']} ({t['count']}次) {trend}\n"
    
    return output

def format_hotboard_for_slack(ai_comments, news, hot_terms):
    """Slack 格式化"""
    blocks = [{
        "type": "header",
        "text": {"type": "plain_text", "text": "📱 ninchat 热点看板", "emoji": True}
    }]
    
    if ai_comments:
        blocks.append({"type": "divider"})
        ai_comment_text = "🤖 *AI热评*：\n"
        for i, c in enumerate(ai_comments[:3], 1):
            ai_comment_text += f"{i}. {c['title']}\n"
            if c['status'] == 'ready':
                ai_comment_text += f"   💬 {c['comment'][:200]}...\n"
            elif c['status'] == 'generating':
                ai_comment_text += f"   ⏳ 生成中...\n"
            else:
                ai_comment_text += f"   ❌ 生成失败\n"
        blocks.append({
            "type": "section",
            "text": {"type": "mrkdwn", "text": ai_comment_text}
        })
    
    if news:
        blocks.append({"type": "divider"})
        blocks.append({
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": "🔥 *热点新闻*：\n" + "\n\n".join([
                    f"*{i+1}*. {n['title']}\n   📰 {n['source_count']}个来源"
                    for i, n in enumerate(news[:4])
                ])
            }
        })
    
    if hot_terms:
        blocks.append({"type": "divider"})
        blocks.append({
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": "🔍 *今日热搜*：\n" + "\n".join([
                    f"{t['rank']}. {t['term']} ({t['count']}次) {'🔺' if t['trend'] == 'up' else '🔻' if t['trend'] == 'down' else '➡️'}"
                    for t in hot_terms[:8]
                ])
            }
        })
    
    return {"blocks": blocks}

# 使用示例
if __name__ == "__main__":
    print(ninchat_hotboard_skill(platform="cli"))
```

### 2. CLI 命令行工具
```python
"""
OpenClaw CLI 模式示例
"""
def cli_tool():
    """
    CLI 模式下的搜索工具
    """
    import argparse
    
    parser = argparse.ArgumentParser(description="ninchat 搜索工具")
    parser.add_argument("query", nargs="?", help="搜索关键词，不提供则显示热点看板")
    parser.add_argument("--limit", type=int, default=5, help="结果数量")
    parser.add_argument("--detail", action="store_true", help="显示完整内容")
    parser.add_argument("--hotboard", action="store_true", help="显示热点看板")
    
    args = parser.parse_args()
    
    if args.hotboard or not args.query:
        print(ninchat_hotboard_skill(platform="cli"))
        return
    
    result = search_ninchat(args.query, args.detail, args.limit)
    
    if result.get("status") == "ok":
        print(format_for_cli(result["results"]))
    else:
        print(f"错误: {result.get('message')}")

if __name__ == "__main__":
    cli_tool()
```

### 3. Telegram Bot 集成
```python
"""
OpenClaw Telegram Bot 集成示例
"""
def telegram_bot_command(update, context):
    """
    Telegram 命令处理
    支持 /search 关键词 和 /hotboard
    """
    if context.args and context.args[0] == "hotboard":
        response = ninchat_hotboard_skill(platform="telegram")
        update.message.reply_markdown(response)
        return
    
    query = ' '.join(context.args) if context.args else ""
    
    if not query:
        update.message.reply_text("请输入搜索关键词或使用 'hotboard'，例如：/search 人工智能")
        return
    
    result = search_ninchat(query, detail=True, limit=5)
    
    if result.get("status") == "ok":
        response = format_for_telegram(result["results"])
        update.message.reply_markdown(response)
    else:
        update.message.reply_text(result.get("message", "搜索失败"))
```

### 4. Discord Bot 集成
```python
"""
OpenClaw Discord Bot 集成示例
"""
import discord

async def discord_search_command(interaction, query: str):
    """
    Discord Slash Command 处理
    """
    await interaction.response.defer()
    
    if query.lower() == "hotboard":
        response = ninchat_hotboard_skill(platform="slack")
        await interaction.followup.send(blocks=response["blocks"])
        return
    
    result = search_ninchat(query, detail=True, limit=5)
    
    if result.get("status") == "ok":
        response = format_for_discord(result["results"])
        await interaction.followup.send(response)
    else:
        await interaction.followup.send(result.get("message", "搜索失败"))
```

### 5. Slack Bot 集成
```python
"""
OpenClaw Slack Bot 集成示例
"""
from slack_bolt import App

def slack_search_command(ack, body, say):
    """
    Slack 命令处理
    """
    ack()
    query = body.get('text', '').strip()
    
    if query.lower() == "hotboard":
        response = ninchat_hotboard_skill(platform="slack")
        say(blocks=response["blocks"])
        return
    
    if not query:
        say("请输入搜索关键词或使用 'hotboard' 查看热点看板")
        return
    
    result = search_ninchat(query, detail=True, limit=5)
    
    if result.get("status") == "ok":
        blocks = format_for_slack(result["results"])
        say(blocks=blocks)
    else:
        say(result.get("message", "搜索失败"))
```

### 6. 批量搜索与报告生成
```python
"""
多主题批量搜索与汇总报告
"""
import requests
import os
from typing import Dict

def batch_search_report(topics: list) -> Dict:
    """
    批量搜索并生成报告
    
    Args:
        topics: 主题列表
    
    Returns:
        报告字典
    """
    NINCHAT_BASE_URL = os.getenv("ninchat_base_url", "https://www.ninchat.cn")
    NINCHAT_API_KEY = os.getenv("ninchat_api_key", "")
    
    report = {
        "generated_at": "",
        "topics": {},
        "summary": ""
    }
    
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
                report["topics"][topic] = {
                    "total": data["total"],
                    "top_titles": [r["title"] for r in data["results"][:3]]
                }
            else:
                report["topics"][topic] = {"error": "搜索失败"}
        except Exception as e:
            report["topics"][topic] = {"error": str(e)}
    
    # 生成摘要
    total_results = sum(
        t.get("total", 0) for t in report["topics"].values() 
        if "error" not in t
    )
    report["summary"] = f"共搜索 {len(topics)} 个主题，找到 {total_results} 条结果"
    
    return report

# 使用示例
topics = ["科技", "财经", "体育", "国际"]
report = batch_search_report(topics)
print(report)
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
- **多平台支持**: CLI、Telegram、Discord、Slack、WhatsApp、Signal

## 常见问题

**Q: 如何在不同平台获得最佳显示效果？**
A: 使用相应平台的格式化函数，Telegram/Discord 使用 Markdown，Slack 使用 Block Kit。

**Q: API 请求超时怎么办？**
A: 建议设置 30 秒超时，如遇超时可重试 1-2 次。

**Q: 搜索不到想要的内容？**
A: 尝试更精确的关键词，或减少限定词。

**Q: 如何知道索引状态？**
A: 调用 /api/indexed 接口查看文档总数和索引状态。

**Q: OpenClaw 支持哪些平台？**
A: 支持 CLI、Telegram、Discord、Slack、WhatsApp、Signal 等多个平台。
