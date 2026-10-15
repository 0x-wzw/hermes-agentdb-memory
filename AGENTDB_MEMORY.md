# AGENTDB Memory System — Unified Schema v2 + Hermes OS Flywheel + Obsidian Vault

**Unified agent memory system integrating Schema v2 frontmatter, Hermes OS flywheel intelligence pipeline, and Obsidian vault knowledge lifecycle.**

> **Status:** active • **Type:** system • **Domain:** knowledge • **Created:** 2026-10-15

---

## 1. Memory Tiers (Hot / Warm / Cold)

| Tier | Temperature | Location | Purpose | Max Size |
|------|-------------|----------|---------|----------|
| **Hot** | < 6000 chars | `~/.hermes/memories/MEMORY.md` + `~/.hermes/memories/USER.md` | Active session context, current projects, hot state | < 6000 chars combined |
| **Warm** | Daily-accessible | `System/Wiki/` (LLM Wiki tier) | Evergreen facts, concepts, entities, reference docs | Unlimited (generated) |
| **Cold** | Archive/backup | `.obliviarch/` (JSON compressed) + `System/_generated/` | Historical records, pruned captures, compressed traces | Unlimited (gitignored) |

---

## 2. Schema v2 — Frontmatter Contract (Canonical)

**Every note in the vault must carry the Schema v2 frontmatter.** This is the single source of truth for note classification.

### Required Fields (Core Contract)

| Field | Type | Question | Enforced |
|-------|------|----------|----------|
| `type` | string | **What IS this?** | ✅ everywhere |
| `created` | date | **When conceived?** | ✅ everywhere |
| `domain` | string | **What business domain?** | ✅ scoped (Knowledge/, System/Wiki/) |
| `status` | string | **Lifecycle state** | ✅ scoped (Knowledge/, System/Wiki/) |

### Controlled Vocabularies

- **`status`**: exactly 6 lowercase values — `unprocessed · active · draft · shipped · published · archived`
- **`type`**: 22 forms — `raw · wiki · concept · entity · comparison · query · report · distilled · template · index · moc · daily · handoff · analysis · reference · meeting-notes · guide · plan · workflow · interaction · purchase · health`

### Conditional/Optional Fields

| Field | Type | When Used |
|-------|------|-----------|
| `category` | string | Note variant classification |
| `slug` | string | Unique identifier |
| `timestamp` | string | Exact creation time (YYYYMMDD-HHmm) |
| `topics` | array | Discovery keywords (non-empty for new notes) |
| `area` | enum | Life domain: `work`, `home`, `self`, `social` |
| `project` | string | Short project code (e.g., `bam`, `mui`) |
| `entity` | array | Linked entities: `[john-doe, acme-corp]` |
| `source` | string | Origin system (manual, codex-chat, telegram, apple-mail, imessage, browser) |
| `source_id` | string | Stable source identifier |
| `thread_id` | string | Conversation/thread identifier |
| `reply_to` | string | Slug/source_id this note replies to |
| `agent` | string | Agent that produced this note |
| `url` | string | External URL for web/reference captures |
| `confidence` | string/number | Evidence confidence (high/medium/low or 0-1) |
| `contested` | boolean | True if note contains unresolved contradictions |
| `recurring` | string | For recurring report series (`daily`, `weekly`) |
| `distilled` | boolean | Reports: true once best practices extracted to `30-Distilled/` |
| `distilled_into` | string | Link to the distilled note that consumed this report |
| `pipeline` | CRM state (`Initial Contact`, `Proposal`, `KIV`, `Closed Lost`) |
| `cycle` | Knowledge-lifecycle cycle (e.g., `2026-Q4`) |
| `aliases` | array | Alternative names for Obsidian linking |
| `area` | enum | Life domain: `work`, `home`, `self`, `social` |
| `sources` | Array | Array of source paths/URLs for compiled synthesis |
| `source_hash` | SHA256 | Raw content for drift detection |
| `source_checked_at` | When source was last verified |
| `supersedes` | Older slugs this note replaces |
| `contradictions` | Related slugs that conflict with this note |

### Authorship Tracking

Every note written by an agent carries one authorship tag in `tags`:

| Tag | Author |
|-----|--------|
| `claude-code` | macOS Claude Code session |
| `macbook-pro-hermes-agent` | Hermes cron/agent on MacBook Pro (`Darwin`) |
| `ubuntu-server-agent-hermes` | Hermes cron/agent on Ubuntu VM (`Linux`) |
| `hermes-agent` | Legacy — pre-host-split; never applied to new notes |

> ⚠️ **Scripts must never hand-write these literals.** Use `hermes_feeds.frontmatter_block()` and `hermes_feeds.authorship_tag()`.

### Minimum Viable Frontmatter Example

```yaml
type: raw
category: note
domain: knowledge
created: 2026-10-15
slug: my-new-note
timestamp: 20261015-1430
status: unprocessed
topics: [keyword1, keyword2]
source: manual
tags: [optional, but, useful]
```

---

## 3. Knowledge Lifecycle (Raw → Wiki → Report → Distilled → Templates)

The vault knowledge lifecycle is a compounding pipeline. Each stage graduates forward and links back.

### Flow Direction

```
Knowledge/00-Raw/          →  capture, unprocessed (status: unprocessed)
      │  cite / promote
      ▼
System/Wiki/              →  evergreen facts (canonical wiki tier)
      │  ↑   ↓
      │  │  extract lessons
      ▼  ▼
Knowledge/20-Reports/    →  dated deliverables, frozen once shipped (status: published)
      │  extract lessons
      ▼
Knowledge/30-Distilled/  →  reusable best practices & workflows (status: active)
      │  instantiate
      ▼
Knowledge/40-Templates/  →  formats the next cycle uses (status: active)
```

### Folder-to-Lifecycle Mapping

| Folder | Lifecycle Position | Status When Done |
|--------|-------------------|------------------|
| `Knowledge/00-Raw/` | raw | `archived` once cited/promoted |
| `Knowledge/10-Wikis/` | **removed 2026-10-10** — staging drafts archived | — |
| `System/Wiki/` | evergreen facts | — |
| `Knowledge/20-Reports/` | dated outputs | `published` |
| `Knowledge/30-Distilled/` | compounding layer | `active` |
| `Knowledge/40-Templates/` + `_Scaffolds/` | reusable formats | `active` |

### Promotion Rules (When to Advance)

- **Raw → Wiki**: Evergreen fact that should outlive the cycle, cited by multiple reports
- **Raw → Report**: Dated deliverable with specific time-bound value
- **Report → Distilled**: Reusable lessons extracted, no longer time-bound
- **Distilled → Template**: Pattern generalized for future cycle use

### Link Fabrics

1. **Lifecycle links (vertical)**: Raw captures are cited by reports (`sources:` / body links); reports link their distilled notes (`distilled_into:`); distilled notes link back; wiki pages link the reports that fed them. Direction: always raw → wiki/report → distilled → templates; links point back down.

2. **Index links (horizontal)**: Generated daily by `~/.hermes/scripts/vault-rebuild-indexes.py`:
   - `System/Wiki/index.md` — every wiki page by section
   - `Knowledge/Knowledge MOC.md` — the whole lifecycle tree
   - `Knowledge/00-Raw/_index.md` + per-subfolder indexes

3. **Health/ops links (machine)**: Each cron job writes `System/<job>-health.json`; `flywheel-health-check.py` aggregates them.

---

## 4. Hermes OS Flywheel — Intelligence Pipeline

The Hermes OS flywheel is the daily intelligence processing pipeline that runs atop the vault knowledge lifecycle.

### Flywheel Flow (Daily 06:00–08:20)

```
P[Producer crons 06:00-07:00]
    │
    ├─► raw captures (schema v2) ──► V[vault Knowledge/00-Raw]
    │
    └─► L[daily-intelligence-loop 08:00]
          │
          ├─► Daily-Intelligence-Brief ──► R[Knowledge/20-Reports]
          │
          └─► P2[intelligence-publish 08:15]
                │
                ├─► N[Notion - 1 page/day]
                │
                ├─► KV[intelligence.teoh.my Cloudflare KV]
                │
                ├─► F[intelligence-notify 08:20]
                │    │
                │    ├─► TG[Telegram home] (link + stats only)
                │    └─► LK[Lark home chat] (link + stats only)
                │
                └─► Weekly prune Sun 07:30
```

### Components

| Stage | Schedule | Script | Output |
|-------|----------|--------|--------|
| **Producers** (6 verticals) | 00:00–05:30 | `market-trends*.py` ×6 | `Knowledge/00-Raw/trends/*.md` (vault-only) |
| **Policy / venture / arXiv / thought-leaders / podcasts / LLM-scan** | 04:30–06:45 | respective scripts | `Knowledge/00-Raw/{policy,research,thought-leaders,podcasts}/` |
| **Loop closer** | 08:00 | `daily-intelligence-loop.py` | `Knowledge/20-Reports/Daily-Intelligence-Brief-YYYY-MM-DD.md` + daily-note entry |
| **Distilled publisher** | 08:15 | `intelligence-publish.py` | Notion: 1 page/day, KV dashboard + mirror |
| **Link share** | 08:20 | `intelligence-notify.py` | Telegram home + Lark home chat (link + stats only) |
| **Weekly prune** | Sun 07:30 | `intelligence-prune.sh` → `intelligence-prune.py --apply` | `.trash/` + indexes + KV dated-key cleanup |

### Rules Enforced in Code

1. **Producers never write to Notion.** All six market-trends scripts have `USE_NOTION = False`. Vault writes go through `hermes_feeds.frontmatter_block()` (Schema v2: type/created/domain/status/cycle/authorship).

2. **Notion gets the distillation, not the firehose.** One page per day (`Daily Intelligence — YYYY-MM-DD`), executive synthesis + ≤3 top signals per vertical. Raw article lists stay in the vault.

3. **Idempotent publish.** The publisher queries by Date+Category before creating; re-runs update the existing page (archive-then-append children) instead of duplicating.

4. **Stop-ship gate.** `publication_gate()` withholds publication when ≥3 of the 5 commercial verticals are empty (false-completeness guard).

5. **Single taxonomy.** `intelligence_source_of_truth.PRODUCER_LIST` remains the only place producer categories are defined; both publisher copies import it.

6. **Link-only human touchpoint.** Telegram/Lark get the Notion URL + counts, never article dumps. The Notion URL is passed via `~/.hermes/logs/notion-daily-page-url.txt`.

7. **Retention: prune only what is ingested.** `intelligence-prune.py` (weekly, Sundays 07:30) deletes raw captures only when (a) >30 days old AND a `Daily-Intelligence-Brief-<date>` with `synthesis: true` exists, or (b) >60 days absolute ceiling. Everything moves to `.trash/` (recoverable), never `rm`. Never pruned: `seen-urls.json`, `arxiv-seen.json`, 20-Reports briefs.

### Indexes (Duplicate Prevention & Retention Ledger)

| Index | Path | Purpose |
|-------|------|---------|
| **Seen-URLs store** | `~/.hermes/data/seen-urls.json` | Producer dedup memory — must outlive captures — never prune |
| **arXiv seen store** | `~/.hermes/arxiv-seen.json` | Hash-based paper dedup (~7.5k entries). Never prune |
| **Raw-captures index** | `System/raw-captures-index.json` | Regenerated each prune run: live inventory of every raw capture |
| **Pruned-captures index** | `System/pruned-captures-index.json` | Append-only ledger of every pruned file + URLs it contained |
| **KV dated keys** | `daily-insights-YYYY-MM-DD` | Dashboard history. Keys >30d deleted weekly; current view persists |

### Paused / Removed

- `[bot:vaultkeeper] daily-insights-publish` (id `7ed16ae8ec2b`, 07:45) — **paused**, not deleted. Bypassed the vault capture layer and collided with the 08:15 publisher.

---

## 5. Vault Harness — Folder & Relationship Map

The Obsidian vault is organized into distinct zones with clear mapping rules.

### Top-Level Map

```
Obsidian Vault/
├── Daily/             ← daily notes, one per day (YYYY-MM-DD.md) — append-only log
├── Inbox/             ← UNPROCESSED intake (mostly Notion syncs) — triage queue
├── Knowledge/         ← the knowledge lifecycle (raw → report → distilled → templates)
├── People/            ← person dossiers (Career History, bios)
├── Personal/          ← personal projects & life notes (non-work)
├── System/            ← how the machine works: wiki, schema, context, health
├── Work/              ← client/project workspaces (MUI, AISED, Business/*, EOI, Tasks)
├── handoffs/          ← ALL session handoffs — single register (handoffs/_index.md)
├── _embeddings/       ← machine-owned vector batches (.npy) — gitignored, do not edit
└── (dot-dirs)         ← .git .obsidian .obliviarch .trash .wrangler — app-owned
```

### Knowledge Lifecycle (Knowledge/ ⇄ System/Wiki/)

```
Knowledge/00-Raw/          capture, unprocessed
      │  cite / promote
      ├──────────────► System/Wiki/            evergreen facts (canonical wiki tier)
      │
      ├──► Knowledge/20-Reports/               dated deliverables, frozen once shipped
      │            │  extract lessons
      │            ▼
      ├────► Knowledge/30-Distilled/           reusable best practices & workflows
      │            │  instantiate
      │            ▼
      └────► Knowledge/40-Templates/           formats the next cycle uses
                   +  Knowledge/_Scaffolds/             note starters (emit conforming frontmatter)
```

### System/ — How the Machine Works

| Path | Role |
|------|------|
| `System/AGENTS.md` | Vault Context Protocol — what every agent reads first |
| `System/SCHEMA.md` | Frontmatter contract (Schema v2) |
| `System/VAULT-STRUCTURE.md` | This document — folder/file/relationship map |
| `System/Assistant/` | Living context: PREFLIGHT.md, context.md, preferences.md, environment.md |
| `System/Wiki/` | Canonical knowledge base — index generated, change log, schema addendum |
| `System/Intelligence/` | Published intelligence site (Cloudflare Pages) |
| `System/_generated/` | Machine-written only: daily flywheel-health-*.md, never hand-edit |
| `System/Backups/` | Recovery artifacts, schema remediation records, index backups |
| `System/*.md` (root) | Operational docs: Intelligence-Cron-Reference, intelligence-flywheel, etc. |
| `System/*-health.json` | One health file per cron job — flywheel check reads these |

### Conventions (enforced elsewhere, summarised here)

- **Frontmatter**: Schema v2 (`System/SCHEMA.md`) — every note carries `type`, `created`, `domain` (+ `status` under Knowledge/ and System/Wiki); new notes add `category`, `slug`, `timestamp`, `topics`, `source`. Authorship tag in `tags` via `hermes_feeds.authorship_tag()`.
- **Cron writers** build frontmatter with `hermes_feeds.frontmatter_block()` and must never hand-write authorship literals.
- **Status vocabulary** is exactly six lowercase values: `unprocessed active draft shipped published archived`.
- **Same-day report protection**: cron report writers never overwrite a better same-day report.
- **Idempotent writers**: scripts that maintain wiki pages must skip writes when nothing but `updated:` would change.

### Decision Table — "Where Does a New File Go?"

| The file is… | Put it in |
|--------------|-----------|
| An unprocessed capture (clip, transcript, cron output) | `Knowledge/00-Raw/` (subfolder by family) |
| An evergreen fact that should outlive the cycle | `System/Wiki/` (entities/concepts/comparisons/queries) |
| A finished dated deliverable | `Knowledge/20-Reports/` |
| A reusable lesson extracted from shipped work | `Knowledge/30-Distilled/` |
| A session handoff / continuity doc | `handoffs/` + register in `handoffs/_index.md` |
| Project-scoped analysis for a client | `Work/<project>/` |
| A run summary from today's work | `Daily/YYYY-MM-DD.md` (append) |
| Machine-generated output | `System/_generated/` (gitignored) or the job's `00-Raw/` sink |
| A recovery artifact / old version | `System/Backups/` |
| A daily-operational log entry | `Daily/` — never a new top-level folder |

### Anti-patterns (each caused a real problem once)

- A second wiki tier (`10-Wikis`) — removed 2026-10-10
- A second handoff home (handoffs at vault root or `System/`)
- Index copies (`20-Reports/Wiki-index.md`)
- Duplicate rolling writes (`00-Raw/trends/TODAY.md`)
- Empty stubs at root
- Dead placeholder projects (`knowledge-kb/`)

All removed/archived 2026-10-10 — keep it that way.

---

## 6. Unified Memory Architecture — Integration Summary

### Hot Memory (`~/.hermes/memories/MEMORY.md`)

- **Current session state**: active projects, recent cron results, flywheel health snapshot
- **Schema v2 compliance**: compliance percentage, common violations, recent fixes
- **Hermes OS flywheel**: last run status, next scheduled jobs, pipeline health
- **Vault status**: raw capture count, wiki page count, report/distilled counts
- **Memory budget**: keep under 6000 chars; promote stable entries to vault files at ~67% capacity

### Warm Memory (`System/Wiki/`)

- **Evergreen facts**: all wiki pages with Schema v2 frontmatter
- **Concept/Entity/Comparison/Query pages**: the structured knowledge base
- **Index pages**: `System/Wiki/index.md`, `Knowledge/Knowledge MOC.md`
- **Schema compliance**: tracked weekly via `vault-schema-lint.py`

### Cold Memory (`.obliviarch/` + `System/_generated/`)

- **Historical records**: past flywheel runs, pruned captures
- **Compressed traces**: JSON archives, vector batches (.npy)
- **Backup artifacts**: `System/Backups/`, cron job health history
- **Recovery data**: `.trash/` recoveries, index backups

### Integration Points

1. **Schema v2 → Flywheel**: All producer cron scripts emit Schema v2 frontmatter when writing to `Knowledge/00-Raw/`. The flywheel reads these captures and synthesizes the Daily Intelligence Brief.

2. **Flywheel → Vault**: The daily loop (`daily-intelligence-loop.py`) reads from `Knowledge/00-Raw/` and writes to `Knowledge/20-Reports/`. The publisher (`intelligence-publish.py`) then distills to Notion/KV.

3. **Vault → Schema**: All vault notes must conform to Schema v2. The linter (`vault-schema-lint.py`) enforces compliance weekly.

4. **Memory tiers → AgentDB**: 
   - Hot: `MEMORY.md` + `USER.md` (session state)
   - Warm: `System/Wiki/` (evergreen knowledge)
   - Cold: `.obliviarch/` + `System/_generated/` (historical archive)

5. **Authorship**: All agent-written notes carry `hermes-agent` or platform-specific authorship tag via `hermes_feeds.authorship_tag()`.

---

## 7. Cron Job Integration (43 Jobs + Vault Maintenance)

The Hermes OS flywheel is driven by cron jobs organized by profile:

### transformation-architect profile (43 jobs — the intelligence flywheel)

| Category | Jobs | Purpose |
|----------|------|---------|
| **Market trends** (6) | `market-trends`, `hospitality-trends`, `fnb-trends`, `property-infra-energy-trends`, `csuite-leadership-trends`, `ai-science-investment-trends` | Fetch latest market trends, push to Notion |
| **Content pipeline** | `content-pipeline`, `market-intelligence`, `policy-tracker`, `podcast-transcribe` | Content processing and analysis |
| **Dealcrusher** (3) | `bias-balance-check`, `signal-noise`, `sied-kpi-report` | Quality and KPI checks |
| **Analyst** (5) | `action-nudge`, `intelligence-grilling`, `category-contamination-check` | Analysis and grilling sessions |
| **Vault keeper** (5) | `notion-filing-chain`, `obliviarch-consolidation`, `vault-daily`, `schema-lint`, `intelligence-prune` | Vault maintenance |
| **Other** (24) | Various model routing, OODA bots, email attention, etc. | System operations |

### vaultkeeper profile (3 jobs + launchd)

| Job | Schedule | Script |
|-----|----------|--------|
| `vault-sync` (launchd) | Every 2h | `vault-daily.py` |
| `obliviarch-consolidation` | Daily 6AM | `obliviarch-unified.py` |
| `vault-semantic-index` | Daily 5:15AM | `vault-index-cron.py` |

### Launchd Agents

- `ai.hermes.vault-sync` every 2h on the host
- `ai.hermes.vault-autocommit` every 15min

---

## 8. AgentDB Initialization Checklist

When initializing a new agent session or rehydrating the memory system:

- [ ] Read `System/AGENTS.md` first (vault context protocol)
- [ ] Read `System/Assistant/PREFLIGHT.md` (vault rules orientation)
- [ ] Confirm `System/VAULT-STRUCTURE.md` is current (map agrees with reality)
- [ ] Verify Schema v2 compliance on all new notes (`hermes_feeds.frontmatter_block()`)
- [ ] Check `~/.hermes/memories/MEMORY.md` < 6000 chars
- [ ] Check `~/.hermes/memories/USER.md` < 1375 chars
- [ ] Confirm `HERMES_RAG_DATA=~/.hermes_rag` and `HERMES_EMBED_DIM=1024` env vars
- [ ] Verify `ai.hermes.rag-health` launchd last run is `ok` in `~/.hermes/logs/hermes-rag-health.json`
- [ ] Check `~/.hermes/.env` for valid API tokens (NOTION_API_TOKEN, TELEGRAM_BOT_TOKEN, etc.)
- [ ] Confirm `ILMU_API_KEY` is live (health monitor gated by key liveness)
- [ ] Review `profile.yaml` for current UI metadata
- [ ] Verify `config.yaml` model routing is current

---

## 9. Quick-Reference Cheatsheet

### Frontmatter Block (generate via helper)

```python
hermes_feeds.frontmatter_block({
    "type": "raw",        # or wiki, report, entity, etc.
    "created": "2026-10-15",
    "domain": "knowledge",
    "status": "unprocessed",
    "category": "note",
    "slug": "my-new-note",
    "timestamp": "20261015-1430",
    "topics": ["keyword1", "keyword2"],
    "source": "manual",
    "tags": ["optional", "but", "useful"]
})
```

### Status Values (exactly 6 lowercase)

`unprocessed · active · draft · shipped · published · archived`

### Lifecycle Folder Mapping

| Folder | Type | Status |
|--------|------|--------|
| `Knowledge/00-Raw/` | raw | `archived` when promoted |
| `System/Wiki/` | wiki/ evergreen | — |
| `Knowledge/20-Reports/` | report | `published` |
| `Knowledge/30-Distilled/` | distilled | `active` |
| `Knowledge/40-Templates/` | template | `active` |

### Daily Flywheel Hours

| Time | Activity |
|------|----------|
| 00:00–05:30 | Producers (6 verticals run) |
| 04:30–06:45 | Policy/venture/arXiv/thought-leaders/podcasts/LLM-scan |
| 06:00–07:00 | Vault sync (launchd) |
| 07:30 | Weekly prune (Sundays) |
| 08:00 | Daily-intelligence-loop |
| 08:15 | Intelligence-publish (Notion/KV) |
| 08:20 | Intelligence-notify (Telegram/Lark link share) |

---

**End of AGENTDB Memory System**