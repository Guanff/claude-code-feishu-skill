# Feishu — Claude Code Skill for Feishu Document Operations

A reusable foundation skill for creating wiki-attached documents in Feishu/Lark
knowledge bases with rich content, via Claude Code.

## What It Does

- Finds wiki spaces and folders by name
- Creates documents under any wiki folder with formatted content
- Writes rich text: headings, bold, bullet lists, links, emoji
- Grants full_access permissions automatically
- Returns document URL on success

## Why a Separate Skill

The Feishu MCP tools can create empty wiki nodes but cannot write content into
them. This skill bridges the gap using the Feishu Block API (via Python) to fill
documents with properly formatted UTF-8 content.

## Prerequisites

- Claude Code with Feishu MCP configured (`FEISHU_APP_ID`, `FEISHU_APP_SECRET`)
- Valid user_access_token (if expired, run `feishu_oauth_write.py` and restart)
- Python 3 with `requests` library

## Quick Start

```bash
cp -r feishu ~/.claude/skills/feishu
```

Then in any skill that needs Feishu docs:

```
Delegate to feishu.skill:
  1. Locate target folder by name in the wiki space
  2. Create empty wiki node via MCP
  3. Fill content via Python Block API
  4. Grant full_access to specified user
  5. Return document URL
```

## Block Format Reference

| Format | Implementation |
|--------|---------------|
| H1 heading | `block_type: 3` + emoji |
| H2 heading | `block_type: 4` |
| Bold text | `text_style: {bold: true}` on text_run |
| Bullet list | text block_type: 2 + `style.list.type: 'bullet'` |
| Link | `text_style: {link: {url: '...'}}` |
| Plain text | `block_type: 2` |

## Known Limitations

- Divider (block_type 19) not supported by Block API
- Table blocks require complex cell format — under investigation
- Bash/curl corrupts Chinese UTF-8 — always use Python for Block API
- Cannot delete wiki nodes via API

## License

MIT
