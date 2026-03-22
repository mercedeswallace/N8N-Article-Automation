# Automated Weekly Article Product Management Publisher

An N8N automation that writes, reviews, and publishes a weekly Substack newsletter end-to-end using Claude AI, Notion, Gmail, and Substack.

## What it does

**Every Monday at 8am:**
1. Pulls the top-priority idea from a Notion Idea Bank database
2. Sends the idea to Claude (claude-sonnet-4-6) with a custom system prompt matching the newsletter's voice
3. Saves the generated draft to a Notion Drafts database
4. Sends a review email via Gmail with Approve / Reject buttons

**On approval:**
1. Fetches the (optionally edited) draft from Notion — so manual edits in Notion are what gets published
2. Converts markdown to Substack's ProseMirror JSON format
3. Creates and publishes the draft to Substack via its internal API
4. Archives the page in Notion and returns a success page

## Architecture

```
Every Monday 8am
  → Pull top idea from Notion (Ideas DB, Status = "Ready", sort by Priority)
  → Claude writes draft (Anthropic API, claude-sonnet-4-6, 8192 max_tokens)
  → Parse draft (Code node — extract title/subtitle/body, store in static data)
  → Save draft to Notion (Drafts DB)
  → Append blocks to Notion page (3 × 2000-char chunks)
  → Send review email (Gmail — Approve / Reject links)

Approval webhook GET /webhook/flowstate-approve
  → Approved?
    → [yes] Fetch Notion draft → Get blocks → Assemble body
            → Convert markdown → ProseMirror JSON
            → POST to Substack drafts API
            → Publish to Substack
            → Mark published in Notion → Success page
    → [no]  Mark rejected in Notion → Rejection page
```

## Setup

### Prerequisites
- [N8N](https://n8n.io) self-hosted (tested on localhost:5678)
- Anthropic API key
- Notion integration token with access to your Idea Bank and Drafts databases
- Gmail OAuth2 credentials in N8N
- A Substack publication

### Import

1. Import `workflow-template.json` into N8N
2. Replace all placeholder values (see Configuration below)
3. **Do not reimport after setting credentials** — importing overwrites credential links

### Configuration

After importing, replace these placeholders in the relevant nodes:

| Placeholder | Where | What to put |
|---|---|---|
| `YOUR_NOTION_IDEAS_DB_ID` | "Pull top idea from Notion" node | Your Notion Idea Bank database ID |
| `YOUR_NOTION_DRAFTS_DB_ID` | "Save draft to Notion" node | Your Notion Drafts database ID |
| `YOUR_GMAIL_ADDRESS` | "Send draft for review" node | Your Gmail address |
| `YOUR_SUBSTACK_COOKIE_STRING` | "Create Substack draft" + "Publish to Substack" nodes | Full cookie string from DevTools (see below) |
| `YOUR_NOTION_CREDENTIAL_ID` | Notion nodes | Link to your Notion credential in N8N |
| `YOUR_ANTHROPIC_CREDENTIAL_ID` | "Claude writes draft" node | Link to your Anthropic API header-auth credential |
| `YOUR_GMAIL_CREDENTIAL_ID` | "Send draft for review" node | Link to your Gmail OAuth2 credential |

### Getting the Substack Cookie

The Substack API uses session cookies for authentication. These expire and need refreshing periodically.

1. Go to your Substack publication, log out, and log back in
2. Open DevTools → Network tab → filter by "drafts"
3. Create a new post to trigger a POST request
4. Click the `drafts` request → Headers tab
5. Copy the entire `Cookie:` header value (starts with `ab_testing_id=...`)
6. Paste into the `Cookie` header in both **Create Substack draft** and **Publish to Substack** nodes

### Notion Database Schema

**Idea Bank database** — fields used:
- `Name` (title) — article title idea
- `Angle` (text) — the specific angle to take
- `Key Insight` (text) — the core insight to convey
- `Reader Pain` (text) — the problem this solves for readers
- `Status` (status) — set to `Ready` to queue for writing
- `Priority` (multi-select) — used for sort order

**Drafts database** — auto-created by the workflow, no special schema required

## Key Technical Notes

- **Substack uses ProseMirror JSON** — plain text or markdown won't work; the "Code in JavaScript" node handles conversion
- **Substack API is two-step** in practice: POST creates the draft, content goes in at creation time via `draft_body`
- **Notion blocks have a 2000-character limit** — draft body is split across 3 blocks
- **Static data** (`$getWorkflowStaticData('global')`) passes title/subtitle between the Monday trigger execution and the webhook execution
- **Workflow must be Published** (green dot in N8N) for the production webhook URL to work

## Newsletter

[Flow State PM by Mercedes](https://flowstatepm.substack.com) — weekly articles on agentic workflows, AI-native product development, and building entrepreneurial PM careers.
