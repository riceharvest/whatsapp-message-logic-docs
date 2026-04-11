# Style Profiles — `contacts.json`

Per-contact style profiles + Dario's global writing style. Used to build the system prompt that tells the LLM how to write messages "as Dario."

## File Location

```
backend/styleguides/contacts.json
../styleguides/contacts.json  (same file, synced)
```

Loaded at startup via `prompts.ts → loadContactProfiles()`, cached in memory.

## Meta

```json
{
  "meta": {
    "generated": "2026-03-29",
    "note": "Auto-generated style profiles from conversation analysis. Used by the control center for AI message generation."
  }
}
```

## Profile Structure

Each profile is a named entry under `profiles`:

```json
"{ProfileName}": {
  "jid": "31612644205@s.whatsapp.net",
  "language": "dutch",
  "tone": "close friend, bro vibes, unfiltered",
  "style": "Very casual Dutch...",
  "emoji_frequency": "moderate",
  "avg_message_length": "variable",
  "formality": "very_casual",
  "sample_phrases": ["man heeft dr gwn binnen gelaten", "ewa ben vrij maar ga chillen"],
  "note": "Optional freeform note",
  "never_do": ["optional contact-specific prohibitions"]
}
```

### Field Meanings

| Field | Type | Purpose |
|-------|------|---------|
| `jid` | string | WhatsApp JID used for incoming message matching |
| `language` | enum | `english`, `dutch`, `mixed_dutch_english` |
| `tone` | string | Freeform description of conversational tone |
| `style` | string | Detailed behavioral description for the LLM |
| `emoji_frequency` | enum | `high`, `moderate`, `low`, `none` |
| `avg_message_length` | enum | `very_short`, `short`, `medium`, `variable` |
| `formality` | enum | `very_casual`, `casual`, `casual_work`, `intimate`, etc. |
| `sample_phrases` | string[] | **Critical.** Real messages Dario has sent to this person |
| `note` | string? | Special handling instructions |
| `never_do` | string[]? | Contact-specific things to avoid |

## Matching Logic

`findContactProfile(chatId)` in `prompts.ts`:

1. Split `chatId` by `@` → extract phone number (e.g. `31612644205`)
2. For each profile, match:
   - Exact JID match: `profile.jid === chatId`
   - Phone match: `profile.jid.split('@')[0] === phone`
3. Returns first match

Note: JIDs with `@lid` (low ID) are matched by phone part only, since WhatsApp can upgrade LID→WhatsApp JID.

## Global Style — `dario_global_style`

Applied to every contact, always.

```json
{
  "dario_global_style": {
    "languages": ["dutch", "english"],
    "language_switching": "Switches based on contact preference...",
    "common_patterns": [
      "Short messages, rarely writes paragraphs in chat",
      "Uses 'u' and 'ur' instead of 'you' and 'your'",
      "Emoji: 😇🤣😴 most common",
      "Uses 'Yo', 'Ewa', 'Yess', 'Ai'",
      "Dutch slang: 'jwz', 'gwn', 'vnv'",
      "Stickers and GIFs frequently",
      "Often quotes/replies to specific messages",
      "Apologetic when late responding ('sorry man ik sliep')",
      "Tech-literate, drops AI/dev references naturally",
      "Direct, doesn't beat around the bush"
    ],
    "never_do": [
      "Never write formal or corporate-sounding messages",
      "Never use proper punctuation consistently",
      "Never write long paragraphs in casual chats",
      "Never sign off messages",
      "Never say 'Hallo' or 'Goedemorgen' to friends"
    ]
  }
}
```

## Current Profiles

| Profile | JID | Language | Tone | Key traits |
|---------|-----|----------|------|------------|
| Call Me By Ur Name | `@lid` | english | flirty, playful | Heavy emoji, stickers, short, teasing |
| Rudo | `@lid` | english | friendly bro | Tech talk, casual, low emoji |
| Omar | `@lid` | dutch | transactional | Very short Dutch, shop logistics |
| Familie | `@g.us` | dutch | warm family | Group chat, kitchen reno, no auto-reply |
| Bilal Freedom | `@s.whatsapp.net` | dutch | work colleague | Shop ops, direct, business casual |
| s | `@lid` | english | deep, emotional | Vulnerable exchanges, supportive, longer |
| B (colleague) | `@lid` | dutch | work brief | Weighing system issues, transactional |
| B (pin) | `@lid` | dutch | work brief | Payment terminal, minimal |
| Ghjiulijah | `@s.whatsapp.net` | english | flirty, bold | GIFs, short punchy lines, very casual |
| Alex (lid) | `@lid` | mixed | close friend | Bro vibes, stickers, memes, Dutch+English mixed |
| Alex (whatsapp) | `@s.whatsapp.net` | dutch | close friend | Very casual Dutch, memes, unfiltered |

## Important Notes

### Group Chats
- `Familie` (family group) is marked with `note: "GROUP CHAT - do not auto-reply, only draft"`
- Auto-mode should be `manual` or `draft` for groups — never `auto`

### LID vs WhatsApp JIDs
- Some contacts have both `@lid` and `@s.whatsapp.net` JIDs depending on privacy mode
- The profile matching handles both via phone extraction

### Sample Phrases Are Key
The `sample_phrases` array is the most powerful part of each profile. These are actual messages Dario has sent. The LLM uses these as anchor examples for style matching.

## Updating Profiles

1. Edit `backend/styleguides/contacts.json`
2. The file is re-read on next server start (or implement cache invalidation)
3. Changes affect new message generation immediately if cache-busting reload is added

## Adding a New Contact Profile

1. Add a new entry to `profiles` with a descriptive name
2. Fill in all fields (especially `sample_phrases`)
3. The contact must have a matching `jid` or phone number in the DB (`contacts.jid`)
4. When a message comes in from that JID, the profile is automatically matched
