# AltaCoda Skills

Cross-agent **[Agent Skills](https://github.com/anthropics/skills)** published by
[AltaCoda](https://github.com/AltaCoda) for its products. Each skill is a folder
containing a `SKILL.md` and installs unchanged into Claude, Codex, Gemini, and any
other agent that follows the open standard.

These skills are **guidance-only**: they explain concepts, draft the exact artifact
you need, and tell you where to click. They never hold credentials or change
anything in your account. The one thing a skill may do on your machine is register
its product's MCP server in your agent's own config — always asking first — because
that is the step between installing a skill and it being any use.

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
| [`sendops`](./sendops) | `npx skills add AltaCoda/skills/sendops` | In-agent guide to [SendOps](https://sendops.dev) — AWS SES setup, domains & DKIM, production access, Lists & Segments, SendQL predicates, Drip Workflows, deliverability reports, git-synced templates. |
| [`bench47`](./bench47) | `npx skills add AltaCoda/skills/bench47` | In-agent guide to [Bench47](https://bench47.com) (formerly Markbase) — the MCP-reached Markdown document store. Tells your agent to read the project's workspace before a task and write back what it learned after. Covers workspaces & paths, typed collections (`_schema.md`), per-folder conventions (`_markbase.md`), targeted edits, optimistic concurrency, document freezes, soft-delete, temporary inboxes (mint a short-lived address so the user can forward you an email, then read it back with its authentication verdicts), and due dates (park work in a document and have the user reminded with a resume prompt). Also carries the per-host MCP registration steps, so a CLI agent can finish connecting itself after the install. |
| [`markbase`](./markbase) | `npx skills add AltaCoda/skills/markbase` | The old name of the `bench47` skill, with the same instructions. It stays here because existing installs and released self-hosted dashboards use it. New installs should use `bench47`. |

## How this repo is maintained

Where each skill's source of truth lives depends on the skill:

- **`sendops`** lives in the SendOps repository and is mirrored here
  automatically when it changes there. Edit it upstream, not here: direct edits
  will be overwritten by the next mirror.
- **`bench47`** is maintained here. Change it with a pull request against
  `main`; `npx skills add` and `npx skills update` install from `main`.
- **`markbase`** is the same skill under its old name. Keep it at this path, and
  make each change to `bench47` here too. It differs only in its `name` and a
  one-line note at the top.

File issues against the relevant product.
