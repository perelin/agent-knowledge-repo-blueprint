---
type: Runbook
title: Agent Knowledge Repo — Bootstrap Guide
description: Self-contained, shareable instructions for an AI agent to set up a personal "agents" knowledge repo — an Open Knowledge Format bundle with domains, operative AGENTS.md files, generated indexes, and a secrets-restore pattern.
resource: https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md
tags: [okf, bootstrap, meta, shareable]
timestamp: 2026-10-05T00:00:00Z
status: active
---

# Agent Knowledge Repo — Bootstrap Guide

> **To the agent reading this:** this document tells you how to build a
> knowledge-base repository that you (and other agents) will use to operate the
> user's systems — servers, self-hosted services, apps, home devices, SaaS APIs.
> Follow the steps in order. Interview the user for the actual content; everything
> named `example`, `photo-library`, `nas` etc. below is a placeholder.

## 1. What you are building

A git repository of plain markdown that any agent runtime can read, with no
proprietary tooling. It follows the
[Open Knowledge Format (OKF v0.1)](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md):
a tree of markdown files with YAML frontmatter.

Three layers coexist. Keep them separate:

| Layer | Files | Purpose | Loaded when |
|-------|-------|---------|-------------|
| **Operative** — *how to act* | `AGENTS.md` (canonical) + `CLAUDE.md` (symlink to it) | Danger zones, access rules, common one-liners, pointers into concepts | Automatically, by the agent runtime |
| **Knowledge** — *what is true* | `*.md` **concepts**, each with frontmatter | Facts about one system/topic each | On demand |
| **Navigation** | `index.md` (reserved, generated) | Directory listing for progressive disclosure | On demand |

The unit of organization is the **domain**: one directory per system
(`applications/photo-library/`, `infrastructure/dns/`, …), grouped into a few
risk-ordered **categories**.

Why this shape works:

- The operative layer stays short, so it is cheap to auto-load and the danger
  zones are always in context.
- Knowledge is split into small typed concepts, so an agent reads only what the
  task needs (root `AGENTS.md` → domain `AGENTS.md` → domain `index.md` → one concept).
- Frontmatter (`type`, `resource`, `status`, `tags`) makes the tree queryable
  without parsing prose.
- Superseded knowledge is marked `status: legacy`, not deleted, so history is
  preserved but clearly separated from what is live.

## 2. Bootstrap sequence

Do these in order. Ask the user before step 8 which systems to cover first.

1. `git init` a new repo (suggested name: `agents`).
2. Create the category directories. Default set (adapt to the user):
   ```
   infrastructure/   # foundational: hypervisors, bare metal, DNS, VPN — high risk, rare changes
   services/         # platform components: reverse proxy, container host, app platform — medium risk
   applications/     # end-user apps with APIs — lower risk, frequent calls
   home/             # home-network devices (separate location/network)
   external/         # third-party SaaS APIs, nothing self-hosted
   runbooks/         # cross-domain step-by-step workflows
   docs/okf/         # the format contract + tooling
   ```
3. Write `docs/okf/conventions.md` from [section 3](#3-conventions-the-format-contract)
   (it must carry frontmatter itself, `type: Reference`).
4. Write `docs/okf/validate.sh` and `docs/okf/generate-index.sh` verbatim from
   [section 7](#7-tooling-embed-verbatim); `chmod +x` both.
5. Write `.okfignore` and `.gitignore` from [section 7](#7-tooling-embed-verbatim).
6. Write the root `AGENTS.md` from the [template](#41-root-agentsmd), then
   `ln -s AGENTS.md CLAUDE.md`. Always edit `AGENTS.md`, never the symlink.
7. Write the root `index.md` (hand-maintained, first line `<!-- okf:manual -->`)
   and `glossary.md` (`type: Glossary`) — templates in [section 4](#4-templates).
8. Create the first domain using the [worked example](#5-worked-example-domain)
   as the shape. Store its credentials per [section 6](#6-secrets-and-multi-machine-sync).
9. Run:
   ```bash
   docs/okf/generate-index.sh      # writes every non-manual index.md
   docs/okf/validate.sh            # must print PASS
   ```
10. Add the domain's row to the root `AGENTS.md` domain table. Commit.

Repeat steps 8–10 per system. A domain is "done" when an agent with no prior
context can find access, danger zones, and the API spec from `AGENTS.md` alone.

## 3. Conventions (the format contract)

Port this section into `docs/okf/conventions.md`.

### 3.1 Frontmatter

Every non-reserved `.md` MUST start with a YAML frontmatter block containing a
non-empty `type`.

```yaml
---
type: Application            # REQUIRED — see taxonomy
title: Photo Library         # recommended — display name (used by generate-index)
description: One sentence.   # recommended — shown in the generated index.md
resource: https://photos.example.com   # recommended — URI of the underlying asset
tags: [photos, self-hosted]  # recommended
timestamp: 2026-01-01T00:00:00Z        # recommended — ISO 8601 last-modified
status: active               # extension — active | legacy | deprecated | planned
access: API key (X-API-Key)  # extension — how the asset is reached (never the secret itself)
---
```

`status` and `access` are custom extension keys (OKF allows them; consumers keep
unknown keys).

### 3.2 Type taxonomy

Types are not centrally registered. Use a small consistent set:

| `type` | For |
|--------|-----|
| `Infrastructure` | Hypervisors, bare metal, DNS, VPN (`infrastructure/`) |
| `Service` | Reverse proxy, container host, app platform (`services/`) |
| `Application` | End-user apps (`applications/`) |
| `Home Device` | Home-network devices (`home/`) |
| `External API` | Third-party SaaS (`external/`) |
| `Toolkit` | A script/automation collection the user maintains; the scripts are assets, the concept describes them |
| `Runbook` | Ordered steps to execute |
| `Operational Decision` | A recorded decision, its rationale and its reversal path |
| `Reference` | Pointers to external tools, repos, resources |
| `Glossary` | Terminology |

Domain-specific types are fine when they add meaning (e.g. `Reading Order`).
Consumers must tolerate unknown types.

### 3.3 Concepts and IDs

One file = one concept. **Concept ID** = repo-relative path minus `.md`
(`applications/photo-library/photo-library.md` → `applications/photo-library/photo-library`).
Each domain has a **primary concept** named after the directory
(`<domain>/<domain>.md`) carrying the domain's main `type`.

### 3.4 Split and disambiguation rules

- **Decision vs Runbook:** if the load-bearing content is the *why* (rationale +
  reversal path) → `Operational Decision`; if it is the *how* (ordered steps) →
  `Runbook`. If both, primary intent wins; cross-link the counterpart.
- **Split granularity (medium):** create a separate concept only when a topic
  (a) would be queried on its own **and** (b) has about a screen of content or more,
  **or** carries its own `type`/`status`. Otherwise fold it into the primary
  concept. Don't shatter one cohesive document into micro-files.
- **No READMEs inside domains.** Navigation is the generated `index.md`; content
  lives in the primary concept plus topic concepts.
- **`docs/` subfolder:** longer standalone material (runbooks, investigations,
  troubleshooting) may live in `<domain>/docs/`; it still needs frontmatter.
- **Point-in-time figures** (counts, versions) are written "as of `<date>`" and
  backed by `timestamp`. They are not `legacy`; they are dated.
- **Legacy:** mark superseded content `status: legacy` and keep it. First lift
  any still-true facts into an active concept.
- **AGENTS.md is operative only:** danger zones, access, common one-liners,
  pointers. Knowledge belongs in concepts, never duplicated into `AGENTS.md`.

### 3.5 Reserved files

- **`index.md`** — directory listing, **no frontmatter**. Generated by
  `docs/okf/generate-index.sh`; do not hand-edit. Exception: a file whose first
  line is `<!-- okf:manual -->` is hand-maintained and skipped by the generator
  (use this for the root `index.md`).
- **`log.md`** — optional per-domain change history, **no frontmatter**, newest
  first, `## YYYY-MM-DD — <what>` headings. Record what git does not show:
  the starting question, findings, what was changed, and corrections of earlier
  wrong assumptions.
- **`AGENTS.md` / `CLAUDE.md`** — operative layer, exempt from concept validation.

### 3.6 Linking

Prefer bundle-relative absolute links (`/applications/photo-library/photo-library.md`);
relative (`./other.md`) is allowed. Avoid broken links.

### 3.7 Conventional body sections

`# Schema` (structured fields), `# Examples` (runnable snippets), `# Citations`
(numbered external sources).

### 3.8 Conformance

Conformant = every non-reserved `.md` has parseable frontmatter with a non-empty
`type`, and `index.md` / `log.md` have none. Check with `docs/okf/validate.sh`.

## 4. Templates

### 4.1 Root `AGENTS.md`

```markdown
# Agents Knowledge Base

This repo is an Open Knowledge Format bundle (see docs/okf/conventions.md).
Knowledge lives in per-concept markdown files with frontmatter; navigation is via
generated index.md files; this AGENTS.md is the operative layer.

## How to use this repo

1. Identify the domain the user is asking about (tables below).
2. Read `<domain>/AGENTS.md` first if it exists — danger zones and common operations.
3. Read the domain's concepts — start at `<domain>/index.md`, then `<domain>/<domain>.md`.
4. Load credentials with `source <domain>/.env` before making API calls.
5. Consult the domain's API spec (`openapi.json` / `api-docs.yaml`) if present.
   Do NOT guess endpoints.
6. Check `<domain>/docs/` and `runbooks/` for procedures.

## Domains

### Infrastructure
| Domain | System | Access Method |
|--------|--------|---------------|
| `infrastructure/dns/` | DNS provider | CLI (`<profile>`) |

### Applications
| Domain | System | Has API Spec | Access Method |
|--------|--------|--------------|---------------|
| `applications/photo-library/` | Photo library | Yes (OpenAPI) | REST via curl (API key) |

<!-- one table per category; one row per domain; keep rows to one line -->

## Danger zones (global)

- <the one host whose restart takes everything down, and the safe alternative>
- <destructive commands that must never run without explicit confirmation>
- <systems that manage their own state (e.g. DNS) and must not be edited by hand>

## Secret management

- All credentials live in <secret store>. `.env` files are gitignored and ephemeral.
- Restore all `.env` files: `./restore-env.sh`
- NEVER create or modify a `.env` without also storing the secret in the store
  AND adding its block to `restore-env.sh`.

## Conventions

- When modifying infrastructure, update the domain's docs BEFORE finishing.
- Never hardcode secrets in scripts; always `source <domain>/.env`.
```

### 4.2 Root `index.md`

```markdown
<!-- okf:manual -->
# Agents Knowledge Base

An Open Knowledge Format bundle describing <the user's> systems.

## Start here
1. Going to act on something? Read AGENTS.md first.
2. New to the vocabulary? See glossary.md.
3. Want to understand the format? See docs/okf/conventions.md.

## Categories
- [infrastructure/](/infrastructure/index.md) — foundational systems. High risk.
- [services/](/services/index.md) — platform components. Medium risk.
- [applications/](/applications/index.md) — end-user apps. Lower risk.
- [home/](/home/index.md) — home-network devices.
- [external/](/external/index.md) — third-party SaaS APIs.
- [runbooks/](/runbooks/index.md) — cross-domain workflows.
```

### 4.3 `glossary.md`

Tables of terms an agent would otherwise misread: host aliases, container/VM
IDs, the public domain, internal shorthand, secret-store paths. One line per
term, each linking to the concept that owns it. `type: Glossary`.

### 4.4 Domain `AGENTS.md`

```markdown
# <System> — agent notes

<One line: what it is and where it runs.>

## Access at a glance
- **URL:** `https://<system>.example.com`
- **Auth:** <header name + scheme; note look-alikes, e.g. "Token, not Bearer">
- **Credentials:** `source <category>/<system>/.env` → `<SYSTEM>_URL`, `<SYSTEM>_TOKEN`
- **Scope of the credential:** <what it can see/change — treat admin tokens as full access>

## Danger zones
- <operations that are slow, irreversible, or have side effects on other systems>
- <known traps that look like success but are not>

## Common operations
- <one-liners; anything longer becomes a Runbook concept>

## Where the knowledge lives
- [<system>.md](./<system>.md) — overview, API, configuration
- [docs/](./docs/index.md) — runbooks and investigations
```

Then `ln -s AGENTS.md CLAUDE.md` inside the domain too.

### 4.5 Primary concept `<domain>/<domain>.md`

```markdown
---
type: Application
title: <System>
description: <one sentence — appears in the generated index>
resource: https://<system>.example.com
tags: [<topic>, self-hosted]
timestamp: <ISO 8601>
status: active
access: <auth scheme, never the secret>
---

# <System>

## Quick Reference
| Property | Value |
|----------|-------|
| **URL** | `https://<system>.example.com` |
| **Access** | <auth scheme> |
| **API Spec** | `openapi.json` (version, as of <date>) |
| **Runs on** | <host / platform> |

## Credentials
    source <category>/<system>/.env

# Examples
    curl -s -H "X-API-Key: $SYSTEM_API_KEY" "$SYSTEM_URL/api/v1/..."
```

## 5. Worked example domain

```
applications/photo-library/
├── AGENTS.md              # operative: access, danger zones (template 4.4)
├── CLAUDE.md -> AGENTS.md
├── index.md               # GENERATED — never hand-edit
├── log.md                 # newest-first history, no frontmatter
├── photo-library.md       # primary concept, type: Application
├── album-sync.md          # topic concept, type: Toolkit (has its own status/scripts)
├── storage-migration.md   # type: Operational Decision (why + how to revert)
├── openapi.json           # vendored API spec — the endpoint ground truth
├── .env                   # gitignored, produced by restore-env.sh
├── scripts/               # the Toolkit's code
└── docs/
    └── restore-from-backup.md   # type: Runbook
```

What `generate-index.sh` then writes into `applications/photo-library/index.md`:

```markdown
# Photo Library

## Concepts

- [album-sync.md](./album-sync.md) — Nightly script that mirrors phone albums into the library.
- [photo-library.md](./photo-library.md) — Self-hosted photo library — browse, tag and share via REST API.
- [storage-migration.md](./storage-migration.md) — Why originals moved to network storage, and how to move them back.

## Subdirectories

- [docs/](./docs/index.md)
- [scripts/](./scripts/index.md)
```

## 6. Secrets and multi-machine sync

The pattern, independent of the store used (the original uses
[`pass`](https://www.passwordstore.org/) — GPG-encrypted, git-backed; any CLI-readable
store works):

- **The secret store is the single source of truth.** Entry naming:
  `servers/<host>/<secret>` for host secrets, `services/<name>/<secret>` for API keys.
- **`.env` files are derived artifacts.** One per domain, gitignored, regenerated
  by `restore-env.sh`. Agents `source` them; scripts never contain secrets.
- **A missing store entry is a hard failure.** The restore script must count it,
  name it, and exit non-zero — and also flag any empty `KEY=` it wrote (a missing
  *field* inside an existing entry). Otherwise you get an `.env` with an empty
  key while the run reports success.
- **Rule for agents:** never create or edit a `.env` without (1) storing the
  secret in the store and (2) adding its block to `restore-env.sh`.
- Commit an `.env.example` per domain if the variable names are not obvious.

Minimal `restore-env.sh` skeleton:

```bash
#!/usr/bin/env bash
# Restore all .env files from the secret store.
set -euo pipefail
REPO_DIR="$(cd "$(dirname "$0")" && pwd)"
FAILED=0; RESTORED=0
MISSING_LOG="$(mktemp "${TMPDIR:-/tmp}/restore-env-missing.XXXXXX")"
trap 'rm -f "$MISSING_LOG"' EXIT

# Never fail inside $(…) — a heredoc would swallow the error and write an empty value.
secret() { pass show "$1" 2>/dev/null || printf '%s\n' "$1" >> "$MISSING_LOG"; }

check() {   # call after writing each file
  local f="$REPO_DIR/$1"
  if [ -s "$MISSING_LOG" ]; then
    echo "  ✗ $1 — not in store: $(sort -u "$MISSING_LOG" | tr '\n' ' ')" >&2
    : > "$MISSING_LOG"; FAILED=$((FAILED + 1)); return 0
  fi
  if grep -qE '^[A-Za-z_][A-Za-z0-9_]*=("")?$' "$f"; then
    echo "  ✗ $1 — empty value(s)" >&2; FAILED=$((FAILED + 1)); return 0
  fi
  echo "  → $1"; RESTORED=$((RESTORED + 1))
}

cat > "$REPO_DIR/applications/photo-library/.env" <<EOF
PHOTO_LIBRARY_URL=https://photos.example.com
PHOTO_LIBRARY_API_KEY=$(secret services/photo-library/api-key)
EOF
check "applications/photo-library/.env"

echo "restored $RESTORED, failed $FAILED"
[ "$FAILED" -eq 0 ]
```

**Multiple machines (optional):** keep the repo and the secret store as two git
remotes. A `sync.sh` pulls both fast-forward-only and then runs `restore-env.sh`;
it refuses to run on a dirty tree and never pushes. Pushing stays a deliberate
step. A scheduled variant may pull unattended but should not decrypt secrets.
Back up the key that decrypts the store — it is the single point of failure.

## 7. Tooling (embed verbatim)

### 7.1 `.okfignore`

```gitignore
# Paths excluded from the OKF bundle (glob patterns, repo-relative).
.git/
.claude/
.worktrees/
*/.pytest_cache/
*/.venv/
node_modules/
*/node_modules/
```

Add any vendored upstream docs, generated reports, or synced foreign folders.

### 7.2 `.gitignore` (minimum)

```gitignore
.env
!.env.example
.DS_Store
__pycache__/
.venv/
.claude/settings.local.json
.claude/worktrees/
.worktrees/
```

Never commit machine-local agent settings: a tracked file that each machine
edits leaves every tree permanently dirty and blocks fast-forward syncs.

### 7.3 `docs/okf/validate.sh`

```bash
#!/usr/bin/env bash
# OKF conformance validator for this repo.
#   - non-reserved *.md must start with YAML frontmatter containing a non-empty `type`
#   - reserved index.md / log.md must NOT have frontmatter
# Honors .okfignore (glob patterns, repo-relative). Exit 1 on any violation.
# Usage: docs/okf/validate.sh [scope-subdir]   (default = whole bundle)
set -euo pipefail
ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/../.." && pwd)"
cd "$ROOT"
SCOPE="${1:-}"

python3 - "$ROOT" "$SCOPE" <<'PY'
import os, sys, fnmatch, re

root = sys.argv[1]
scope = sys.argv[2] if len(sys.argv) > 2 else ""
walk_root = os.path.join(root, scope) if scope else root
ignore = []
igf = os.path.join(root, ".okfignore")
if os.path.exists(igf):
    for line in open(igf):
        line = line.strip()
        if line and not line.startswith("#"):
            ignore.append(line.rstrip("/"))

def ignored(rel):
    for pat in ignore:
        if fnmatch.fnmatch(rel, pat) or fnmatch.fnmatch(rel, pat + "/*") or rel.startswith(pat + "/"):
            return True
    return False

def has_frontmatter(text):
    return text.startswith("---\n") or text.startswith("---\r\n")

def frontmatter_type(text):
    lines = text.splitlines()
    if not lines or lines[0].strip() != "---":
        return None
    for i in range(1, len(lines)):
        if lines[i].strip() == "---":
            block = lines[1:i]
            break
    else:
        return None
    for l in block:
        m = re.match(r"^type:\s*(.+?)\s*$", l)
        if m and m.group(1).strip().strip('"\''):
            return m.group(1).strip()
    return None

errors, checked = [], 0
for dirpath, dirnames, filenames in os.walk(walk_root):
    dirnames[:] = [d for d in dirnames if not ignored(os.path.relpath(os.path.join(dirpath, d), root))]
    for fn in filenames:
        if not fn.endswith(".md"):
            continue
        full = os.path.join(dirpath, fn)
        rel = os.path.relpath(full, root)
        if ignored(rel):
            continue
        # operative layer, not OKF concepts — exempt entirely
        if fn in ("AGENTS.md", "CLAUDE.md"):
            continue
        checked += 1
        text = open(full, encoding="utf-8").read()
        if fn in ("index.md", "log.md"):
            if has_frontmatter(text):
                errors.append(f"{rel}: reserved file must NOT have frontmatter")
        else:
            if not has_frontmatter(text):
                errors.append(f"{rel}: missing YAML frontmatter")
            elif not frontmatter_type(text):
                errors.append(f"{rel}: frontmatter has no non-empty `type`")

print(f"OKF validate: checked {checked} markdown files")
if errors:
    print(f"FAIL — {len(errors)} violation(s):")
    for e in errors:
        print("  - " + e)
    sys.exit(1)
print("PASS — all files conformant")
PY
```

### 7.4 `docs/okf/generate-index.sh`

```bash
#!/usr/bin/env bash
# Regenerate index.md files across the OKF bundle from concept frontmatter.
#   - lists non-reserved concepts (title + description) and subdirectories
#   - skips any index.md whose first line is "<!-- okf:manual -->"
#   - honors .okfignore; never writes frontmatter into index.md
# Usage: docs/okf/generate-index.sh [--check] [scope-subdir]
#   --check       fail if any index.md is stale instead of writing
#   scope-subdir  limit generation to this subtree (repo-relative); default = whole bundle
set -euo pipefail
ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/../.." && pwd)"
cd "$ROOT"
MODE="write"; SCOPE=""
for arg in "$@"; do
  if [ "$arg" = "--check" ]; then MODE="--check"; else SCOPE="$arg"; fi
done

python3 - "$ROOT" "$MODE" "$SCOPE" <<'PY'
import os, sys, fnmatch, re

root, mode = sys.argv[1], sys.argv[2]
scope = sys.argv[3] if len(sys.argv) > 3 else ""
walk_root = os.path.join(root, scope) if scope else root
ignore = []
igf = os.path.join(root, ".okfignore")
if os.path.exists(igf):
    for line in open(igf):
        line = line.strip()
        if line and not line.startswith("#"):
            ignore.append(line.rstrip("/"))

def ignored(rel):
    rel = rel.replace(os.sep, "/")
    for pat in ignore:
        if fnmatch.fnmatch(rel, pat) or rel == pat or rel.startswith(pat + "/"):
            return True
    return False

def fm(text):
    """Return dict of top-level scalar frontmatter fields (title/description/type)."""
    out = {}
    lines = text.splitlines()
    if not lines or lines[0].strip() != "---":
        return out
    for i in range(1, len(lines)):
        if lines[i].strip() == "---":
            for l in lines[1:i]:
                m = re.match(r"^(\w+):\s*(.*)$", l)
                if m:
                    out[m.group(1)] = m.group(2).strip().strip('"\'')
            break
    return out

def title_case(name):
    return name.replace("-", " ").replace("_", " ").title()

stale = []

def concept_dirs():
    for dirpath, dirnames, filenames in os.walk(walk_root):
        rel = os.path.relpath(dirpath, root)
        if rel != "." and ignored(rel):
            dirnames[:] = []
            continue
        dirnames[:] = [d for d in dirnames if not ignored(os.path.join(rel, d) if rel != "." else d)]
        yield dirpath, rel, sorted(dirnames), sorted(filenames)

for dirpath, rel, dirnames, filenames in concept_dirs():
    concepts = [f for f in filenames if f.endswith(".md")
                and f not in ("index.md", "log.md", "AGENTS.md", "CLAUDE.md")]
    def has_md(sub):
        for _, _, fns in os.walk(sub):
            if any(f.endswith(".md") for f in fns):
                return True
        return False
    subdirs = [d for d in dirnames if has_md(os.path.join(dirpath, d))]
    if not concepts and not subdirs:
        continue
    index_path = os.path.join(dirpath, "index.md")
    if os.path.exists(index_path):
        first = open(index_path, encoding="utf-8").readline().strip()
        if first == "<!-- okf:manual -->":
            continue

    dirname = os.path.basename(dirpath) if rel != "." else "Knowledge Base"
    heading = title_case(dirname)
    primary = os.path.join(dirpath, dirname + ".md")
    if os.path.exists(primary):
        heading = fm(open(primary, encoding="utf-8").read()).get("title", heading)

    out = [f"# {heading}", ""]
    if concepts:
        out.append("## Concepts")
        out.append("")
        for c in concepts:
            meta = fm(open(os.path.join(dirpath, c), encoding="utf-8").read())
            desc = meta.get("description", "")
            out.append(f"- [{c}](./{c})" + (f" — {desc}" if desc else ""))
        out.append("")
    if subdirs:
        out.append("## Subdirectories")
        out.append("")
        for d in subdirs:
            out.append(f"- [{d}/](./{d}/index.md)")
        out.append("")
    content = "\n".join(out).rstrip() + "\n"

    existing = open(index_path, encoding="utf-8").read() if os.path.exists(index_path) else ""
    if existing != content:
        stale.append(os.path.relpath(index_path, root))
        if mode != "--check":
            open(index_path, "w", encoding="utf-8").write(content)

if mode == "--check":
    if stale:
        print(f"STALE — {len(stale)} index.md file(s) out of date:")
        for s in stale:
            print("  - " + s)
        sys.exit(1)
    print("index.md files up to date")
else:
    print(f"generated/updated {len(stale)} index.md file(s)")
    for s in stale:
        print("  - " + s)
PY
```

Optionally wire both into a pre-commit hook (`validate.sh && generate-index.sh --check`).

## 8. Operating rules to put in the agent's instructions

These are what keep the repo trustworthy over time. Put them in the root `AGENTS.md`
(or the agent's global instructions):

- **Docs are part of the change.** Modifying a system without updating its domain
  concepts (and `log.md` for non-obvious decisions) is unfinished work.
- **Endpoints come from the vendored spec**, never from memory.
- **Measure state, don't infer it.** "It's in the folder" → `ls`; "it's in the DB" →
  query. Logs and history prove *events*, not current state. Before writing "X does
  not exist" into a concept, look at the unfiltered raw object once. Label anything
  unverified as unverified.
- **Never delete superseded knowledge** — mark it `status: legacy` after lifting
  live facts out.
- **Danger zones are written once the hard way.** After any incident or near-miss,
  add a one-line danger zone to the domain `AGENTS.md` that names the trap and the
  safe alternative.
- **Keep secrets out of git, out of concepts, out of logs.** The `access:` field and
  `AGENTS.md` name the *scheme* and the *store path*, never the value.
- **Before sharing the repo with anyone:** it is personal by design (hosts, IPs,
  account names, decisions). Share this guide, not the repo.
