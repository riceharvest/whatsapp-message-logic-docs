# Contacts & Profiles

## Contact Settings

Each contact has configurable settings stored in the `contacts` table:

| Column | Type | Purpose |
|--------|------|---------|
| `id` | INTEGER PK | Auto-increment |
| `name` | TEXT | Display name from WhatsApp |
| `phoneNumber` | TEXT UNIQUE | Phone number, e.g. `31612345678` |
| `jid` | TEXT | Full WhatsApp JID: `31612345678@s.whatsapp.net` or `31612345678@lid` |
| `mode` | TEXT | `manual` \| `draft` \| `auto` \| `ignore` — AI behavior |
| `stage` | TEXT | Current conversation stage (auto-classified, see `01-SYSTEM-OVERVIEW.md`) |
| `notes` | TEXT | Freeform notes — used as context in prompt building |
| `llm_model` | TEXT | Per-contact model override (optional) |
| `context_messages` | INTEGER | Per-contact context window override (optional) |
| `is_group` | BOOLEAN | True for group chats |

### Mode meanings

- **`manual`** — no AI involvement; human writes everything
- **`draft`** — generate a draft, human reviews and sends
- **`auto`** — generate and send immediately (no human review)
- **`ignore`** — skip all AI processing for this contact

### Notes field

The `notes` field is the primary per-contact context passed to the LLM. It's freeform text that can include:
- Relationship context ("met on Hinge, been on 3 dates")
- Topic preferences ("doesn't like discussing work")
- Style guidance ("speaks Dutch mostly")
- Sensitive topics ("asked me to stop asking about her family")

Notes are extracted by `ExtractRelationshipStatus()` in `simplified.go` — the first 2 lines (non-`style:` lines) are injected into the prompt under "## Where things stand:"

### Stage field

Stage is auto-classified by `stages.classify()` based on message count + keywords. For contacts with >20 messages, `stages.EmbedClassify()` is used as a secondary check. Stage is stored in `contacts.stage` and passed to `BuildSimplifiedPrompt()`.

## JID Matching

WhatsApp uses two JID formats:
- **Standard:** `31612345678@s.whatsapp.net`
- **Low ID (privacy mode):** `31612345678@lid`

Both formats refer to the same phone number. The system matches contacts by phone number extraction — it splits the JID at `@` and uses the phone part for matching.

## Per-Contact Model Override

The `llm_model` field on a contact overrides the global default for that contact only. If not set, the global `llm_model` setting is used (default: `openai/gpt-4.1-nano`).

## Per-Contact Context Override

The `context_messages` setting overrides the global `context_messages` setting for that contact only. Default global is 4. Some contacts benefit from more context (deep conversations), others from less (shallow or one-word exchanges).