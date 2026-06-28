# CLAUDE.md

Persistent context for Claude Code sessions on this repo. Reference material — concise and dense, not a tutorial.

## Project Overview

Personal website — a static collection of single-file HTML pages, no build step or framework.

- **index.html** — hub/landing page; card grid linking to the other pages.
- **dashboard.html** — daily dashboard: date, quote, to-do/goals checklists (client-only).
- **phillies.html** — multi-agent AI Phillies Analyst (same architecture as china.html); no longer a static batting stats table.
- **pokemon.html** — random Pokémon explorer, filterable by generation; uses the public PokeAPI (`pokeapi.co`).
- **china.html** — the most advanced page: a multi-agent AI historian with persistent conversation memory.

Each page is a self-contained `.html` file with inline `<style>` and `<script>`. There is no shared CSS/JS file — conventions are duplicated per page by hand.

## Architecture — Cloudflare Worker Proxy

- All AI calls go through a Cloudflare Worker proxy at `https://claude-proxy.kevinbyrd580.workers.dev` (referenced as `WORKER_URL` in china.html).
- **NEVER call the Anthropic API directly from any HTML/JS file.** API keys must never appear in client-side code — the Worker holds the key.
- The Worker **source code lives in Cloudflare, not in this repo**, and is edited directly in the Cloudflare dashboard. Changes to agent orchestration, prompts, or DB logic happen there, not here.
- The Worker orchestrates a **5-agent system**:
  `Master Planner → [Researcher, History, Language, Data, Culture — run in parallel via Promise.all] → Master Synthesizer`
- Browser ↔ Worker communication uses **Server-Sent Events (SSE)**. Event types:
  `conversation_id`, `status`, `plan`, `workers_starting`, `worker_done`, `worker_error`, `final_answer`, `done`, `error`.
  The client parses these in `handleEvent()` / `parseSSEEvent()` in china.html (split on `\n\n`, parse `event:` / `data:` lines).

## Architecture — Dual-Domain Worker

- The Worker is **dual-domain**: it serves both the **"china"** and **"phillies"** pages from the same codebase.
- The browser sends a `domain` field in requests — `{ domain: "china" }` or `{ domain: "phillies" }` — to select which page's behavior applies.
- The **`getPrompts(domain)`** function switches between the China and Phillies agent prompt sets.
- Worker **status labels are domain-specific** — e.g. `📜 History Specialist` for China vs. `⚾ Franchise Historian` for Phillies.

## Architecture — D1 Database

- Database name: **china-chat-memory**, bound to the Worker as `env.DB`.
- Tables:
  - `conversations (id, title, domain, created_at, updated_at)`
  - `messages (id, conversation_id, role, content, created_at)`
- The `conversations` table has a **`domain` column** (`TEXT NOT NULL DEFAULT 'china'`) so the same DB stores both China and Phillies conversations.
  - **`getAllConversations` filters by `domain`** so each page's sidebar only shows its own conversations.
  - **`getOrCreateConversation` saves the `domain`** when creating new conversations.
- Worker request modes (sent as `mode` in the POST body): `multi_agent`, `get_conversations`, `get_messages`, `delete_conversation`.
- **Conversation history is loaded server-side from the database, NOT tracked client-side.** The client only holds `currentConversationId` and sends it with each `multi_agent` request; the Worker rehydrates context from D1.

## Phillies Page (phillies.html)

- **No longer a static batting stats table** — it is now a **multi-agent AI Phillies Analyst** built on the **same architecture as china.html**: same SSE streaming, conversation sidebar, markdown rendering, and `Sources:` section.
- Uses **Phillies red `#E81828`** as its accent (not the crimson `#C41E3A` used on china.html).
- **Five Phillies-specific agents:**
  - **Researcher** — live MLB data
  - **Franchise Historian** — 1883–present
  - **Sabermetrics Expert** — WAR / OPS+ / FIP
  - **Data Analyst** — stat comparisons
  - **Fan Culture Expert** — rivalries / traditions

## Visual Conventions

- **Base aesthetic:** warm cream background `#f7f5f0`, text `#2c2c2c`, Georgia serif (`Georgia, 'Times New Roman', serif`) throughout.
  - Exception: pokemon.html body uses a system sans-serif stack; its titles still use Georgia.
- **Accent colors:** china.html crimson `#C41E3A`; phillies.html Phillies red `#E81828`.
- **Dynasty cards (china.html)** have unique colored left borders via `.border-*` classes:
  gold `#B8960C`, deep red `#8B1A1A`, jade `#2E6B4F`, imperial blue `#1B3A6B`, purple `#4A1B6B`, crimson `#C41E3A`.
- **Sidebar (china.html):** `#ede9e0` background, 240px wide. Hidden below 768px with a `.sidebar-toggle` button that toggles `.open`.
- Cards/panels: white or `#fdfbf7`, rounded corners, soft `box-shadow`. Section labels are uppercase, letter-spaced, muted gray.

## Markdown Rendering

- Agent responses contain markdown (`**bold**`, `##` headings, `---` dividers) that must be converted to HTML via the **`markdownToHtml()`** function in china.html before rendering with `innerHTML`.
- **Always escape HTML first** (`escapeHtml()`) **for XSS safety, then apply markdown conversion.** `markdownToHtml()` already escapes before converting — preserve that ordering if editing.
- `renderFinalAnswer()` additionally splits off a trailing `Sources:` block and renders detected URLs as a sources footer.

## Workflow Conventions

- All changes flow through: **new Claude Code session → branch → PR → review → merge → delete branch.**
- Repo is currently **PUBLIC** — never commit API keys, secrets, or credentials to any file.
- **htmlpreview.github.io** is used to preview pages (does not work on private repos).

## Gotchas

- **SSE + CORS:** Worker responses must explicitly allow `anthropic-version` and `x-api-key` in CORS headers (this caused a prior debugging session). Keep these in the Worker's CORS config.
- **Merged PRs are immutable:** a merged PR cannot receive new commits. If a session continues after merge, open a **NEW PR** for further changes — do not try to push to the merged one.
- **Secret scanning revokes keys:** GitHub Actions secret scanning will revoke any API key committed to code, even in an edit/diff. Always use environment variables / Worker bindings, never hardcoded keys.
