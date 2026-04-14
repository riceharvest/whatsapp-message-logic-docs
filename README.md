# WhatsApp Message Logic

Documentation for the Dario's WhatsApp AI draft generation system.

## What's in here

| File | Description |
|------|-------------|
| `01-SYSTEM-OVERVIEW.md` | Pipeline flow, generation paths, stage system |
| `02-CONTACTS-PROFILES.md` | Contact settings, notes, stage, JID matching |
| `03-SYSTEM-PROMPT.md` | Full DarioPersona + prompt assembly details |
| `04-DATABASE.md` | Schema for all tables |
| `05-API.md` | REST endpoints + WebSocket events |
| `06-TECHNICAL-NOTES.md` | Decisions, migrations, known issues |

## Quick Summary

The system generates WhatsApp reply drafts for Dario using a **simplified pipeline** (as of 2026-04-13):

1. Classify conversation stage (intro → getting_to_know → flirty → deep → stale)
2. Build system prompt: `DarioPersona` + stage directive + contact notes + conversation thread
3. Single LLM call → draft text → save to DB → push via WebSocket to UI

No embeddings at generation time. No judge. No humanization. Identity-first, not rules-first.

## Key Constants

- **Default model:** `openai/gpt-4.1-nano`
- **Temperature:** `0.7`
- **Context window:** 4 messages (intentionally lean)
- **Auto-draft:** 1 draft per inbound message burst
- **Manual generate:** 1 draft (web UI button, fixed 2026-04-14)
- **/generate command:** 3 drafts (Telegram bot)