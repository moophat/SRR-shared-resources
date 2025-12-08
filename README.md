# SRR-shared-resources

**Master repository for shared network templates and resources.**

This is the central repository containing authoritative copies of shared folders (Jinja2 templates, TextFSM parsers, JSNAPy tests, etc.) that are automatically synchronized to consumer repositories.

---

## What This Repo Does

**Central hub for shared resources** that automatically propagates to consumer repos:
- SRR-audit-tool
- SRR-monitoring-tool
- SRR-template-editor

Any change pushed to `main` triggers automatic downstream sync to all configured consumers.

---

## Repository Structure

```
SRR-shared-resources/
├── .github/
│   ├── consumer-config.yml              # Defines which folders sync to which consumers
│   └── workflows/
│       ├── populate-downstream.yml      # [Auto] Syncs updates to all consumers
│       ├── reverse-upstream.yml         # [Auto] Processes consumer changes
│       └── auto-pr-upstream.yml         # [Auto] Creates PR for upstream changes
├── grafana-dashboard/                   # Grafana dashboards
├── jinja2/                              # Jinja2 templates
├── jsnapy/                              # JSNAPy test templates
├── query-rule/                          # Network query rules
├── robottest/                           # Robot Framework tests
├── tableview/                           # Table view templates
├── textfsm/                             # TextFSM parsing templates
└── ttp/                                 # Template Text Parser templates
```

---

## How It Works

### Downstream Sync (This Repo → Consumers)

1. Developer pushes changes to any shared folder (e.g., `jsnapy/test-bgp.yml`)
2. GitHub Actions workflow `populate-downstream.yml` triggers
3. System syncs folders to all configured consumers
4. Auto-PR created in each consumer repo (sync-auto → main)
5. Consumer developers review and merge

**Time:** ~60 seconds for 3 consumers

### Upstream Sync (Consumers → This Repo)

1. Developer edits template in consumer repo
2. Consumer sends webhook to this repo
3. `reverse-upstream.yml` processes change, creates branch `from-<consumer>-<folder>`
4. `auto-pr-upstream.yml` creates PR
5. Maintainer reviews and merges
6. Change propagates back to all other consumers via downstream sync

**Time:** ~45 seconds + manual review

---

## Configuration

### consumer-config.yml

**Location:** `.github/consumer-config.yml`

Defines which folders sync to which consumers and where they're placed:

```yaml
audit-tool:
  repo: https://github.com/moophat/SRR-audit-tool.git
  folders:
    jsnapy: network_template/jsnapy
    robottest: network_template/robottest
    tableview: network_template/tableview

monitoring-tool:
  repo: https://github.com/moophat/SRR-monitoring-tool.git
  folders:
    grafana-dashboard: resource/grafana-dashboard
    query-rule: resource/query-rule
    tableview: resource/tableview

template-editor:
  repo: https://github.com/moophat/SRR-template-editor.git
  folders:
    jsnapy: template/jsnapy
    jinja2: template/jinja2
    ttp: template/ttp
    textfsm: template/textfsm
    tableview: template/tableview
```

**Key feature:** Same folder can map to different paths per consumer (e.g., `tableview` → 3 different locations).

---

## Workflows

### populate-downstream.yml
**Trigger:** Push to `main`
**Function:** Syncs all shared folders to configured consumers
**Output:** Auto-PR in each consumer repo

### reverse-upstream.yml
**Trigger:** `repository_dispatch` webhook from consumer
**Function:** Creates branch with consumer changes
**Output:** Branch `from-<consumer>-<folder>` pushed to this repo

### auto-pr-upstream.yml
**Trigger:** Invoked by `reverse-upstream.yml`
**Function:** Creates PR for upstream changes
**Output:** PR from `from-*` branch → `main`

**All workflows fully automated, no manual intervention required.**

---

## Making Changes

### Editing Shared Resources

```bash
# 1. Edit any file in shared folders
vim jsnapy/test-bgp.yml

# 2. Commit and push
git add jsnapy/test-bgp.yml
git commit -m "Update BGP test for Junos 23.4"
git push origin main

# 3. Watch Actions tab - populate-downstream.yml runs
# 4. Check consumer repos - PRs appear automatically
# 5. Consumer teams review and merge
```

**Do NOT:**
- Push directly to consumer repos' synced folders (changes get overwritten)
- Edit `sync-auto` branch in consumers (force-pushed on every sync)

### Adding New Consumer

1. Update `.github/consumer-config.yml`:
   ```yaml
   new-consumer:
     repo: https://github.com/org/new-consumer.git
     folders:
       jsnapy: path/to/jsnapy
   ```

2. Grant `GH_PAT` secret access to new consumer repo

3. Deploy workflows to new consumer (see specs)

4. Push to `main` - new consumer receives sync

---

## Branch Strategy

### Permanent Branches
- `main` - Production, triggers downstream sync on push

### Temporary Branches (Auto-Created)
- `split-<folder>` - Local only, exists during workflow execution
- `from-<consumer>-<folder>` - Upstream PR source branches (e.g., `from-audit-tool-jsnapy`)

**Manual cleanup needed periodically for `from-*` branches** (see troubleshooting).

---

## Loop Prevention

All automated commits are tagged with `[auto-sync]`:

```
[auto-sync][audit-tool] update jsnapy from consumer

Consumer commit: github.com/moophat/SRR-audit-tool@abc123
```

Consumer workflows skip commits containing `[auto-sync]` - prevents infinite sync loops.

---

## Required Secrets

### GH_PAT
**Type:** Fine-grained Personal Access Token
**Access:** All consumer repositories
**Permissions:**
- Contents: Read/Write
- Pull Requests: Read/Write
- Metadata: Read

**Used by:** All 3 workflows to clone consumers and create PRs

---

## Troubleshooting

### Sync Failed - Auth Error
**Cause:** GH_PAT expired or lacks permissions
**Fix:** Regenerate token, update secret in Settings → Secrets

### Consumer Not Receiving Updates
**Cause:** Not in consumer-config.yml or wrong repo URL
**Fix:** Check config syntax, verify repo URL

### PR Created But No Changes
**Cause:** Folder content unchanged since last sync
**Fix:** Close PR, not a bug (known behavior)

### Branch Accumulation (`from-*`)
**Cause:** No auto-cleanup (by design)
**Fix:** Manual cleanup periodically:
```bash
git branch -r | grep 'origin/from-' | cut -d'/' -f2- | xargs -I{} git push origin --delete {}
```

**Full troubleshooting:** See `ai_spec_v4/07_TROUBLESHOOTING.md` in demo folder

---

## Performance

**Typical downstream sync (3 consumers, 8 folders):**
- Folder splits: ~24 seconds
- Consumer processing: ~30 seconds
- **Total: ~60 seconds**

**Typical upstream sync:**
- Process consumer change: ~35 seconds
- PR creation: ~10 seconds
- **Total: ~45 seconds** (+ manual review time)

---

## Technical Specs

**Full documentation:** `ai_spec_v4/` folder in demo repository

Key files:
- `00_OVERVIEW_v4.md` - System overview
- `01_ARCHITECTURE_v4.md` - Detailed architecture
- `02_DOWNSTREAM_v4.md` - Downstream flow
- `03_UPSTREAM_v4.md` - Upstream flow
- `07_TROUBLESHOOTING.md` - Common issues

---

## Important Notes

### File Deletions Sync Automatically
Deleting a file in any shared folder propagates to all consumers (via `rm -rf` before injection).

### Consumer Folder Mappings Are Flexible
Same shared folder can map to different paths per consumer - this is the key feature enabling this architecture.

### History Preserved Here Only
Consumers receive snapshot only. For commit history and git blame, check this repo.

### Force-Push Strategy
`sync-auto` branch in consumers is force-pushed on every sync. Never commit manually to that branch.

---

**Questions? Check ai_spec_v4/00_OVERVIEW_v4.md for complete system documentation.**
