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

### Helper Functions (Python) — Verified Working (2026-05-04)

Field name MUST be  (NOT ).
Empty blocks (type 2 with empty elements) cause error 1770001 — use single space instead.



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

### Block Types (Verified 2026-05-04)

| Block | Type | Status |
|-------|------|--------|
| Page (root) | 1 | ✅ |
| Text/Paragraph | 2 | ✅ |
| Heading 1-3 | 3-5 | ✅ |
| Table Cell | 32 | Auto-generated |
| Table | 31 | ✅ See below |
| Divider | 19 | ❌ API bug |
| Empty block | 2+elements:[] | ❌ API bug |

### Text Styles (text_element_style)

| Style | Field |
|-------|-------|
| Bold | `{bold: True}` |
| Italic | `{italic: True}` |
| Underline | `{underline: True}` |
| Strikethrough | `{strikethrough: True}` |
| Inline code | `{inline_code: True}` |
| Highlight | `{background_color: N}` |
| Link | `{link: {url: '...'}}` |

### Paragraph Styles (text block style)

| Style | Value |
|-------|-------|
| Bullet list | `{list: {type:'bullet', indentLevel:1}}` |
| Numbered list | `{list: {type:'number', indentLevel:1, number:N}}` |
| Block quote | `{quote: True}` |

### Divider Workaround

Block type 19 returns 1770001 on create. API can READ dividers but not CREATE them.
Use a text block with dash characters: `text('──────────')`

### Table Workflow (type 31, VERIFIED)

1. Create table block: Feishu auto-creates NxM type-32 cell blocks
2. Each cell contains one empty text block (type 2)
3. PATCH the inner text blocks with content

```python
table = {'block_type':31,'table':{'property':{'row_size':4,'column_size':4}}}
r = POST(f'{API}/documents/{DOC}/blocks/{DOC}/children', json={'children':[table]})
table_id = r.json()['data']['children'][0]['block_id']

# Get cell blocks, then their inner text blocks
cells = GET(f'{API}/documents/{DOC}/blocks/{table_id}/children').json()['data']['items']

for i, cell in enumerate(cells):
    text_bid = GET(f'{API}/documents/{DOC}/blocks/{cell["block_id"]}/children').json()['data']['items'][0]['block_id']
    PATCH(f'{API}/documents/{DOC}/blocks/{text_bid}', json={
        'update_text_elements': {'elements': [{'text_run': {
            'content': row_data[i],
            'text_element_style': {'bold': True} if is_header else {}
        }}]}
    })
```

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
