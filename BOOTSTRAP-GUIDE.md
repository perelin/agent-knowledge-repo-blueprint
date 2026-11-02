---
type: Runbook
title: Agent Knowledge Repo — Bootstrap Guide
description: Self-contained, shareable instructions for an AI agent to set up a personal "agents" knowledge repo — an Open Knowledge Format bundle with domains, operative AGENTS.md files, generated indexes, and a secrets-restore pattern.
resource: https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md
tags: [okf, bootstrap, meta, shareable]
generated: { by: claude-code/opus-5.5, at: 2026-11-02T14:09:00Z }
status: stable
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
[Open Knowledge Format (OKF v0.2)](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md):
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
- Frontmatter (`type`, `resource`, `status`, `tags`, `generated`) makes the tree
  queryable without parsing prose, including who wrote a concept and when.
- Superseded knowledge is marked `status: deprecated`, not deleted, so history is
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
7. Write the root `index.md` (hand-maintained: `okf_version` frontmatter, then the
   `<!-- okf:manual -->` marker)
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

Every non-reserved `.md` MUST start with a YAML frontmatter block that parses as
YAML and contains a non-empty `type`. Quote values containing `: ` or starting with
a backtick (`access: "Bearer token (header Authorization: Bearer)"`).

```yaml
---
type: Application            # REQUIRED — see taxonomy
title: Photo Library         # recommended — display name (used by generate-index)
description: One sentence.   # recommended — shown in the generated index.md
resource: https://photos.example.com   # recommended — URI of the underlying asset
tags: [photos, self-hosted]  # recommended
generated: { by: claude-code/opus-5.5, at: 2026-01-01T12:00:00Z }  # who last wrote it, when
status: stable               # OKF — draft | stable | deprecated (absent = stable)
access: "API key (header X-API-Key)"  # extension — how the asset is reached (never the secret)
---
```

`access` and `subject_status` are custom extension keys (OKF allows them; consumers
keep unknown keys).

**Provenance and trust (OKF v0.2).**

- `generated: { by, at }` — set on every meaningful change. `by` is an *actor*:
  `<harness>/<model>` for an agent (`claude-code/opus-5.5`), `agent/unknown` if not
  recorded, `process:<id>` for a script that writes the file, `human:<id>` for the
  owner. `at` is an ISO 8601 datetime **with offset**.
- `verified: { by, at }` (or a list of them) — only for a real confirmation against
  the source or live system, never as a side effect of writing. A `human:` verifier
  makes the concept *human-reviewed*; never use `human:` for agent-written content.
- `status` — the concept's own lifecycle: `stable` (default), `draft`, `deprecated`.
  The state of the *thing* described (a retired server, a published post) goes into
  the extension key `subject_status`.
- `stale_after: <datetime>` — only where a real expiry exists (review cadence,
  revisit date).
- `sources:` — list of `{ id, resource, title }` the content comes from; attribute a
  single claim with a footnote keyed by the id (`…[^api-docs]`).

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
  backed by `generated.at`. They are not `deprecated`; they are dated.
- **Deprecation:** mark superseded content `status: deprecated` and keep it. First
  lift any still-true facts into a current concept.
- **AGENTS.md is operative only:** danger zones, access, common one-liners,
  pointers. Knowledge belongs in concepts, never duplicated into `AGENTS.md`.

### 3.5 Reserved files

- **`index.md`** — directory listing, **no frontmatter**. Generated by
  `docs/okf/generate-index.sh`; do not hand-edit. Exception: a file whose first
  line (after optional frontmatter) is `<!-- okf:manual -->` is hand-maintained and
  skipped by the generator (use this for the root `index.md`). The root `index.md`
  is also the one index allowed frontmatter: `okf_version: "0.2"`.
- **`log.md`** — optional per-domain change history, **no frontmatter**, newest
  first, `## YYYY-MM-DD — <what>` headings. Record what git does not show:
  the starting question, findings, what was changed, and corrections of earlier
  wrong assumptions.
- **`AGENTS.md` / `CLAUDE.md`** — operative layer, exempt from concept validation.

### 3.6 Linking

Prefer bundle-relative absolute links (`/applications/photo-library/photo-library.md`);
relative (`./other.md`) is allowed. Avoid broken links.

### 3.7 Conventional body sections

`# Schema` (structured fields), `# Examples` (runnable snippets). Sources go into
the `sources` frontmatter, not into a body `# Citations` list (superseded in v0.2).

### 3.8 Conformance

Conformant = every non-reserved `.md` has parseable YAML frontmatter with a non-empty
`type`, and `index.md` / `log.md` have none (except `okf_version` on the root
`index.md`). Check with `docs/okf/validate.sh`; with PyYAML installed it also enforces
the v0.2 field rules (no `timestamp`, `status` enum, `generated.by`, datetimes with
offset, `resource` on every `sources` entry).

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
---
okf_version: "0.2"
---
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
generated: { by: <actor>, at: <ISO 8601 datetime with offset> }
status: stable
access: "<auth scheme, never the secret>"
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
#   - non-reserved *.md must start with *parseable* YAML frontmatter containing a non-empty `type`
#     (full YAML parse needs PyYAML; without it, falls back to a line-regex check and warns)
#   - OKF v0.2 field rules (with PyYAML): no legacy `timestamp`; `status` in
#     draft|stable|deprecated; `generated.by` present; `generated.at`, `verified[].at`,
#     `stale_after` are datetimes with a UTC offset; every `sources` entry has a `resource`
#   - reserved index.md / log.md must NOT have frontmatter — except the bundle-root
#     index.md, which may carry `okf_version` only
# Honors .okfignore (glob patterns, repo-relative). Exit 1 on any violation.
# Usage: docs/okf/validate.sh [scope-subdir]   (default = whole bundle)
set -euo pipefail
ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/../.." && pwd)"
cd "$ROOT"
SCOPE="${1:-}"

python3 - "$ROOT" "$SCOPE" <<'PY'
import os, sys, fnmatch, re, datetime
try:
    import yaml
except ImportError:
    yaml = None

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

def frontmatter_block(text):
    lines = text.splitlines()
    if not lines or lines[0].strip() != "---":
        return None
    for i in range(1, len(lines)):
        if lines[i].strip() == "---":
            return "\n".join(lines[1:i])
    return None

def yaml_check(text):
    """Return an error string, or None. Requires PyYAML."""
    block = frontmatter_block(text)
    if block is None:
        return "frontmatter block is not closed with ---"
    try:
        data = yaml.safe_load(block)
    except yaml.YAMLError as e:
        mark = getattr(e, "problem_mark", None)
        where = f" (frontmatter line {mark.line + 1})" if mark else ""
        return f"frontmatter is not parseable YAML{where}: {getattr(e, 'problem', e)}"
    if not isinstance(data, dict):
        return "frontmatter is not a YAML mapping"
    t = data.get("type")
    if not isinstance(t, str) or not t.strip():
        return "frontmatter has no non-empty string `type`"
    problems = v02_problems(data)
    return "; ".join(problems) if problems else None

STATUSES = ("draft", "stable", "deprecated")

def is_aware_datetime(v):
    return isinstance(v, datetime.datetime) and v.tzinfo is not None

def v02_problems(d):
    p = []
    if "timestamp" in d:
        p.append("legacy `timestamp` — use `generated: { by: <actor>, at: <datetime> }` (OKF v0.2)")
    if "status" in d and d["status"] not in STATUSES:
        p.append(f"`status: {d['status']}` is not one of {'|'.join(STATUSES)} (repo detail goes in `subject_status`)")
    if "generated" in d:
        g = d["generated"]
        if not isinstance(g, dict) or not isinstance(g.get("by"), str) or not g["by"].strip():
            p.append("`generated` needs a non-empty `by` actor")
        elif "at" in g and not is_aware_datetime(g["at"]):
            p.append("`generated.at` must be an ISO 8601 datetime with offset (e.g. 2026-11-02T00:00:00Z)")
    if "verified" in d:
        vs = d["verified"] if isinstance(d["verified"], list) else [d["verified"]]
        for v in vs:
            if not isinstance(v, dict) or not isinstance(v.get("by"), str) or not is_aware_datetime(v.get("at")):
                p.append("each `verified` entry needs `by` (actor) and `at` (datetime with offset)")
                break
    if "stale_after" in d and not is_aware_datetime(d["stale_after"]):
        p.append("`stale_after` must be an ISO 8601 datetime with offset")
    if "sources" in d:
        ss = d["sources"]
        if not isinstance(ss, list) or not all(isinstance(x, dict) and x.get("resource") for x in ss):
            p.append("every `sources` entry needs a `resource`")
    return p

def root_index_problem(text):
    """Bundle-root index.md may carry frontmatter with `okf_version` only."""
    block = frontmatter_block(text)
    if block is None:
        return "frontmatter block is not closed with ---"
    if yaml is None:
        return None
    try:
        data = yaml.safe_load(block)
    except yaml.YAMLError:
        return "frontmatter is not parseable YAML"
    if not isinstance(data, dict) or set(data) - {"okf_version"}:
        return "root index.md frontmatter may only carry `okf_version`"
    if not isinstance(data.get("okf_version"), str):
        return "`okf_version` must be a quoted string, e.g. \"0.2\""
    return None

def frontmatter_type(text):
    # extract block between first two --- lines, find a non-empty top-level `type:`
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
                if rel == "index.md":
                    err = root_index_problem(text)
                    if err:
                        errors.append(f"{rel}: {err}")
                else:
                    errors.append(f"{rel}: reserved file must NOT have frontmatter")
        else:
            if not has_frontmatter(text):
                errors.append(f"{rel}: missing YAML frontmatter")
            elif yaml is not None:
                err = yaml_check(text)
                if err:
                    errors.append(f"{rel}: {err}")
            elif not frontmatter_type(text):
                errors.append(f"{rel}: frontmatter has no non-empty `type`")

print(f"OKF validate: checked {checked} markdown files")
if yaml is None:
    print("WARN — PyYAML not installed: frontmatter only regex-checked, not parsed (pip install pyyaml)")
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
#   - skips any index.md whose first line (after optional frontmatter) is "<!-- okf:manual -->"
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

def unquote(v):
    """Resolve a quoted YAML scalar ("..." with \\ escapes, '...' with '' escapes)."""
    if len(v) >= 2 and v[0] == v[-1] == '"':
        return re.sub(r'\\(.)', lambda m: m.group(1), v[1:-1])
    if len(v) >= 2 and v[0] == v[-1] == "'":
        return v[1:-1].replace("''", "'")
    return v

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
                    out[m.group(1)] = unquote(m.group(2).strip())
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
    # only generate where there is something to list
    if not concepts and not subdirs:
        continue
    index_path = os.path.join(dirpath, "index.md")
    if os.path.exists(index_path):
        body = open(index_path, encoding="utf-8").read()
        # the bundle-root index.md may carry frontmatter (okf_version); look past it
        fmm = re.match(r"---\n.*?\n---\n", body, re.S)
        if fmm:
            body = body[fmm.end():]
        first = body.lstrip("\n").split("\n", 1)[0].strip()
        if first == "<!-- okf:manual -->":
            continue

    # heading: prefer the primary concept named after the dir, else title-case
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
- **Never delete superseded knowledge** — mark it `status: deprecated` after lifting
  live facts out.
- **Record who wrote it.** Every concept edit updates `generated: { by, at }` with
  the agent's own actor; only the owner's explicit confirmation becomes
  `verified: { by: human:<id>, … }`.
- **Danger zones are written once the hard way.** After any incident or near-miss,
  add a one-line danger zone to the domain `AGENTS.md` that names the trap and the
  safe alternative.
- **Keep secrets out of git, out of concepts, out of logs.** The `access:` field and
  `AGENTS.md` name the *scheme* and the *store path*, never the value.
- **Before sharing the repo with anyone:** it is personal by design (hosts, IPs,
  account names, decisions). Share this guide, not the repo.
