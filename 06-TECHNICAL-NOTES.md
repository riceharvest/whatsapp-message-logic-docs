# Technical Notes

## Simplified Pipeline Migration (2026-04-13)

### Why the Change

The previous system used:
1. Rules-based persona
2. Static few-shot examples from past chosen drafts
3. Embedding lookups at generation time
4. Humanization pass
5. Banned phrase filter
6. Judge scoring

The result was generic output: "lol 😏 u binge watch" patterns. The model was correctly following rules but producing flat, recognizable AI prose.

### What Changed

The new pipeline (`DraftSimplified` in `internal/llm/draft_simplified.go`) does one thing:

```
BuildSimplifiedPrompt() → single LLM call → return text
```

No embeddings at generation time. No judge. No humanization. No banned phrase filter.

The identity block (`DarioPersona`) is self-sufficient — personality + preferences + output rules give the model enough signal to produce sharp output without examples of old drafts.

### Few-Shot Disable

Static few-shot examples were disabled because DB-chosen drafts were contaminating the output. The problem: past "chosen" drafts weren't actually good Dario voice — they were just drafts he approved because they were adequate, not because they matched his voice.

Fix: clean the chosen draft corpus (curated to actually sound like Dario) before re-enabling.

## Web UI Manual Generate (2026-04-14)

Fixed the web UI generate button to produce 1 draft instead of 3.

**Before:** `POST /api/contacts/{id}/generate-stream` → `GenerateDraftsForContact(ctx, contactID)` → 3 drafts  
**After:** `HandleAPIGenerateStream` calls `GenerateNOptionsForContact(ctx, contactID, 1)` directly

The Telegram `/generate` command also uses `GenerateDraftsForContact` which still generates 3. If you want that changed too, let me know.

## Stage Classification

Stage is determined by `stages.classify()` which uses:
1. **Message count thresholds:** intro (<10), getting_to_know (10-30), deeper (30+)
2. **Keyword matching:** flirty keywords (cute, gorgeous, miss you, etc.) and deep keywords (love, trust, future, etc.)
3. **Staleness:** conversations quiet >48h → `stale` stage

For contacts with >20 messages, `stages.EmbedClassify()` runs as a secondary check using cosine similarity between the contact's conversation embedding and pre-embedded stage directive vectors. This requires `contact_embeddings` table to have a vector for the contact.

Stage directive vectors are cached in memory on first embed (`stageVecCache`). Stage directives:

```
intro:         "This is a new conversation. Be curious, ask open-ended questions, keep it light and fun."
getting_to_know: "You are getting to know this person. Show genuine interest in what they share, ask follow-ups about their life."
flirty:         "The vibe is flirty. Be more playful and bold, use more pet names, tease a little."
deep:           "This is getting serious/emotional. Be real and vulnerable, reference things they have told you before."
stale:          "This conversation has been quiet for a while. Re-engage naturally, don't be needy, casually ask what they've been up to."
```

## Context Window

Global default is 4 messages (set in code, `service.go`). The DB `settings` table has `context_messages` defaulting to 40, but that's a legacy value — the code reads the settings table and uses 4.

This is intentionally lean: 4 messages keeps the model focused on recent context without overcrowding the prompt. Can be overridden per-contact via the contact's `context_messages` field.

## Temperature

Global default: `0.7` (set in code + DB). Configurable via `llm_temperature` setting. Per-call override supported via `SimplifiedOpts.Temperature`.

## Model

Default: `openai/gpt-4.1-nano`. Can be overridden per-contact via `llm_model` field on the contact, or via global `llm_model` setting.

## Debounce

Inbound messages are debounced per JID — a conversation must be quiet for a random delay between `debounce_min_secs` (default: 120) and `debounce_max_secs` (default: 600) before a draft is auto-generated.

The debounce state is persisted in the `debounce` table so crashes don't cause duplicate fires on restart.

## A/B Test Infrastructure

The `draft_outcomes` table and `ab_test_enabled` setting exist but the simplified pipeline doesn't currently use them. The infrastructure was built for multi-variant comparison (model, temperature, etc.) but is not active with the current setup.

## LID vs Standard JID

WhatsApp privacy mode uses `@lid` JIDs. The system matches contacts by phone number extraction, so both LID and standard JIDs for the same phone work correctly.

## Audit Logging

The `audit_log` table records all significant events. When `llm_call: true`, `llm_prompt_tokens` and `llm_completion_tokens` are also recorded.

## Media Messages

Media messages are filtered from conversation context (📎 Media, 📷 Image, 🎥 Video, 🎵 Audio, 🏷️ Sticker, "Reacted ..."). These are stored in the DB but excluded from the LLM context to avoid noise.

## Output Format

The LLM is instructed to output **only the message text** — no quotes, no labels, no JSON (unless multiple separate messages are genuinely needed). Multiple messages formatted as `["first", "second"]` — max 2.

Emoji: match the contact's energy. If she uses none, use none. If she uses 1, use 1. Never stack.

## WebSocket

The `Hub` type in `internal/api/hub.go` manages WebSocket connections. It broadcasts events (new_message, draft_created, etc.) to all connected clients. Used by the frontend to show real-time updates without polling.