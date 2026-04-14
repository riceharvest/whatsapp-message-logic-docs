# WhatsApp Message Generation — System Overview

## Architecture

The current pipeline is the **Simplified Pipeline** (`DraftSimplified`), introduced 2026-04-13. It replaced an earlier rules-based + few-shot system that produced generic patterns like "lol 😏 u binge watch."

The simplified approach is: **identity first, not rules first.** Instead of enumerating do's and don'ts, the model receives who Dario is, what stage the conversation is at, and what's been said recently.

## Pipeline Flow

```
Inbound message (or manual generate trigger)
       ↓
Debounce deduplication (prevents double-fires)
       ↓
Fetch last N conversation messages (default: 4, configurable)
       ↓
Classify conversation stage (intro | getting_to_know | flirty | deep | stale)
       ↓
BuildSimplifiedPrompt(contactName, contactNotes, stage, fewShots)
  → DarioPersona (identity block)
  → Stage directive (FLIRTY / DEEP / GETTING_TO_KNOW tone guidance)
  → Relationship status from contact notes
  → Few-shot examples (currently disabled — see Note below)
       ↓
Format conversation thread (relative timestamps, gap separators, media filtered)
       ↓
Single LLM call: systemPrompt + formatted thread → draft text
       ↓
Save draft to DB → WebSocket broadcast → UI shows draft card
```

**Note on few-shots:** Static few-shot examples were disabled 2026-04-13. The `DarioPersona` identity block is self-sufficient; old chosen drafts in the DB were contaminating output with generic patterns. Re-enable once chosen drafts are curated to match Dario's actual voice.

## Two Generation Paths

| Path | Trigger | Draft count | Code |
|------|---------|-------------|------|
| **Auto-draft** | Inbound WA message (autoMode=draft) | 1 | `handleDebounceFire` in `main.go` |
| **Manual generate** | `/generate` command or web UI button | 1 (was 3, fixed 2026-04-14) | `HandleAPIGenerateStream` in `generate.go` |

Both paths use the same `GenerateNOptionsForContact` service method, which calls `DraftSimplified`.

## Key Files

| File | Role |
|------|------|
| `internal/llm/simplified.go` | `DarioPersona` constant + `BuildSimplifiedPrompt()` + `ExtractRelationshipStatus()` |
| `internal/llm/draft_simplified.go` | `DraftSimplified()` — the main generation function |
| `internal/stages/stages.go` | Stage classification (keyword-based + embedding fallback) |
| `internal/api/service.go` | `GenerateNOptionsForContact()` — orchestrator |
| `internal/api/generate.go` | `HandleAPIGenerateStream()` — HTTP SSE handler |
| `cmd/whatsapp-personal/main.go` | `handleDebounceFire()` — auto-draft on inbound |

## Stage System

Contacts are classified into one of five stages:

| Stage | Trigger | Prompt Directive |
|-------|---------|-----------------|
| `intro` | < 10 total messages | "Be curious, ask open-ended questions, keep it light and fun" |
| `getting_to_know` | 10–30 messages, no keywords | "Show genuine interest, ask follow-ups about their life" |
| `flirty` | ≥30 messages + flirty keywords | "Be playful and bold, use pet names, tease" |
| `deep` | ≥60 messages + deep keywords | "Be real and vulnerable, reference things they told you before" |
| `stale` | No message in 48h | "Re-engage naturally, don't be needy" |

For contacts with >20 messages, an **embedding-based classification** is attempted as a secondary check. Stage vectors are embedded once and cached.

## What Changed (2026-04-13)

Before:
- Rules + few-shots → generic output ("lol 😏 u binge watch")
- Many moving parts: embedding lookup, humanization, banned phrase filter, judge scoring

After:
- `DarioPersona` identity block → sharp, specific output
- Single LLM call → no judge, no embedding at generation time
- Stage-based tone guidance embedded in prompt
- Few-shots disabled (clean persona sufficient)

The result: cleaner voice, fewer moving parts, lower cost.