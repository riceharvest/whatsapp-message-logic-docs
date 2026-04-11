# Database Schema — Drizzle ORM

## Tables

### `contact`

Stores contacts synced from WhatsApp via wacli, plus per-contact control settings.

| Column | Type | Notes |
|--------|------|-------|
| `id` | INTEGER PK | Auto-increment |
| `phoneNumber` | TEXT UNIQUE | Phone number, e.g. `31612644205` |
| `jid` | TEXT | Full WhatsApp JID: `31612644205@s.whatsapp.net` or `31612644205@lid` |
| `name` | TEXT | Display name from WhatsApp |
| `isGroup` | BOOLEAN | True for group chats |
| `lastSeen` | INTEGER | Unix timestamp of last seen |
| `lastMessageTs` | INTEGER | Unix timestamp of most recent message |
| `label` | TEXT | User-set label: "Customer", "Lead", "Friend", etc. |
| `stylePrompt` | TEXT | Custom override system prompt (nullable) |
| `autoMode` | TEXT | `manual` \| `draft` \| `auto` — AI behavior for this contact |
| `ignored` | BOOLEAN | If true, all AI processing is skipped |
| `autoKeywords` | JSON TEXT | Array of keywords — only trigger AI if inbound matches one |
| `contextLimit` | INTEGER | How many messages to feed as context (default 20, max 100) |
| `createdAt` | INTEGER | Unix timestamp |

### `message`

All messages stored locally (inbound + outbound).

| Column | Type | Notes |
|--------|------|-------|
| `id` | INTEGER PK | Auto-increment |
| `chatId` | TEXT | Full JID |
| `contactId` | INTEGER FK | References `contact.id` |
| `direction` | TEXT | `inbound` \| `outbound` |
| `body` | TEXT | Message text |
| `timestamp` | INTEGER | Unix timestamp (from WA) |
| `messageIdWa` | TEXT UNIQUE | WhatsApp message ID |
| `isRead` | BOOLEAN | Read status |
| `mediaUrl` | TEXT | Local path or remote URL if media was sent |
| `createdAt` | INTEGER | Unix timestamp |

### `outbound_queue`

Drafts and pending messages awaiting human review or send.

| Column | Type | Notes |
|--------|------|-------|
| `id` | INTEGER PK | Auto-increment |
| `contactId` | INTEGER FK | References `contact.id` |
| `messageText` | TEXT | The generated message content |
| `status` | TEXT | `pending` \| `sent` \| `failed` \| `draft` |
| `scheduledAt` | INTEGER | Unix timestamp for scheduled sends (future) |
| `sentAt` | INTEGER | Unix timestamp when actually sent |
| `error` | TEXT | Error message if failed |
| `metadata` | JSON TEXT | `{ model, tokens, cost, retryCount }` |
| `createdAt` | INTEGER | Unix timestamp |

### `usage`

Daily OpenRouter API usage for cost tracking.

| Column | Type | Notes |
|--------|------|-------|
| `id` | INTEGER PK | Auto-increment |
| `date` | TEXT | ISO date string: `2026-04-10` |
| `model` | TEXT | OpenRouter model slug |
| `tokens` | INTEGER | Total tokens for this request |
| `costCents` | INTEGER | Estimated cost in cents |
| `createdAt` | INTEGER | Unix timestamp |

### `style_preset`

Custom style presets stored in DB (extends `DEFAULT_STYLES`).

| Column | Type | Notes |
|--------|------|-------|
| `id` | INTEGER PK | Auto-increment |
| `name` | TEXT UNIQUE | Preset name |
| `prompt` | TEXT | Full system prompt text |
| `createdAt` | INTEGER | Unix timestamp |

## Auto-Mode Decision Flow

```
contact.ignored = true        → skip all AI, store only
contact.autoMode = 'manual'  → skip AI, store only
contact.autoMode = 'draft'   → generate, store as draft in outbound_queue
contact.autoMode = 'auto'    → generate, send immediately + store as 'sent'
```

## Keyword Filtering

When `contact.autoKeywords` is non-empty:
```ts
const matches = autoKeywords.some((kw: string) =>
  body.toLowerCase().includes(kw.toLowerCase())
);
if (!matches) return; // don't trigger
```

## Context Window

`contact.contextLimit` (default 20, max 100) controls how many messages are fetched from wacli and included in the LLM context.

## Indexes

- `contact.phoneNumber` — unique
- `contact.jid` — for JID lookups
- `message.chatId` — for fetching conversation history
- `message.contactId` — for joins
- `outbound_queue.status` — for draft queue queries
- `usage.date` — for daily stats
