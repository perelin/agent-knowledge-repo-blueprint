# Agent Knowledge Repo — Blueprint

A single, self-contained guide you hand to an AI coding agent (Claude Code, Codex,
OpenCode, …) so it can bootstrap a **personal "agents" knowledge repo** for you:
a git repo of plain markdown that agents use to operate your servers, self-hosted
services, apps, home devices and SaaS APIs.

The structure follows the
[Open Knowledge Format (OKF)](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md):

- **Operative layer** — `AGENTS.md` (+ `CLAUDE.md` symlink) per repo and per domain:
  danger zones, access, common operations. Auto-loaded by the agent.
- **Knowledge layer** — one markdown concept per topic, with YAML frontmatter
  (`type`, `resource`, `status`, …). Read on demand.
- **Navigation layer** — generated `index.md` files for progressive disclosure.

Plus a secrets pattern (secret store as source of truth, `.env` files restored
from it) and two small scripts to validate the bundle and regenerate indexes —
embedded in the guide, so there is nothing else to download.

## How to use

1. Create an empty directory / git repo for your knowledge base.
2. Give your agent [`BOOTSTRAP-GUIDE.md`](BOOTSTRAP-GUIDE.md), e.g.:
   > Read BOOTSTRAP-GUIDE.md and set up this repo accordingly. Interview me about
   > my systems, then create the first domain.
3. Add one domain per system as you go. Run `docs/okf/validate.sh` and
   `docs/okf/generate-index.sh` after changes.

The guide itself is an OKF concept (note its frontmatter) — a live example of the format.

## Background

Distilled from a knowledge repo that has been in daily agentic use for managing a
homelab and a set of self-hosted services. The original stays private because it is
personal by design (hosts, accounts, decisions); this blueprint contains only the
structure and conventions.

## License

MIT — see [LICENSE](LICENSE).
