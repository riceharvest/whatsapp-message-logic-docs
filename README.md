# whatsapp-message-logic-docs

Documentation for Dario's WhatsApp Control Center message generation logic.

This repo contains technical documentation about how the AI-powered WhatsApp message drafting system works — system prompts, style profiles, database schema, API endpoints, and lessons learned from real conversations.

## Contents

- [01-SYSTEM-OVERVIEW](01-SYSTEM-OVERVIEW.md) — Architecture, message flow, core components
- [02-CONTACTS-PROFILES](02-CONTACTS-PROFILES.md) — Per-contact style profiles (`contacts.json`)
- [03-SYSTEM-PROMPT](03-SYSTEM-PROMPT.md) — Generated prompt structure + what was learned from the Jess incident
- [04-DATABASE](04-DATABASE.md) — Drizzle ORM schema
- [05-API](05-API.md) — REST + WebSocket endpoints
- [06-TECHNICAL-NOTES](06-TECHNICAL-NOTES.md) — Real-world incidents, bugs, design decisions

## Purpose

This documentation is meant to be shared with another LLM agent so it can review and critique the message generation logic without having to read the source code.

## Quick Summary

**What it does:** Takes inbound WhatsApp messages, builds a conversation context, generates a reply draft using OpenRouter, and either stores it for human review (`draft` mode) or sends it immediately (`auto` mode).

**Key components:**
- `prompts.ts` — builds the system prompt from style presets + `contacts.json`
- `contacts.json` — per-contact style profiles with sample phrases
- wacli CLI — WhatsApp backend (no Chrome/Puppeteer)
- Hono server + WebSocket — serves the UI and broadcasts real-time updates
- Drizzle ORM + SQLite — local storage for contacts, messages, drafts
