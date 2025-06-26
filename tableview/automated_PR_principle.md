## ✅ Automated PR Creation – Final Rules and Working Pattern (All Contexts)

### 🧠 Core Principle:
> **Creating a pull request is an API operation. It does NOT require checking out code locally.**

### ✅ Always Follow These Rules:

- ❌ Do NOT use `actions/checkout` unless you're modifying files before the PR.
- ❌ Do NOT use `gh pr create` unless you want to manage fine-grained PATs per repo.
- ❌ Do NOT use `peter-evans/create-pull-request` if your source branch content is managed elsewhere (e.g., subtree injection).
- ✅ Use GitHub REST API (`POST /repos/:owner/:repo/pulls`) for all automated PRs.
- ✅ `GITHUB_TOKEN` works, but only if:
  - You enable "Read and write permissions" in repo settings
  - You enable "Allow GitHub Actions to create and approve pull requests"

---

### ✅ Minimal, Correct GitHub Action Job for PR Creation:

No checkout, no CLI, no plugin bloat:

- name: Create Pull Request via REST API  
  run: |
    curl -X POST \
      -H "Authorization: Bearer $GITHUB_TOKEN" \
      -H "Accept: application/vnd.github+json" \
      https://api.github.com/repos/${{ github.repository }}/pulls \
      -d @- <<EOF
    {
      "title": "[auto-sync] Update from shared-resources",
      "head": "sync-auto",
      "base": "main",
      "body": "Automated update via CI. Please review and merge."
    }
EOF  
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

---

### 🧱 Use This For:
- Downstream sync PRs
- Upstream back-prop PRs
- Auto-PRs from patch workflows
- Any CI flow where the branch is already prepared

---

### ✅ Outcome:
- No branch overwrite
- No race conditions
- No CLI requirements
- ✅ Consistent, secure PR behavior across all repos
