# Vybora — Marketing and Partnership Opportunities Agent

An n8n-based autonomous agent that runs nightly, scrapes a configurable source
registry, extracts opportunities into a fixed schema, scores them against
per-task-type weights, detects changes vs. the previous run, and emails a
nightly report plus a separate "top pick" alert. Failures trigger a dedicated
error-alert workflow.

## Repo layout

`
## Repo layout

`

├── Vybora Marketing and Partnership Opportunities Agent.json # main workflow

├── Vybora --- Error Alert.json # error-handler workflow

├── Readme.md # this file

│

├── Architecture/

│ └── Architecture.jpg # canvas + flow diagram

│

├── Demo Output Images/

│ ├── Email Inbox Image.png # nightly email as received

│ ├── Report.jpg # full report body

│ └── Top pick.jpg # top-pick alert body

│

└── n8n-files/ # runtime data (import into n8n's /data if you are using docker)

├── opportunities.jsonl # append-only memory of every opportunity

├── sources.json # source registry

├── sessions/

│ ├── session-log.jsonl # one line per session summary

│ └── session_id_146.jsonl # per-session staging (scored rows)

└── tasks/

├── community.json # weights + rationale for community type

├── conference.json # weights + rationale for conference type

└── grants.json # weights + rationale for grants type

`
`

## File-by-file

### Workflows
- **`Vybora Marketing and Partnership Opportunities Agent.json`** — the main workflow. Schedule Trigger (cron `0 2 * * *`, Asia/Kolkata) → read `sources.json` → attach task weights → loop over sources → HTTP fetch → HTML extract → per-page size cap → LLM extract + score → normalize + ID → diff vs previous run → persist to `opportunities.jsonl` → score → build report → send full report + top-pick alert → write session log. Exposes an `errorWorkflow` link to the alert workflow below.
- **`Vybora — Error Alert.json`** — Error Trigger → Gmail. Sends the failing workflow name, execution ID/URL, last node, timestamp, error message and first 500 chars of the stack. Wired via the main workflow's `errorWorkflow` setting.

### Architecture
- **`Architecture/Architecture.jpg`** — the n8n canvas plus the node-level flow (input → extract → verify-score → memory → diff → report → notify). Use this to follow the data path.

### Demo Output Images
- **`Email Inbox Image.png`** — the nightly email landing in the inbox (subject line: `Vybora nightly — N new, M changed`).
- **`Report.jpg`** — the rendered full report (Top pick, Action needed, What's new, Deadlines in 14 days, parse-error line if any).
- **`Top pick.jpg`** — the standalone top-pick notification (separate email so it works as a push).

### Runtime data (`n8n-files/`)
- **`opportunities.jsonl`** — append-only memory. Every extracted opportunity is one JSON line, carrying `opportunity_id`, `status` (`new` / `changed` / `unchanged`), `score` (1–5), `ranking_explanation`, `signals`, `signal_justification`, `weights_source`, `checked_date`, `source_urls`, `conflicts`. Diffing against this file is what produces "new vs changed vs unchanged".
- **`sources.json`** — the source registry. Each entry declares `urls`, `feed_type` (`html_public` / `rss`), `opportunity_types`, `access_type` (`free` / `login` / `email` / `paid` / `sponsored` / `captcha` / `403_blocked`), `reliability`, `update_frequency`, `coverage`, `check_cadence`, `verified`, `last_verified_at`, `notes`. A sentinel `{ "__is_last": true }` marks the end of the loop.
- **`sessions/session-log.jsonl`** — one summary line per nightly run: `run_at`, `session_id`, `sources_checked`, `new`, `changed`, `unchanged`, `parse_errors`, `errors`, `top_pick`, `total_rows`. This is the run log.
- **`sessions/session_id_146.jsonl`** — the per-run staging file written by the Score node. Contains the fully-scored rows for that execution; consumed by the Load Run node to build the report. Included as a concrete example of what a run produces.
- **`tasks/community.json`, `tasks/conference.json`, `tasks/grants.json`** — per-opportunity-type scoring configs. Each defines `weights` (sum to 1.0) and `rationale` (why each weight is what it is). The workflow loads all of these at startup and attaches the matching weights to each source via its `_task_type`. If a source's type has no task file, it falls back to `{ audience_fit: 0.5, cost_efficiency: 0.5 }` and records `weights_source: "FALLBACK_default"` on every row so it is visible in the output.

## How to run

1. Import both JSON files into n8n (Workflows → Import from File).
2. Place the `n8n-files/` contents at `/data` inside the n8n container (so `sources.json`, `tasks/`, `sessions/`, and `opportunities.jsonl` are all under `/data`).
3. Connect credentials:
   - **Google Gemini (PaLM) API** on the `Google Gemini Chat Model` node.
   - **Gmail OAuth2** on the two send nodes and on the error-alert node.
4. Set the destination email on the three Gmail nodes (currently `swaroop.ratnaparkhi@gmail.com`).
5. Run the main workflow manually once, then enable it. It will fire at 02:00 Asia/Kolkata.

## Changing task weights (no code)
Edit or add a file under `tasks/<task_type>.json`. Weights must sum to 1.0. The workflow picks it up on the next execution; every row records `weights_source` so you can see which file produced the score.

## Assumptions
- Vybora is a startup helping corporate professionals communicate better.
- Reference cities in `sources.json` are Bengaluru and US tech hubs; the registry is the only place a new city or source is added.
- `access_type` in `sources.json` is treated as ground truth for access classification; the LLM's `access_class` is informational and disagreements are recorded in `conflicts`.

## Known limits
- **Verification is single-source.** Each fact is cited with `checked_date` and `source_urls`, but a second-source cross-check is not yet automated; rows are therefore effectively "single-source cited", not "cross-verified".
- **Opportunity-type coverage in this build is 3** (community, conference, grants). The engine itself is type-agnostic — adding `media`, `partnership`, `pitch_competition`, `accelerator`, `meetup` is a `tasks/<type>.json` + `sources.json` entry, no code.
- **Cost cap is character-count based** (100000 characters per <i>run</i>). It is not an LLM-token cap.
- **The source registry is small.** It needs to have more sources including diverse channels - APIs, html_pages, rss.
## What I'd add with more time
- A real Verify node: second-source fetch + field-level `verified` flag and `conflicts` merge.
- Token-level cost cap.
- Remaining opportunity types with task files and sources (media, partnerships, pitch comps, accelerators, meetups).
- A more descriptive email report.
