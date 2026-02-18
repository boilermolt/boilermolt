# Boilermolt — Command Cheat Sheet

_What to say to get things done fast. Keep this page updated when we add new “UX commands” or automations._

## 1) Daytime command channel (Email → Proton inbox)
When you can’t access Telegram/social during the day:
- Send commands to Boilermolt via email to the agent’s Proton inbox.
- **Allowed senders** (Boilermolt will read):
  - `Chris.Wylde@brucepower.com` (work email)
  - `128boilerrat@gmail.com`

**Important rules**
- Boilermolt will **never** reply to `Chris.Wylde@brucepower.com`.
- Boilermolt **may** reply to `128boilerrat@gmail.com`.

**How to write command emails**
- Put the command in the first line.
- Keep it explicit (what to do + desired output file + whether to email results).
- If you want a follow-up question instead of execution, say “ask me questions first.”

Examples:
- “Create a DAO digest for tomorrow with extra coverage on grants programs; save to Obsidian.”
- “Add Todoist task: pay hydro due Friday @Admin”

---

## 2) Todoist (natural-language command phrasing)
Preferred phrasing:
- **`add task: <content> due <due-string> @<Label>`**

Notes:
- Default project: **Claw Tasks**
- Labels can be created on-demand (if you say `@Overdue`, it will be created if missing).

Examples:
- `add task: file taxes due 2026-04-30 @Admin`
- `add task: call dentist due next Tue @Health`

---

## 3) DAO Digest outputs (Obsidian + DOCX for Substack)
Daily DAO digest is written into Obsidian (vault-friendly, wikilinks) and now also exported to a Substack-friendly Word doc.

Files:
- Obsidian note:
  - `Claw/AI/Boilermolt/DAO/Daily Digests/YYYY-MM-DD.md`
- Source markdown (workspace):
  - `~/clawd/reports/dao/YYYY-MM-DD.md`
- **DOCX export (for Substack copy/paste):**
  - `~/clawd/reports/dao/YYYY-MM-DD.docx`

If Substack paste is the goal, ask for “DOCX export” explicitly.

---

## 4) Markets / Portfolio emails (HTML preference)
All emailed reports should be **HTML** (Gmail-friendly) — not markdown.

If you’re requesting a new email/report, you can specify:
- “Send as HTML email.”
- “Keep it readable on mobile.”

---

## 5) DAO ‘wisdom’ posting UX (X)
Character limits + long source URLs are a constant issue.

Default posting patterns:
- **Two-tweet mini thread** (recommended): 1 idea + 1 link per tweet.
- Single tweet: 1 idea + (at most) 1 link.

If you want a post drafted:
- “Draft a 2-tweet thread (1 link each) summarizing today’s spiciest governance lesson.”

---

## 6) Vacation Planner
Daily HTML email with **10 deal cards** sent to:
- `128boilerrat@gmail.com`
- `scwylde1@gmail.com`

If you want tweaks, say:
- “Vacation planner: tighten to smaller resorts” / “more Mexico” / “no Dominican” / “raise budget ceiling” / etc.

---

## 7) last30days (planned skill)
Status: **planned / being implemented**.

Goal: “What mattered in the last 30 days” research across:
- Web + Reddit discovery (no creds in v1)
- X (using current setup)
- Substack discovery via web search (Option A)

Proposed usage:
- `last30 <topic>`
- `last30 <topic> --days 7`
- `last30 <topic> --deep`

---

## 8) General request patterns that work well
- **“Give me 3 options”** (headlines, hooks, outlines, post copy)
- **“Make this Substack-ready”** (tighten + structure + strong lead)
- **“Obsidian-ready”** (wikilinks + entity stubs + index updates)
- **“Only ping me when action changes”** (useful for trading/status loops)

---

## 9) Installed skills — useful commands you can ask for

### Web search (fallbacks)
- **Bing (no API key)**
  - `search bing: dao governance forum --num 10`
  - Under the hood:
    ```bash
    python3 skills/bing-search/scripts/search.py "dao governance forum" --num 10
    ```
- **Tavily (clean AI search)**
  - `search tavily: <query> (deep|news|n=10)`
  - Under the hood:
    ```bash
    node skills/tavily-search/scripts/search.mjs "query" -n 10
    node skills/tavily-search/scripts/search.mjs "query" --deep
    node skills/tavily-search/scripts/search.mjs "query" --topic news --days 7
    ```
  - Extract a page:
    ```bash
    node skills/tavily-search/scripts/extract.mjs "https://example.com"
    ```

### Color palettes (Colormind)
- `colormind models`
  ```bash
  node skills/colormind/scripts/list_models.mjs
  ```
- `colormind palette ui` / `colormind palette default`
  ```bash
  node skills/colormind/scripts/generate_palette.mjs --model ui --pretty
  ```
- `colormind from image <path>`
  ```bash
  bash skills/colormind/scripts/image_to_palette.sh /path/to/image.jpg --model ui
  ```

### Writing style
- `humanize: <text>` (make it sound less AI / more “you”)

### Memory utilities
- `remember: <facts>` / `what did we decide about <topic>?`
  - (Internal helpers exist; you can just ask in plain English and I’ll log/recall.)

### Deep research (local-first)
Default deliverable: **Obsidian**

- `deep research: <question>`
- `deep research: <question> last 30 days`
- `deep research: <question> --deliverable obsidian|substack|github`
  - Includes Substack discovery (Option A) via web search: `site:substack.com <topic>`

### Proton email (agent-side)
- Send an email (HTML supported):
  ```bash
  python3 ~/clawd/scripts/send_email.py --to you@example.com --subject "Hi" --body "..." --html
  ```

### Trello (working)
Primary board: **Boilermolt**

Default lists:
- Backlog
- Planning
- In Progress
- Review
- Complete

Email-friendly command phrasing (what to write):
- `trello: add "<card title>" to Backlog`
- `trello: add "<card title>" to Planning`
- `trello: move "<card title>" to In Progress`
- `trello: move "<card title>" to Review`
- `trello: complete "<card title>"` (moves to Complete)
- `trello: list In Progress`
- `trello: list Review`
- `trello: comment "<card title>" -> <comment text>`

Defaults:
- If you omit the board name, I assume the **Boilermolt** board.

(Under the hood we use the Trello API; see `skills/trello/SKILL.md` for raw curl examples.)

### Base DEX Trader (paper)
- `run base paper trader now`
  ```bash
  node skills/base_dex_trader/scripts/run.js --config skills/base_dex_trader/config.example.json
  ```

### Personal brand site scaffold
- `scaffold personal brand site`
  ```bash
  bash skills/personal-brand-site/scripts/create-site.sh
  ```

---

## 10) When to ask for a confirmation
If you’re about to have the agent do anything external (posting, sending email, changing schedules):
- “Draft first, don’t send/post until I confirm.”
