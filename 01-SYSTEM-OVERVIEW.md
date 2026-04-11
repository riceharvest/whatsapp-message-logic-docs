# WhatsApp Message Generation Logic

Comprehensive documentation of the system used to generate AI-powered WhatsApp message drafts for Dario.

## Repository Structure

```
├── backend/
│   ├── src/
│   │   ├── openrouter/
│   │   │   ├── client.ts        # OpenRouter API calls with retry logic
│   │   │   └── prompts.ts        # System prompt building + style profiles
│   │   ├── whatsapp/
│   │   │   ├── client.ts         # wacli CLI wrapper
│   │   │   ├── manager.ts       # wacli process manager
│   │   │   └── contact-sync.ts  # Contact cache + sync
│   │   ├── db/
│   │   │   ├── index.ts          # DB connection
│   │   │   └── schema.ts         # Drizzle ORM schema
│   │   └── index.ts              # Hono server + all routes
│   ├── styleguides/
│   │   └── contacts.json         # Per-contact style profiles
│   └── .env                       # API keys, model config
├── frontend/
│   └── (React SPA for UI)
└── styleguides/
    └── contacts.json              # Same as backend/styleguides/contacts.json
```

## Core Components

| File | Responsibility |
|------|---------------|
| `prompts.ts` | Builds the system prompt from style presets + contact profile |
| `client.ts` (openrouter) | Calls OpenRouter chat completions API |
| `contacts.json` | Per-contact style profiles + Dario's global style |
| `index.ts` | Routes: `/api/auto/generate`, inbound handling, rate limiting |

## Message Generation Flow

```
Inbound WA message
       ↓
  ignored? → discard
       ↓
  autoMode = manual? → discard (draft only)
       ↓
  autoMode = draft/auto
       ↓
  keyword filter check (if set)
       ↓
  rate limit check (30/hour/contact)
       ↓
  Fetch last N messages via wacli
  (N = contextLimit, default 20)
       ↓
  buildMessages(context) → adds system prompt
       ↓
  generateCompletion() → OpenRouter API
       ↓
  autoMode=auto  → send immediately
  autoMode=draft → store in outbound_queue as "draft"
       ↓
  WebSocket broadcast to UI
```

## Key Files Detail

### `prompts.ts` — System Prompt Building

**`buildSystemPrompt(styleName?, customPrompt?, stylePresets?, chatId?)`**
Priority order:
1. `customPrompt` — raw override, used directly as system prompt
2. `chatId` → contact profile from `contacts.json` → built dynamically
3. `styleName` → preset match from `DEFAULT_STYLES`
4. Falls back to `DEFAULT_STYLES[0]` (Professional)

**`buildMessages(context, ...)`**
- Prepends system prompt as `{ role: "system", content: ... }`
- Appends conversation context as `{ role: "user" }` (inbound) or `{ role: "assistant" }` (outbound)
- Context is built by reading the **last N messages** via wacli, ordered oldest→newest

### `contacts.json` — Style Profiles

**Per-contact profile fields:**
- `jid` — WhatsApp JID for matching
- `language` — "english" | "dutch" | "mixed_dutch_english"
- `tone` — freeform description
- `style` — detailed behavioral description
- `emoji_frequency` — "high" | "moderate" | "low" | "none"
- `avg_message_length` — "very_short" | "short" | "medium" | "variable"
- `formality` — "very_casual" | "casual" | "casual_work" | "intimate" | etc.
- `sample_phrases[]` — actual messages Dario has sent (most important)
- `note` — special handling instructions
- `never_do[]` — contact-specific prohibitions

**Global style (`dario_global_style`):**
- `languages[]` — active languages (dutch, english)
- `language_switching` — when to switch languages
- `common_patterns[]` — behavioral patterns
- `never_do[]` — global prohibitions

### Contact Profile Prompt Assembly

When a chatId matches a contact, `buildContactStylePrompt(chatId)` assembles:

```
You are ghostwriting WhatsApp messages as Dario. Reply as if you ARE Dario.

## Dario's General Style
- Languages: dutch, english
- [language_switching rule]
- [each common_pattern]

## Never Do (Global)
- [each never_do rule]

## Contact: {name}
- Language: {profile.language}
- Tone: {profile.tone}
- Style: {profile.style}
- Emoji frequency: {profile.emoji_frequency}
- Message length: {profile.avg_message_length}
- Formality: {profile.formality}

## Example Messages Dario Sends to {name}
- "{phrase1}"
- "{phrase2}"

## Note: {note}

## Never Do (Contact-Specific)
- [each contact-level never_do]

Write a single reply message. Match Dario's style exactly for this contact. No meta-commentary, no quotation marks around the reply — just the raw message text.
```

## Auto-Mode Logic

Three modes per contact:
- **`manual`** — no AI involvement, human writes everything
- **`draft`** — generate a draft, store in `outbound_queue`, human reviews and sends
- **`auto`** — generate and send immediately (no human review)

## Rate Limiting

- **30 messages per contact per hour** (rolling window)
- Implemented in-memory (not persistent across restarts)
- Applies to both `draft` and `auto` mode triggers

## Keyword Filtering

If `auto_keywords` is set on a contact, inbound messages must contain at least one keyword to trigger generation. Keywords are case-insensitive substring matched.

## Context Window

- Default: **20 messages** (configurable per contact via `context_limit`)
- Max: 100
- Messages are read via `wacli messages list --chat {chatId} --limit {N} --json`
- Messages fed to the LLM oldest-first

## Model Configuration

- Default model: `openai/gpt-4o-mini`
- Configurable via `OPENROUTER_MODEL` env var
- Temperature: 0.7
- Max tokens: 1000

## Cost Tracking

- `usage` table records: date, model, tokens, cost_cents
- Cost estimated via `estimateCost()` using per-model pricing table
- `/api/stats` returns today's total tokens + cost

## OpenRouter Retry Logic

`generateCompletion()` implements:
- Exponential backoff starting at 1s, capped at 60s
- Max 5 attempts
- Respects `Retry-After` header on 429
- AbortSignal support for cancellation
