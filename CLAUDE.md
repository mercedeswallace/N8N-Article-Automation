# Claude N8N — Flow State PM Weekly Article Publisher

## Project Overview
An N8N workflow that automatically drafts, reviews, and publishes a weekly Substack newsletter called **Flow State PM by Mercedes**. Claude AI writes the article, the draft is saved to Notion for review, an approval email is sent, and upon approval the article is published to Substack.

---

## Workflow Architecture

### Trigger Flow (Every Monday 8am)
```
Every Monday 8am
  → Pull top idea from Notion
  → Claude writes draft (Anthropic API)
  → Parse draft (Code node)
  → Save draft to Notion
  → Append a block (Notion)
  → Send draft for review (Gmail)
```

### Approval Flow (Webhook)
```
Approval webhook (GET /webhook/flowstate-approve)
  → Approved? (IF node)
    → [true]  Fetch final draft from Notion
              → Get draft blocks
              → Assemble draft body (Code node)
              → Code in JavaScript (markdown → ProseMirror)
              → Create Substack draft
              → Publish to Substack
              → Mark published in Notion
              → Success response
    → [false] Mark rejected in Notion
              → Reject response
```

---

## Node-by-Node Reference

### Pull top idea from Notion
- Resource: Database Page
- Operation: Get Many
- Limit: 1
- Filter: `property_status = "Ready"`
- Sort: Priority descending
- Output fields used: `property_name`, `property_angle`, `property_key_insight`, `property_reader_pain`

### Claude writes draft
- Type: HTTP Request node
- Method: POST
- URL: `https://api.anthropic.com/v1/messages`
- Body: Use **expression editor POPUP** (not field-level toggle) — paste pure `JSON.stringify({...})` with NO `=` prefix and NO `={{ }}` wrapper
- The `=` prefix bug: if the body shows `={"model":...}` in the request, it means the expression was set at field level, not popup level

#### Article Writing Rules (add to system prompt)
- **No em dashes** — never use `—` in any form. Use commas, colons, or rewrite the sentence instead
- **Notion is source of truth** — Claude provides a first draft; any edits made in Notion before approval are what gets published (already handled by "Assemble draft body" reading Notion blocks)
- **Include visuals** — add ASCII diagrams, simple text-based frameworks, or structured step visuals (e.g. process flows, before/after tables) to help readers digest concepts. Substack does not render Mermaid or images from Claude directly, so use text-based visuals only

### Parse draft (Code node)
```javascript
const raw = $input.item.json;
const content = raw.content && raw.content[0] ? raw.content[0].text : '';
const lines = content.split('\n').filter(l => l.trim() !== '');
const title = lines.length > 0 ? lines[0].replace(/^#+\s*/, '').trim() : '';
const subtitle = lines.length > 1 ? lines[1].replace(/^#+\s*/, '').trim() : '';
const body = lines.slice(2).join('\n\n').trim();
const sd = $getWorkflowStaticData('global');
sd.full_draft = content;
sd.draft_title = title;
sd.draft_subtitle = subtitle;
return {
  title: title,
  subtitle: subtitle,
  body: body,
  full_draft: content,
  word_count: content.split(' ').length,
  generated_at: new Date().toISOString()
};
```

### Save draft to Notion
- Creates a new Notion page in the Drafts database
- **Do not add blocks here** — expressions in block text fields are stored as literal strings, not evaluated
- Get the page `id` and `url` from this node's output for use in the email

### Append a block (Notion node)
- Operation: Append After
- Appends 3 paragraph blocks using substring(0,2000), substring(2000,4000), substring(4000,6000)
- Block 1: Expression mode ON — `={{ $('Parse draft').item.json.full_draft.substring(0, 2000) }}`
- Block 2: Expression mode ON — `={{ $('Parse draft').item.json.full_draft.length > 2000 ? $('Parse draft').item.json.full_draft.substring(2000, 4000) : ' ' }}`
- Block 3: Expression mode ON — `={{ $('Parse draft').item.json.full_draft.length > 4000 ? $('Parse draft').item.json.full_draft.substring(4000, 6000) : ' ' }}`
- **Important:** Only connect Append a block → Send draft for review. Remove the direct connection from Save draft to Notion → Send draft for review or the email fires twice.

### Send draft for review (Gmail)
- Full HTML email body (paste into expression editor popup):
```html
<html><body style="font-family: Arial, sans-serif; max-width: 640px; margin: 0 auto; padding: 24px; color: #1a1a1a;"><h2 style="color:#0099FF;">Your weekly draft is ready for review</h2><p><strong>Title:</strong> {{ $('Parse draft').item.json.title }}</p><p><strong>Subtitle:</strong> {{ $('Parse draft').item.json.subtitle }}</p><p><strong>Word count:</strong> {{ $('Parse draft').item.json.word_count }}</p><hr style="border:1px solid #eee; margin: 24px 0;"/><p>Read the full draft in Notion, then approve or reject below:</p><a href="{{ $('Save draft to Notion').item.json.url }}" style="display:inline-block; background:#333; color:white; padding:12px 28px; border-radius:6px; text-decoration:none; font-weight:bold; margin-right:12px; margin-bottom:16px;">Open in Notion</a><br/><a href="{{ 'http://localhost:5678/webhook/flowstate-approve?draft_id=' + $('Save draft to Notion').item.json.id + '&action=approve' }}" style="display:inline-block; background:#0099FF; color:white; padding:12px 28px; border-radius:6px; text-decoration:none; font-weight:bold; margin-right:12px;">Approve and Publish</a><a href="{{ 'http://localhost:5678/webhook/flowstate-approve?draft_id=' + $('Save draft to Notion').item.json.id + '&action=reject' }}" style="display:inline-block; background:#f5f5f5; color:#333; padding:12px 28px; border-radius:6px; text-decoration:none; font-weight:bold;">Reject</a></body></html>
```
- word_count renders as a number — works without string conversion when emojis and `&amp;` are removed from button text
- Webhook URLs: build as JS string concatenation inside `{{ }}` to avoid `&` parsing issues in href attributes

### Approval webhook
- HTTP Method: GET
- Path: `flowstate-approve`
- Respond: `Using 'Respond to Webhook' Node`
- **Do NOT set to "Immediately"** if a Success response node exists — causes "Unused Respond to Webhook node" error
- Production URL (used in email): `http://localhost:5678/webhook/flowstate-approve`
- Test URL (only works when listening): `http://localhost:5678/webhook-test/flowstate-approve`
- Workflow must be **Published** (green dot top-right) for production URL to work

### Assemble draft body (Code node)
- Reads from Notion blocks so edits made in Notion are used for publishing
```javascript
const staticData = $getWorkflowStaticData('global');
const page = $('Fetch final draft from Notion').first().json;
const title = page.name || staticData.draft_title || '';
const subtitle = page.property_subtitle || staticData.draft_subtitle || '';
const body = $('Get draft blocks').all().map(item => item.json.content).join('\n\n');
return { title, subtitle, body };
```

### Code in JavaScript (markdown → ProseMirror)
- Converts markdown body to Substack's ProseMirror JSON format before creating draft
```javascript
const body = $json.body || '';
const paragraphs = body.split('\n\n').filter(p => p.trim());
const content = paragraphs.map(p => {
  const trimmed = p.trim();
  if (trimmed.startsWith('## ')) {
    return { type: 'heading', attrs: { level: 2, textAlign: null }, content: [{ type: 'text', text: trimmed.slice(3) }] };
  }
  if (trimmed.startsWith('# ')) {
    return { type: 'heading', attrs: { level: 1, textAlign: null }, content: [{ type: 'text', text: trimmed.slice(2) }] };
  }
  if (trimmed.startsWith('> ')) {
    return { type: 'blockquote', content: [{ type: 'paragraph', content: [{ type: 'text', text: trimmed.slice(2) }] }] };
  }
  return { type: 'paragraph', attrs: { textAlign: null }, content: [{ type: 'text', text: trimmed }] };
});
return {
  title: $json.title,
  subtitle: $json.subtitle,
  draft_body: JSON.stringify({ type: 'doc', content: content.length ? content : [{ type: 'paragraph', attrs: { textAlign: null } }] })
};
```

### Create Substack draft
- Method: POST
- URL: `https://flowstatepm.substack.com/api/v1/drafts`
- Body:
```json
{
  "draft_title": "{{ $json.title }}",
  "draft_subtitle": "{{ $json.subtitle }}",
  "draft_body": "{{ $json.draft_body }}",
  "draft_bylines": [],
  "audience": "everyone",
  "section_chosen": false
}
```
- Headers required (all of these):
  - `Content-Type`: `application/json`
  - `Cookie`: full cookie string from DevTools (see Cookie section below)
  - `Origin`: `https://flowstatepm.substack.com`
  - `User-Agent`: `Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36`
  - `Accept`: `*/*`
  - `Accept-Language`: `en-US,en;q=0.9`
  - `Referer`: `https://flowstatepm.substack.com/publish/post`

### Publish to Substack
- Same Cookie and headers as Create Substack draft
- URL will need the draft ID from the previous node's response

---

## Communication Standards
- Always refer to nodes by their **exact name** as it appears in the N8N canvas (e.g., "HTTP Request", "Create Substack draft", "Assemble draft body")
- When giving instructions, include exact field names as they appear in the N8N UI (e.g., "Body Content Type", "Specify Headers", "Send Body")
- Steps must be explicit and numbered with no assumed context
- When providing any code, JSON, or JavaScript that requires updating a node, always provide the **entire** block so the user can fully replace the existing content — never provide partial snippets that require merging

---

## Critical Learnings & Gotchas

### N8N Expression Editor
- **Two modes exist:** field-level toggle (`={{ }}`) vs expression editor POPUP (pure JS)
- The HTTP Request node Body field uses the **popup** — paste pure JS with no `=` prefix
- In HTML fields (Gmail body), use `{{ expression }}` syntax
- Numbers in HTML expressions work fine without type conversion when used alone
- Build webhook URLs as JS string concatenation: `{{ 'http://...?id=' + $json.id + '&action=approve' }}` — avoids `&` parsing issues in href attributes

### Static Data
- Use `$getWorkflowStaticData('global')` to pass data between the Monday trigger execution and the Approval webhook execution (they run in separate contexts)
- Set in Parse draft node, read in Assemble draft body node

### Notion API
- 2000 character limit per block — split content using substring(0,2000), substring(2000,4000), etc.
- Block text expressions entered via N8N's Notion node UI are stored as literal strings — use a separate "Append After" node instead
- Notion node "Append After" operation evaluates expressions correctly when Expression mode is toggled ON per block

### Substack API
- **URL must use publication subdomain:** `https://flowstatepm.substack.com/api/v1/drafts` NOT `api.substack.com` or `substack.com`
- **Cloudflare bot protection:** Substack sits behind Cloudflare. Without browser-like headers (User-Agent, Accept, Origin, Referer, Accept-Language) the request gets blocked with a fake "Substack is experiencing technical problems" HTML page
- **Cookie:** Must be the FULL cookie string from DevTools (many cookies concatenated with `;`), not just `substack.sid`. Manually extending cookie expiry in DevTools does NOT refresh the server-side session — log out and log back in to get a fresh session
- **draft_body format:** Substack uses ProseMirror JSON document format, NOT plain text or markdown. Must be a JSON-stringified ProseMirror doc
- **draft_bylines:** Required field — send as `[]`
- **Two-step pattern:** Substack editor creates draft via POST (empty), then PATCHes content separately via `/api/v1/drafts/{id}`
- Current status: POST returning `{"error":""}` — investigating whether POST accepts content directly or only via subsequent PATCH

### Cookie Refresh Process
1. Go to substack.com, log out, log back in
2. Open DevTools → Network tab → filter "drafts"
3. Create a new post to trigger the POST request
4. Click the `drafts` request → Headers tab
5. Copy the entire Cookie header value (starts with `ab_testing_id=...`)
6. Paste into Cookie header in both Create Substack draft and Publish to Substack nodes

---

## Workflow File
- Location: `/Users/homecomputer/Downloads/Flow State PM — Weekly Article Publisher (Fixed).json`
- **Do not reimport** — loses all credentials. Edit nodes in place.

---

## Notion Setup
- Idea Bank database: contains ideas with fields `property_name`, `property_angle`, `property_key_insight`, `property_reader_pain`, `property_status`, `property_priority`
- Status filter value: `"Ready"` (not "Ready to Write")
- Priority field must be filled in Notion for sorting to work

---

## Pending / In Progress
- Substack draft creation still returning `{"error":""}` — need to confirm exact POST body format from DevTools payload tab
- Once Create Substack draft works, verify Publish to Substack node and end-to-end flow
