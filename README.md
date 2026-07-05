# ninchat Skills

Skills and integration guides for the ninchat real-time information retrieval system.

## Contents

- [hermes-agent](hermes-agent/SKILL.md) — ninchat skill for **Hermes Agent** (general AI agent framework)
- [openclaw](openclaw/SKILL.md) — ninchat skill for **OpenClaw** (multi-platform chatbot/agent framework)

## About ninchat

ninchat is a real-time information retrieval system and search infrastructure for the AI era. It provides:

- Full-text search across 50+ Chinese media outlets ([Meilishard](https://github.com/nineinfra/meilishard)-powered)
- Hot news aggregation with topic grouping
- AI-generated hot comments
- Hot search term statistics
- Multiple match modes (exact, all terms, fuzzy)

### User Permissions

| User Type | Search Level | Rate Limit | Max Results |
|-----------|-------------|------------|-------------|
| **Anonymous** | basic (snippet) | 10/min | 50 |
| **Regular** | basic (snippet) | 60/min | 100 |
| **Group User** | **full (full text)** | 120/min | 200 |

💬 **Group Users**: Scan the QR code below to join the WeChat group and get full search access (full article content), higher rate limits, and more results.

![WeChat Group QR Code](wechat_group.png)

## Usage

1. Obtain an API key from [ninchat](https://ninchat.cpolar.top)
2. Follow the skill guide for your platform (Hermes Agent or OpenClaw)
3. Configure the environment variables and start searching

## License

MIT
