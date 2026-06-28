# AltaCoda Skills

Cross-agent **[Agent Skills](https://github.com/anthropics/skills)** published by
[AltaCoda](https://github.com/AltaCoda) for its products. Each skill is a folder
containing a `SKILL.md` and installs unchanged into Claude, Codex, Gemini, and any
other agent that follows the open standard.

These skills are **guidance-only**: they explain concepts, draft the exact artifact
you need, and tell you where to click. They never hold credentials or change
anything in your account.

## Install

Install one product's skill (recommended):

```bash
npx skills add AltaCoda/skills/sendops
```

Or browse and pick from everything in this repo:

```bash
npx skills add AltaCoda/skills
```

Keep your installed skills current:

```bash
npx skills update            # all skills
npx skills update sendops    # just one
```

`npx skills` ([vercel-labs/skills](https://github.com/vercel-labs/skills)) records
where each skill came from, so `update` re-pulls the latest version from here. No
manual re-download.

## Skills

| Skill | Install | What it does |
| --- | --- | --- |
| [`sendops`](./sendops) | `npx skills add AltaCoda/skills/sendops` | In-agent guide to [SendOps](https://sendops.dev) — AWS SES setup, domains & DKIM, production access, Lists & Segments, CEL predicates, deliverability reports, git-synced templates. |
| [`markbase`](./markbase) | `npx skills add AltaCoda/skills/markbase` | In-agent guide to [Markbase](https://markbase.cloud) — the MCP-reached Markdown document store: workspaces & paths, typed collections (`_schema.md`), per-folder conventions (`_markbase.md`), optimistic concurrency, and soft-delete. |

## How this repo is maintained

Each skill's **source of truth lives in its product's own repository** and is
mirrored here automatically on release. Edit skills upstream, not here — direct
edits to this repo will be overwritten by the next mirror. File issues against the
relevant product.
