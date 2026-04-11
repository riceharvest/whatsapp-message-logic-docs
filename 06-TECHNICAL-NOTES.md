# Technical Notes — Real-World Incidents & Decisions

## Jess Incident (2026-04-10) — Conversational Style Failure

### What Happened

Dario was texting Jess (contact_id=80, 2am her time, Nigerian timezone). After a conversation about recon/scoping solo activities and life direction, Jess pushed back explicitly:

> "plus i feel like im talking to an ai, with all the questions u're dropping"

> "u had also been using the exact same emoji since 9:54am"

She then set a hard boundary:
> "so respectfully, no more questions about it. coz even i, is figuring it all about, still."

### Root Cause

The system prompt (`prompts.ts`) was generating messages that:
1. Back-to-back questions → interrogative tone
2. Repeated the same emoji (😇) for hours
3. Never let a conversational beat breathe — every message was immediately followed by another question

The model was correctly following the style instructions (casual, short messages, sample phrases) but the **conversational mechanics** were broken. Style matching ≠ conversation skill.

### Fixes Applied to System Prompt

```
- never repeat the same emoji twice in a row
- vary tone (not every message the same energy)
- not every message needs a question
- follow with reaction or observation before another question
- let the conversation breathe
- don't therapist / don't small talk
```

### Lessons Learned

1. **Sample phrases and never_do rules cover "what to say" but not "how to flow."** Conversational mechanics (question density, observation-before-probing, emoji variety) need explicit rules in the system prompt.

2. **When someone shares something vulnerable, react first, probe second.** Jess said "depends" about planning vs. going with the flow — that was an opening, not a question to immediately dissect.

3. **The boundary signal.** Jess saying "no more questions about it" is a direct override. The system should recognize boundary-setting language and stop probing that topic.

4. **Emoji variety is a low-effort high-impact signal of humanness.** Repeating the same emoji for hours is a tell.

### Recommendations Going Forward

Add to the system prompt generation (`prompts.ts`):
- A "conversational mechanics" section with question density rules
- Emoji variety enforcement
- Boundary signal recognition (explicit requests to stop a topic should be honored)

Add to `contacts.json` as a possible per-contact field:
- `sensitive_topics[]` — topics where the contact has set explicit boundaries

---

## wacli Sync Process Auto-Restart

The `wacli sync --follow --json` process can exit unexpectedly. The `WacliClient` in `client.ts` implements:

- Exponential backoff restart (1s, 2s, 4s... capped at 60s)
- Max 10 restart attempts before giving up
- `maxRestartAttempts` guard to prevent infinite restarts

This is transparent to the user — the stream reconnects automatically.

---

## Draft Query Bug (2026-04-10)

A subagent ran a query against the local SQLite DB:

```sql
SELECT d.text, d.created_at FROM drafts d ...
```

Error: `no such column: text` — the column is named `draft_text`.

Fixed by updating the query to use `d.draft_text`.

---

## LID vs Standard JID Matching

WhatsApp privacy mode uses "low ID" (`@lid`) JIDs. Some contacts have both:
- `31612644205@lid` (privacy/LID mode)
- `31612644205@s.whatsapp.net` (standard)

Profile matching in `prompts.ts` handles both by extracting the phone part (`31612644205`) and matching against that. This is why Alex has both an `Alex (lid)` and `Alex (whatsapp)` profile — they may be the same person with different JID formats.

---

## Group Chats — Auto-Reply Guard

`Familie` (family group chat) has explicit note:
```
"note": "GROUP CHAT - do not auto-reply, only draft"
```

Group chats should be `manual` or `draft` mode — never `auto`. The system has no way to enforce this automatically (it's a configuration decision), but the `contacts.json` profile documents it as guidance.

---

## Rate Limiting — In-Memory

The rate limiter (`checkRateLimit()` in `index.ts`) is in-memory:
- Not persisted across server restarts
- Per-process only (no shared state if server is scaled)

This is acceptable for MVP. Production would want Redis or similar.

---

## OpenRouter Cost Estimation

`estimateCost()` in `client.ts` uses a hardcoded pricing table, not live OpenRouter pricing API. Prices can change; this is a simplification. For accurate billing, would need to call OpenRouter's `/api/v1/models` or use their usage API.
