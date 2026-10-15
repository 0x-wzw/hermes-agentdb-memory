# AGENTDB SYNC PROTOCOL — GitHub-First Update Workflow

**Purpose:** Ensure all updates and improvements to the Hermes agentDB memory system first hit the GitHub repository (`0x-wzw/hermes-agentdb-memory`) before being implemented on Hermes OS or server. This guarantees the GitHub repo is the single source of truth, with local environments synchronized from it.

**Last Revised:** 2026-10-15

---

## Sync Architecture Overview

```
GitHub Repo (0x-wzw/hermes-agentdb-memory)
              │
              ▼
Sync Engine (this protocol)
              │
    ┌───────────┴─────────────┐
    │                       │
    ▼                       ▼
Local Hermes OS           Server Environment
(~/.hermes/)              (production/deployment)
    │                       │
    └───────────┬───────────┘
                ▼
      Obsidian Vault
  (/Users/admin/Documents/
   Obsidian Vault/)
```

---

## Sync Workflow — "GitHub-First" Path

### Phase 1: Local Development (On Developer Machine)

**Step 1.1 — Create Update**
- Modify `AGENTDB_MEMORY.md`, `MEMORY.md`, or `USER.md` in the local agentDB repo
- Or modify vault files directly: `System/VAULT-STRUCTURE.md`, `System/SCHEMA.md`, `Knowledge/` lifecycle files

**Step 1.2 — Commit Locally**
```bash
cd /Users/admin/agentdb-memory-system
git add -A
git commit -m "brief description of changes [sync-YYYYMMDD]"
git push origin main
```

**Step 1.3 — Create GitHub Release (Optional but Recommended)**
- Tag the commit: `git tag -a v2026.10.15-sync -m "Sync: agentDB memory system update"`
- `git push origin v2026.10.15-sync`
- Or create a GitHub Release via the web interface

---

### Phase 2: Sync Engine — GitHub → Local Hermes OS

**Step 2.1 — Pull from GitHub (Run on Developer Machine)**
```bash
# From the agentDB repo directory
cd /Users/admin/agentdb-memory-system
git pull origin main
# Or specific tag:
# git pull origin v2026.10.15-sync
```

**Step 2.2 — Validate Schema v2 Compliance**
Run the schema linter to ensure all notes conform:
```bash
# If the linter script exists or can be invoked
python3 -m hermes_feeds vault-schema-lint.py 2>&1 | head -20
# Or manual check:
# - Every note has: type, created, domain, status
# - Status is one of: unprocessed active draft shipped published archived
# - No emoji or capital letters in status field
```

**Step 2.3 — Backup Current Local State**
Before applying sync, backup the current Hermes memory files:
```bash
cp /Users/admin/.hermes/memories/MEMORY.md /Users/admin/.hermes/memories/MEMORY.md.backup-$(date +%Y%m%d%H%M)
cp /Users/admin/.hermes/memories/USER.md /Users/admin/.hermes/memories/USER.md.backup-$(date +%Y%m%d%H%M)
cp /Users/admin/.hermes/memories/AGENTDB_MEMORY.md /Users/admin/.hermes/memories/AGENTDB_MEMORY.md.backup-$(date +%Y%m%d%H%M)
```

**Step 2.4 — Apply Synced Files to Local Hermes OS**
Copy the validated files from the agentDB repo to the Hermes OS memory directory:
```bash
cp AGENTDB_MEMORY.md /Users/admin/.hermes/memories/AGENTDB_MEMORY.md
cp MEMORY.md /Users/admin/.hermes/memories/MEMORY.md
cp USER.md /Users/admin/.hermes/memories/USER.md
```

**Step 2.5 — Update Vault Schema References**
Ensure the vault knows about the new schema version:
```bash
# Update System/SCHEMA.md if frontmatter contract changed
# Update System/VAULT-STRUCTURE.md if folder relationships changed
# Run vault rebuild if needed
python3 -m hermes_feeds vault-rebuild-indexes.py 2>&1
```

**Step 2.6 — Trigger Hermes OS Reinitialization**
Inform the Hermes OS of the new memory state:
```bash
# Log the sync event
echo "SYNC: $(date '+%Y-%m-%d %H:%M') - GitHub-fresh agentDB memory applied" \
    >> ~/.hermes/logs/sync-log.txt

# Signal any running cron jobs to refresh their schemas
hermes cron run schema-lint 2>&1 || true

# Update model routing if model lists changed
# (config.yaml is managed separately; reference only)
```

**Step 2.7 — Validate Post-Sync State**
Run validation checks:
```bash
# Check memory budgets are still within limits
wc -c /Users/admin/.hermes/memories/MEMORY.md
wc -c /Users/admin/.hermes/memories/USER.md

# Verify Schema v2 compliance on new notes
# Check flywheel health
cat ~/.hermes/logs/hermes-rag-health.json

# Confirm no broken references
hermes -p vaultkeeper check-memory 2>&1 | tail -5
```

---

### Phase 3: Sync Engine — Local → Server Environment

**Step 3.1 — Deploy to Server**
```bash
# Assuming server has the same directory structure
rsync -avz /Users/admin/.hermes/memories/ user@server:/path/to/hermes/memories/
rsync -avz /Users/admin/agentdb-memory-system/ user@server:/path/to/agentdb-repo/
```

**Step 3.2 — Server Validation**
On the server, run the same validation checks as Phase 2.5-2.7.

**Step 3.3 — Server Cron Restart**
Restart/reload affected Hermes cron jobs:
```bash
# If using launchd on macOS server
launchctl kickstart -w ai.hermes.vault-sync
# Or systemd on Linux server
systemctl restart hermes-vault-sync
```

---

### Phase 4: Obsidian Vault Sync

**Step 4.1 — Vault Memory Integration**
The Obsidian vault should mirror the Hermes memory system:

```bash
# Copy from Hermes memories to vault knowledge base
cp /Users/admin/.hermes/memories/AGENTDB_MEMORY.md \
   /Users/admin/Documents/Obsidian Vault/System/VAULT-STRUCTURE.md.agentdb
# Or symlink for real-time sync
ln -sf /Users/admin/.hermes/memories/AGENTDB_MEMORY.md \
       /Users/admin/Documents/Obsidian Vault/System/VAULT-STRUCTURE.md.agentdb
```

**Step 4.2 — Vault Index Rebuild**
```bash
# Rebuild wiki indexes from updated knowledge
cd /Users/admin/Documents/Obsidian Vault
python3 -m hermes_feeds vault-rebuild-indexes.py 2>&1
```

---

## Sync Safety Checks

### Pre-Sync Gates (Must Pass Before Push to GitHub)

| Check | Tool | Pass Criteria |
|-------|------|---------------|
| **Schema v2 compliance** | `vault-schema-lint.py` | 100% on all new/modified notes |
| **Memory budget** | `wc -c` | MEMORY.md < 6000 chars, USER.md < 1375 chars |
| **No secret leakage** | `git diff --cached` | No API tokens, passwords, or keys in diff |
| **Frontmatter integrity** | Custom Python script | Every note has required `type`, `created`, `domain`, `status` |
| **Status vocabulary** | Regex check | Status is exactly 6 lowercase values |

### Post-Sync Gates (Must Pass After GitHub → Local Sync)

| Check | Tool | Pass Criteria |
|-------|------|---------------|
| **File integrity** | `md5sum` / `sha256sum` | Synced files match GitHub versions |
| **Memory budget** | `wc -c` | Still within limits after sync |
| **Vault consistency** | Dataview query | No orphaned or mismatched notes |
| **Flywheel health** | `flywheel-health-check.py` | No new errors introduced |
| **Obliviarch consistency** | `obliviarch-unified.py` | Memory compression still valid |

### Rollback Procedure

If post-sync validation fails:

```bash
# Restore from backup
cp /Users/admin/.hermes/memories/MEMORY.md.backup-$(date +%Y%m%d%H%M) \
   /Users/admin/.hermes/memories/MEMORY.md
cp /Users/admin/.hermes/memories/USER.md.backup-$(date +%Y%m%d%H%M) \
   /Users/admin/.hermes/memories/USER.md
cp /Users/admin/.hermes/memories/AGENTDB_MEMORY.md.backup-$(date +%Y%m%d%H%M) \
   /Users/admin/.hermes/memories/AGENTDB_MEMORY.md

# Log the rollback
echo "ROLLBACK: $(date '+%Y-%m-%d %H:%M') - Sync failed validation, restored from backup" \
    >> ~/.hermes/logs/sync-log.txt

# Notify
hermes -p vaultkeeper notify "Sync rollback applied due to validation failure"
```

---

## Sync Command Reference

### Full Sync (GitHub → Local Hermes OS)

```bash
#!/bin/bash
# hermes-agentdb-sync.sh
# Usage: bash hermes-agentdb-sync.sh [--tag v2026.10.15] [--force]

SET -e

REPO_DIR="/Users/admin/agentdb-memory-system"
HERMES_MEM_DIR="/Users/admin/.hermes/memories"
GITHUB_REPO="0x-wzw/hermes-agentdb-memory"

# Parse arguments
TAG=""
FORCE=false
while [[ $# -gt 0 ]]; do
    case $1 in
        --tag) TAG="$2"; shift 2 ;;
        --force) FORCE=true; shift 1 ;;
        *) echo "Unknown option: $1"; exit 1 ;;
    esac
done

cd "$REPO_DIR"

# Step 1: If tag specified, checkout/fetch that tag
if [[ -n "$TAG" ]]; then
    git fetch origin tag "$TAG" 2>/dev/null
    git checkout "tags/$TAG" 2>/dev/null || \
    git pull origin "refs/tags/$TAG" 2>/dev/null || echo "Tag $TAG not found, pulling main"
fi

# Step 2: Pull latest from main
git pull origin main

# Step 3: Validate schema compliance
echo "=== VALIDATING SCHEMA v2 COMPLIANCE ==="
python3 -m hermes_feeds vault-schema-lint.py 2>&1 | \
    grep -E "(ERROR|FAIL|Compliance)" || echo "Schema check completed"

# Step 4: Backup current state
echo "=== BACKING UP CURRENT STATE ==="
mkdir -p "$HERMES_MEM_DIR/backups-$(date +%Y%m%d%H%M)"
cp "$HERMES_MEM_DIR/AGENTDB_MEMORY.md" "$HERMES_MEM_DIR/backups-$(date +%Y%m%d%H%M)/AGENTDB_MEMORY.md.bak"
cp "$HERMES_MEM_DIR/MEMORY.md" "$HERMES_MEM_DIR/backups-$(date +%Y%m%d%H%M)/MEMORY.md.bak"
cp "$HERMES_MEM_DIR/USER.md" "$HERMES_MEM_DIR/backups-$(date +%Y%m%d%H%M)/USER.md.bak"

# Step 5: Apply synced files
echo "=== APPLYING SYNCED FILES ==="
cp "$REPO_DIR/AGENTDB_MEMORY.md" "$HERMES_MEM_DIR/AGENTDB_MEMORY.md"
cp "$REPO_DIR/MEMORY.md" "$HERMES_MEM_DIR/MEMORY.md"
cp "$REPO_DIR/USER.md" "$HERMES_MEM_DIR/USER.md"

# Step 6: Rebuild vault indexes
echo "=== REBUILDING VAULT INDEXES ==="
cd /Users/admin/Documents/Obsidian Vault
python3 -m hermes_feeds vault-rebuild-indexes.py 2>&1 | tail -5

# Step 7: Post-sync validation
echo "=== POST-SYNC VALIDATION ==="
wc -c "$HERMES_MEM_DIR/MEMORY.md"
wc -c "$HERMES_MEM_DIR/USER.md"

# Step 8: Log completion
echo "SYNC: $(date '+%Y-%m-%d %H:%M:%S') - GitHub-fresh agentDB memory applied (tag: $TAG)" \
    >> "$HERMES_MEM_DIR/logs/sync-log.txt"

echo "=== SYNC COMPLETE ==="
```

### Make Executable
```bash
chmod +x /Users/admin/agentdb-memory-system/hermes-agentdb-sync.sh
```

---

## Sync Schedule & Triggers

### Recommended Sync Triggers

| Trigger | Frequency | When to Use |
|---------|-----------|-------------|
| **Manual sync** | On every significant update | Developer makes changes locally |
| **Pre-deployment sync** | Before any Hermes OS deployment | Ensures production has latest |
| **Weekly health sync** | Every Sunday 09:00 | Regular maintenance |
| **Post-cron-run sync** | After daily-intelligence-loop completes | Catch any pipeline-induced drift |
| **Emergency sync** | As needed | When flywheel health degrades or schema drift detected |

### Cron-Integrated Sync (Optional)

Add to `~/.hermes/profiles/transformation-architect/cron/jobs.json`:

```json
{
    "id": "sync-agentdb-memory",
    "name": "[bot:vaultkeeper] agentdb-sync",
    "prompt": "Sync agentDB memory from GitHub repo to local Hermes OS. Pull latest AGENTDB_MEMORY.md, MEMORY.md, USER.md, validate schema v2 compliance, rebuild vault indexes, and log the operation.",
    "skills": [],
    "skill": null,
    "model": "nemo-super",
    "provider": "ilmu",
    "base_url": null,
    "script": "hermes-agentdb-sync.sh",
    "no_agent": true,
    "schedule": {
        "kind": "cron",
        "expr": "0 9 * * 0",  # Sundays at 09:00
        "display": "09:00 every Sunday"
    },
    "enabled": true,
    "state": "scheduled",
    "fire_claim": null,
    "last_run_at": null,
    "last_status": "ok"
}
```

---

## Verification — Is the Sync Working?

After running the sync, confirm:

```bash
# 1. GitHub repo has the latest commit
gh repo view 0x-wzw/hermes-agentdb-memory --json=commits --limit 1

# 2. Local files match GitHub
cd /Users/admin/agentdb-memory-system
git log --oneline -1
# Should match: git log -1 on the server/repo

# 3. Memory budgets OK
wc -c /Users/admin/.hermes/memories/MEMORY.md
wc -c /Users/admin/.hermes/memories/USER.md
# Expected: < 6000 and < 1375 chars respectively

# 4. Schema v2 compliance
python3 -m hermes_feeds vault-schema-lint.py 2>&1 | grep compliance

# 4. Flywheel health no new errors
cat ~/.hermes/logs/hermes-rag-health.json

# 5. Sync log entry exists
tail -1 /Users/admin/.hermes/memories/logs/sync-log.txt
```

---

## Sync Protocol Checklist

### Before Any Change

- [ ] Change made in local agentDB repo (`/Users/admin/agentdb-memory-system/`)
- [ ] `git add -A` and `git commit -m "description [sync-YYYYMMDD]"`
- [ ] `git push origin main` (pushes to GitHub first)
- [ ] Optional: Create GitHub Release/tag

### After Push to GitHub

- [ ] Run sync: `bash hermes-agentdb-sync.sh` or equivalent
- [ ] Validate schema v2 compliance
- [ ] Check memory budgets within limits
- [ ] Rebuild vault indexes (`vault-rebuild-indexes.py`)
- [ ] Log sync operation in `~/.hermes/memories/logs/sync-log.txt`
- [ ] Trigger Hermes OS reinitialization as needed
- [ ] Run post-sync validation suite

### Before Hermes OS Deployment

- [ ] Confirm sync completed successfully
- [ ] Verify server environment has identical memory files
- [ ] Run server-side validation checks
- [ ] Restart affected cron jobs
- [ ] Monitor flywheel health for 24h post-deployment

### Rollback If Needed

- [ ] Restore from `backups-YYYYMMDDHHMM/` directory
- [ ] Log rollback event
- [ ] Investigate root cause of sync failure
- [ ] Re-run sync after fix applied

---

## Key Principles

1. **GitHub is the single source of truth** — Never implement on Hermes OS/server before GitHub has the change
2. **Validate before apply** — Schema v2 compliance and memory budgets must pass
3. **Backup before sync** — Always backup current state before applying synced files
4. **Full audit trail** — Sync logs must capture: what, when, who, outcome
5. **Rollback ready** — Backup copies must be kept and restorable
6. **Minimal blast radius** — Sync one component at a time when possible (AGENTDB_MEMORY.md, then MEMORY.md, then USER.md)

---

## End of AGENTDB SYNC PROTOCOL