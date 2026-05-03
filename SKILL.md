---
name: feishu
description: |
  Feishu/Lark document operations skill. Create wiki-attached documents with rich content,
  search knowledge bases, grant permissions. Used as foundation by other skills that
  need to write to Feishu.
  Triggers: any task involving creating/editing Feishu docs or wiki operations.
author: Claude Code
version: 1.0.0
date: 2026-05-04
allowed-tools:
  - Bash
  - Read
  - Write
  - Grep
  - mcp__feishu__wiki_v2_space_list
  - mcp__feishu__wiki_v1_node_search
  - mcp__feishu__wiki_v2_spaceNode_list
  - mcp__feishu__wiki_v2_spaceNode_create
  - mcp__feishu__wiki_v2_space_getNode
  - mcp__feishu__docx_v1_document_rawContent
  - mcp__feishu__drive_v1_permissionMember_create
  - mcp__feishu__im_v1_message_create
---

# Feishu — 飞书文档操作

Foundation skill for Feishu wiki document operations. Used by `job-tracker`
and other skills that need to create documents in Feishu knowledge bases.

## Architecture

```
caller skill (e.g. job-tracker)
       │
       ▼
feishu.skill
  ├── find_space(name) → space_id
  ├── find_folder(space_id, name) → parent_node_token
  ├── create_doc(title, parent, markdown) → doc_url
  └── grant_access(doc_token, user_id)
```

## Key Limitation

MCP can create empty wiki nodes but cannot write content into them.
The Feishu Block API is used (via Python) to fill the docx with content.

**Always use Python** for Block API calls — bash/curl corrupts UTF-8 Chinese text.

---

## Workflow: Create Wiki Document with Content

### Step 1: Locate Target Folder

Search for the wiki space and folder:

```
wiki_v1_node_search(space_id, query="{folder_name}")
→ returns node list with node_id and parent_id

wiki_v2_spaceNode_list(space_id, parent_node_token)
→ returns children, find the target folder
```

Common space IDs stored in caller's profile:
- `7635789242190269391` → 秋招大战

### Step 2: Create Empty Wiki Node

Use MCP `wiki_v2_spaceNode_create`:
```
{
  "obj_type": "docx",
  "node_type": "origin",
  "parent_node_token": "{folder_node_token}",
  "title": "{document_title}"
}
→ returns: { node_token, obj_token }
```

### Step 3: Fill Content via Python Block API

Run Python script with the `obj_token` from Step 2:

```python
import json, requests

USER_TOKEN = json.load(open('~/.claude/feishu_tokens.json'))['token']
DOC = "{obj_token}"

# Helper functions for block construction
def h1(s): return {'block_type':3,'heading1':{'elements':[{'text_run':{'content':s,'text_style':{}}}]}}
def h2(s): return {'block_type':4,'heading2':{'elements':[{'text_run':{'content':s,'text_style':{}}}]}}
def text(s): return {'block_type':2,'text':{'elements':[{'text_run':{'content':s,'text_style':{}}}]}}
def bullet(s): return {'block_type':2,'text':{'elements':[{'text_run':{'content':s,'text_style':{}}}],'style':{'list':{'type':'bullet','indentLevel':1}}}}
def bold_bullet(label, body):
    return {'block_type':2,'text':{'elements':[
        {'text_run':{'content':label,'text_style':{'bold':True}}},
        {'text_run':{'content':body,'text_style':{}}}
    ],'style':{'list':{'type':'bullet','indentLevel':1}}}}

blocks = [
    h1('Title'),
    h2('Section'),
    text('content...'),
    bullet('item 1'),
    bold_bullet('Bold label',' details'),
]

requests.post(
    f'https://open.feishu.cn/open-apis/docx/v1/documents/{DOC}/blocks/{DOC}/children',
    json={'children': blocks},
    headers={'Authorization': f'Bearer {USER_TOKEN}'})
```

## Block Format Reference (Complete)

### Field Name Correction
The text style field is `text_element_style` (NOT `text_style`).

### Helper Functions (Python)

```python
def h1(s): return {'block_type':3,'heading1':{'elements':[{'text_run':{'content':s,'text_element_style':{}}}]}}
def h2(s): return {'block_type':4,'heading2':{'elements':[{'text_run':{'content':s,'text_element_style':{}}}]}}
def h3(s): return {'block_type':5,'heading3':{'elements':[{'text_run':{'content':s,'text_element_style':{}}}]}}
def text(s): return {'block_type':2,'text':{'elements':[{'text_run':{'content':s,'text_element_style':{}}}]}}
def empty(): return {'block_type':2,'text':{'elements':[],'style':{}}}

def bullet(s):
    return {'block_type':2,'text':{'elements':[{'text_run':{'content':s,'text_element_style':{}}}],'style':{'list':{'type':'bullet','indentLevel':1}}}}

def number_list(s, n):
    return {'block_type':2,'text':{'elements':[{'text_run':{'content':s,'text_element_style':{}}}],'style':{'list':{'type':'number','indentLevel':1,'number':n}}}}

def bold(text): return {'text_run':{'content':text,'text_element_style':{'bold':True}}}
def italic(text): return {'text_run':{'content':text,'text_element_style':{'italic':True}}}
def link(url, text): return {'text_run':{'content':text,'text_element_style':{'link':{'url':url}}}}
def code(text): return {'text_run':{'content':text,'text_element_style':{'inline_code':True}}}
def highlight(text): return {'text_run':{'content':text,'text_element_style':{'background_color':2}}}

def quote(s):
    return {'block_type':2,'text':{'elements':[{'text_run':{'content':s,'text_element_style':{}}}],'style':{'quote':True}}}
```

### Supported Block Types

| Block | Type | Status |
|-------|------|--------|
| Page (root) | 1 | ✅ |
| Text/Paragraph | 2 | ✅ |
| Heading 1 | 3 | ✅ |
| Heading 2 | 4 | ✅ |
| Heading 3 | 5 | ✅ (H4-H9: types 6-11, untested) |
| Divider | 19 | ❌ API error 1770001 |

### Supported Text Styles

| Style | Field | Example |
|-------|-------|---------|
| Bold | `bold: true` | **text** |
| Italic | `italic: true` | *text* |
| Underline | `underline: true` | __text__ |
| Strikethrough | `strikethrough: true` | ~~text~~ |
| Inline code | `inline_code: true` | `code` |
| Background color | `background_color: N` | highlight |
| Link | `link: {url: '...'}` | [click](url) |

### Supported Paragraph Styles

| Style | Value | Example |
|-------|-------|---------|
| Bullet list | `style.list: {type:'bullet',indentLevel:1}` | * item |
| Numbered list | `style.list: {type:'number',indentLevel:1,number:1}` | 1. item |
| Task list | `style.list: {type:'checkBox',indentLevel:1}` | ☐ todo |
| Block quote | `style.quote: true` | > quoted |
| Alignment | `style.align: 'left'/'right'/'center'` | |

### Tables (type 31)

Table cells are child block_id references, not inline content. Multi-step:
1. Create table block with `property: {row_size, column_size}`
2. Create child text blocks for each cell
3. Reference child block_ids in table.cells array

Complex — use bullets as alternative for job listings.

### Step 4: Grant Permission

```
drive_v1_permissionMember_create(
  token: {obj_token},
  member_type: "openid",
  member_id: "ou_6f718f5c13058092668bf236eba2e2dd",
  perm: "full_access",
  type: "docx"
)
```

### Step 5: Return Result

```
Success:
  doc_url: https://fcnmqmoj81j2.feishu.cn/wiki/{node_token}
  blocks_written: N
  permission: full_access

Failure:
  error_code + description
  suggested fix
```

---

## Workflow: Find Wiki Space by Name

```
wiki_v2_space_list(useUAT=true)
→ search for matching space name
→ return space_id
```

Note: Requires valid UAT (user_access_token). If `Authentication token expired`:
1. `python ~/.claude/feishu_oauth_write.py` → re-authorize
2. Restart Claude Code

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| UAT expired | Run feishu_oauth_write.py, restart |
| Block API error 1770001 | Check block type — type 9/10/19 not supported |
| Bash writes garbled Chinese | Use Python for Block API (not bash/curl) |
| Permission denied on wiki node | Grant app `full_access` in Feishu UI |
