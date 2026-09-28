# 🛡️ Kodix Security Scanner GitHub Action

> **AI-consensus code security scanner.**
> Runs multiple Hyper Engine options on your code simultaneously.
> Only vulnerabilities confirmed by **2 or more models** are surfaced ~95% fewer false positives.

---

## Quick Start

```yaml
# .github/workflows/kodix.yml
name: Kodix Security Scan

on:
  push:
    branches: [main]
  # Allow manual trigger
  workflow_dispatch:

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: kodix-security/kodix-action@v1
        with:
          api_key: ${{ secrets.KODIX_API_KEY }}
```

That's it. Add `KODIX_API_KEY` to **Settings → Secrets → Actions** and every push triggers an AI security scan.

---

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `api_key` | **Yes** | — | Your Kodix API key. Always pass via `${{ secrets.KODIX_API_KEY }}` |
| `mode` | No | `full` | `full` scan every source file. `diff` scan only files changed in this push/PR |

No other configuration needed. The action:
- Scans **all readable text files** no extension filters, no file-size limits, no line-count caps
- **Never fails** the build due to vulnerability severity only fails on real errors (invalid key, insufficient tokens, network failure)
- **Polls until completion** no arbitrary timeout

---

## Outputs

| Output | Description | Example |
|--------|-------------|---------|
| `scan_id` | Unique Kodix scan identifier | `8f4c2a1e-9b3d-…` |
| `findings_count` | Total confirmed vulnerabilities | `4` |
| `critical_count` | CRITICAL severity count | `1` |
| `high_count` | HIGH severity count | `2` |
| `medium_count` | MEDIUM severity count | `1` |
| `low_count` | LOW severity count | `0` |
| `consensus_score` | Security score 0–100 | `62` |
| `pdf_url` | URL of the full PDF report | `https://storage…` |

Use outputs in subsequent steps:

```yaml
- name: Kodix Scan
  id: kodix
  uses: kodix-security/kodix-action@v1
  with:
    api_key: ${{ secrets.KODIX_API_KEY }}

- name: Print results
  run: |
    echo "Score    : ${{ steps.kodix.outputs.consensus_score }}/100"
    echo "Critical : ${{ steps.kodix.outputs.critical_count }}"
    echo "Report   : ${{ steps.kodix.outputs.pdf_url }}"
```

---

## Terminal Output

The action streams a live, structured log in three phases:

### Phase 1 File Discovery
```
════════════════════════════════════════════════════════════════════
  🛡️  KODIX AI SECURITY SCANNER
════════════════════════════════════════════════════════════════════
  Repository  : myorg/my-app
  Branch      : main
  Commit      : a3f8b2e1
  Triggered by: alice
  Scan mode   : FULL
  Started at  : 2025-09-14 10:42:01 UTC
════════════════════════════════════════════════════════════════════

  ┌────────────────────────────────────────────────────────────────
  │  COLLECTING FILES FULL MODE
  ├────────────────────────────────────────────────────────────────
  │  + src/auth.js                                           4.2 KB
  │  + src/api.js                                            2.8 KB
  │  + src/routes/users.js                                   5.7 KB
  │  ... (44 more files)
  ├────────────────────────────────────────────────────────────────
  │  Files collected : 47
  │  Total size      : 284.3 KB
  └────────────────────────────────────────────────────────────────
```

### Phase 2 Submission & Live Progress
```
  ┌────────────────────────────────────────────────────────────────
  │  SCAN IN PROGRESS
  ├────────────────────────────────────────────────────────────────
  │  3 AI models are scanning your code in parallel.
  │  Only findings confirmed by 2+ models will be reported.
  ├────────────────────────────────────────────────────────────────
  │  [10:42:14] scanning
  │  hyper-1: [████████████░░░░░░░░]  60%   hyper-2: [██████████████░░░░░░]  70%   hyper-3: [██████░░░░░░░░░░░░░░]  30%   22s
  │  hyper-1: [████████████████████] 100%   hyper-2: [████████████████████] 100%   hyper-3: [████████████████████] 100%   67s
  │  [10:43:08] scanning → completed
  ├────────────────────────────────────────────────────────────────
  │  ✓ All models finished in 67s
  └────────────────────────────────────────────────────────────────
```

### Phase 3 Results
```
════════════════════════════════════════════════════════════════════
  🛡️  KODIX SECURITY SCAN RESULTS
════════════════════════════════════════════════════════════════════
  Score     : 🟡 62/100  (Grade: C)
  Findings  : 4 confirmed vulnerability(ies)
────────────────────────────────────────────────────────────────────
  🔴 CRITICAL         1  █
  🟠 HIGH             2  ██
  🟡 MEDIUM           1  █
  🔵 LOW              0  —

  ┌─ [1/4] 🔴 CRITICAL
  │  File    : src/auth.js
  │  Line    : 47
  │  Title   : SQL Injection via string concatenation
  │  Details : User-controlled input concatenated directly into SQL.
  │  Models  : hyper-1, hyper-2, hyper-3
  │    const q = "SELECT * FROM users WHERE id=" + userId;
  └────────────────────────────────────────────────────────────────

  📄 Full PDF report  : https://storage.googleapis.com/…
  🌐 Dashboard        : https://kodixsecurity.com/scan/8f4c2a1e
════════════════════════════════════════════════════════════════════
```

---

## Scan Modes

| Mode | Scans | Best for |
|------|-------|----------|
| `full` (default) | Every readable file in the repo | `main` branch, weekly audits, initial assessment |
| `diff` | Only files changed in this push/PR | Feature branches, PR validation, high-volume repos |

```yaml
- uses: kodix-security/kodix-action@v1
  with:
    api_key: ${{ secrets.KODIX_API_KEY }}
    mode: diff
```

---

## Error Handling

The action exits with a **non-zero code** (failing the build) only for operational errors:

| Cause | Exit |
|-------|------|
| `api_key` not provided | `1` |
| Invalid or revoked API key (HTTP 401) | `1` |
| Insufficient token balance (HTTP 402) | `1` |
| Rate limit exceeded (HTTP 429) | `1` |
| Network error cannot reach Kodix API | `1` |
| Scan marked `failed` by the API | `1` |

Vulnerability findings regardless of severity **never** cause the action to fail.

---

## Prerequisites

1. A Kodix account at [kodixsecurity.com](https://kodixsecurity.com)
2. Tokens purchased (Profile → Tokens)
3. An API key generated (Profile → API Keys)
4. The key stored as a GitHub Secret named `KODIX_API_KEY`

---

## Examples

| File | Description |
|------|-------------|
| [`examples/basic.yml`](examples/basic.yml) | Minimal push to main |
| [`examples/diff-only.yml`](examples/diff-only.yml) | Diff mode for feature branches |

---

## How It Works

```
Push code
   ↓
Action collects all readable files (FULL or DIFF mode)
   ↓
Files sent to Kodix API with your api_key
   ↓
3 Hyper Engine options scan in parallel (Hyper 1 + Hyper 2 + Hyper 3)
   ↓
Consensus engine: only findings confirmed by 2+ models surface
   ↓
Live progress bars → results table in terminal
   ↓
Outputs written (scan_id, score, counts, pdf_url)
```

---

## Security

- Your API key is passed via GitHub Secrets never hardcoded
- Source files are held in memory only during the scan and immediately discarded
- Only vulnerability metadata is persisted to your Kodix scan history
- Revoke any compromised key instantly at kodixsecurity.com/profile

---

## Support

- Docs: [kodixsecurity.com/docs](https://kodixsecurity.com/docs)
- Email: gabi@kodixsecurity.com

---

## 🛠️ Maintainer Guide: Releasing Updates

When you make changes to this Action's code (such as modifying [action.yml](file:///d:/Gabi/kodix-action/action.yml)) and want to publish them so that users referencing `@v1` automatically get the latest updates, run these commands in your terminal:

### 1. Push your changes to the main branch
Stage, commit, and push your changes to your remote branch first:
```bash
git add .
git commit -m "your description of changes"
git push origin main
```

### 2. Move and force-push the `v1` tag
Move the floating `v1` tag to point to your new commit, and force-push it to GitHub:
```bash
# Update the local tag to point to the latest commit
git tag -fa v1 -m "v1 - latest stable release"

# Force-push the updated tag to GitHub
git push origin v1 --force
```

---

### (Recommended) Creating Specific Patch Releases
For better traceability, it is also recommended to create a specific semantic version tag (like `v1.0.2`, `v1.0.3`, etc.) alongside `v1`:
```bash
# Tag the specific patch version
git tag v1.0.2

# Push the patch tag to GitHub
git push origin v1.0.2
```

> [!NOTE]
> Once you force-push the updated `v1` tag (Step 2), any user using `uses: kodix-security/kodix-action@v1` in their workflow will automatically start using your updated code on their next scan.

