---
name: markbase
description: Use this skill whenever the user wants to read, write, search, query, or organize content in a Markbase workspace, or whenever they mention Markbase, `_markbase.md`, `_schema.md`, or paths in `workspace/...` form. Markbase is a hosted Markdown document store reached over MCP — with typed collections, per-folder conventions, targeted edits, optimistic concurrency, document freezes, and soft-delete. Activate this skill before any first read or write so conventions are honored from the start. Re-read it if the agent's behavior with Markbase tools feels off.
---

# Markbase

Markbase is a hosted Markdown document store you access over MCP. Content lives in organizations → workspaces → folders → `.md` files (free-form documents) or UUIDv7-named records inside typed collections. Two filenames are reserved by the product: `_markbase.md` (per-folder conventions, walk-up resolution; agent- or human-authored with the `agent_md.write` scope, delete is dashboard-only) and `_schema.md` (typed-collection contract).

**The tool surface, capability inventory, and per-host install steps are dynamic — they live at the agent-curated docs and change as the product evolves. Always fetch the index before acting. Don't rely on what this skill says about specifics; this skill is the posture, the docs are the API.**

The agent docs index:

**https://help.markbase.cloud/agents/index.md**

Fetch it first. Everything else — tool reference, host-specific install steps, concepts (workspaces, documents, typed collections, `_markbase.md`, search), editing, concurrency rules, freezes, troubleshooting — is reachable as a link from there. Follow the links rather than guessing at paths.

## Endpoint

```
https://api.markbase.cloud/mcp
```

OAuth 2.1 with PKCE and DCR. If Markbase tools aren't visible in your tool list, the connection isn't wired yet — fetch the index above and follow it to the install steps for your host.

## Principles that don't change

These are stable posture, not implementation details. The specifics (field names, exact tool names, current capabilities) belong to the docs.

1. **Conventions live in `_markbase.md`.** Before writing into any folder, read the chain of `_markbase.md` files from the workspace root down to the target's immediate parent. They compose; later wins on conflict. A `_markbase.md` may exist at any folder root. **You may author one — it is workspace memory, not a scratchpad.** Creating or updating a `_markbase.md` over MCP needs the `agent_md.write` scope (a missing scope returns `forbidden`); write only **conventions for this workspace/folder** (its purpose, naming, layout, the norms an agent should follow), never codebase or generic agent instructions, and don't confuse it with codebase-side `AGENTS.md`/`AGENT.md` files other harnesses keep on the local filesystem (those are unrelated and outside Markbase's scope). If a folder lacks one and conventions are worth capturing, draft the content and confirm with the human before writing it. **Deletion is dashboard-only** — Markbase rejects an MCP `delete_doc` of any `_markbase.md` with `forbidden`.

2. **Typed collections are real.** If a folder contains `_schema.md`, records in that folder must conform to it. Read the schema before writing records; the server validates and rejects non-conforming writes. A schema may also declare its records immutable after creation, in which case rewriting one is refused and the remedy is a new, superseding record.

3. **Edit; don't retype.** For a document that already exists, send the parts that change rather than the whole body. Re-sending a whole file costs tokens in both directions and asks you to reproduce, byte-for-byte, the majority you aren't touching — which is where silent corruption comes from. Reserve whole-body writes for creating a document or deliberately replacing one, and when you already hold exact bytes, use the presigned upload path rather than pasting them into a tool call.

4. **Writes are optimistic, and nothing is rebased for you.** Every write carries the version you read (for updates) or an explicit "create only" marker (for new docs); targeted edits require it outright. There is no force-write. On conflict, look at what the other writer changed, re-read, redo your change against the new text, and retry with a bounded budget. The server will never guess at your intent and reapply a failed edit.

5. **Renaming is a first-class operation.** Move a document with the move primitive, which preserves its content, hash and history. Do **not** emulate a rename by writing the new path and deleting the old one — that starts the destination's history from nothing and leaves a misleading trash entry behind. Actual deletion is soft: it goes to trash, recoverable for a bounded window.

6. **A freeze is irreversible from here.** Markbase can lock a document against all further writes. There is no unlock tool and no unlock scope — only a human, in the dashboard, can lift it. **Say so plainly before you freeze anything**, and never freeze something speculatively or to "protect" your own work in progress. If a write comes back refused because a document is frozen, that is terminal: surface the reason to the human, don't retry, and don't go looking for a way around it.

7. **Discover, don't invent.** Paths are discovered through the tools, not constructed from assumptions about layout. When in doubt, list first. Batch your reads where the tools allow it rather than issuing a run of single reads.

8. **Small surface by design.** If a capability isn't reachable from the index, it doesn't exist. Don't guess at endpoints, tool names, or arguments — fetch the docs.

## Working pattern

For any non-trivial task:

1. Fetch the index (and any pages it routes you to for this task) so you're working from the current API.
2. Use the discovery tool to ground yourself in the target's actual structure.
3. Read any governing `_markbase.md` files along the walk-up path.
4. If you're inside a typed collection, read its `_schema.md`.
5. Read the target document (if it exists) to capture its current version — and note whether it is frozen before you spend a turn composing a change.
6. Then act, with the smallest write that does the job.

The cost is a few reads. The win is no schema violations, no conflict retries, no wasted tokens re-emitting text you weren't changing, and no fighting conventions you didn't know existed.
