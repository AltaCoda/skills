---
name: sendops
description: >-
  Your in-agent guide to SendOps (sendops.dev), the email-infrastructure control
  plane on AWS SES. Use this skill whenever the user works with SendOps or wants
  help with: connecting an AWS/SES account, verifying domains and DKIM, leaving
  the SES sandbox (production access), building Contact Lists or Segments,
  writing CEL segment predicates, defining contact attributes, reading
  deliverability / engagement / per-message reports, the undeliverable list vs.
  the suppression list, bounce and complaint hygiene, open/click tracking, or
  git-synced email templates. Trigger it even when the user never says "SendOps"
  by name — "why aren't my opens being tracked", "segment trial users who
  haven't opened in 30 days", "my domain won't verify", or "suppression list vs
  undeliverable list" all mean this skill should be consulted. It makes a complex
  product feel simple: explains the concepts, drafts the exact CEL, lists, and
  attribute schemas, interprets reports, and says exactly where to click.
---

# SendOps Co-pilot

SendOps ([sendops.dev](https://sendops.dev)) is an **email-infrastructure control plane** on AWS SES. It sets up SES, EventBridge, and supporting resources **inside the customer's own AWS account** (via CloudFormation) and then manages templates, audiences, reporting, deliverability, and analytics on top. **It is not a proxy** — email always sends through the customer's SES directly. Positioning: *"Send more. Pay the same."*

This skill exists because SendOps has grown powerful enough that a normal user can feel lost — between SES setup, Lists and Segments, reports, and templates there's a lot of surface area. Your job is to be the calm expert sitting next to them: translate what they want into the right SendOps concept, hand them the exact thing to create (a CEL predicate, a list, an attribute, a DNS record), and point them to the precise screen or endpoint. You make the product feel small.

This skill is a **single self-contained file**: a short router (this part) followed by four in-depth reference sections (Setup & onboarding, Lists & Segments, Reports & deliverability, Templates). Everything you need is below — read the matching section before giving detailed guidance.

## What this skill does and doesn't do

This is a **guidance skill**. It does not call the SendOps API, hold credentials, or change anything in the user's account on its own. It helps the user *think, draft, and navigate*:

- **Explain** any SendOps concept in plain terms and how it maps to AWS SES underneath.
- **Draft** the exact artifact the user needs — a CEL segment predicate, a List definition, an attribute-registry entry, a template manifest snippet, DNS records to paste — ready for them to apply.
- **Interpret** what a report or a status badge is telling them, and what to do about it.
- **Navigate** — tell them which page, tab, button, or endpoint to use.

When something requires an action in SendOps, say so explicitly and give the path. Don't imply you performed it. Typical phrasing: *"Here's the predicate — save it as a Segment via `POST /api/v1/orgs/{slug}/segments`, or commit it to your connected repo as a `.cel` file."* If the user has the SendOps Public API or its MCP server connected as a tool in this session, you may additionally read live data through it — but never assume that; the baseline is advice, not execution.

## How to help — the loop

1. **Locate the request on the product map** (below). Most confusion is really "which of these four areas am I in?" Name it for the user.
2. **Read the matching reference section below** before answering anything non-trivial — the details (CEL grammar, classification rules, sync lifecycle) are exact and easy to get subtly wrong from memory.
3. **Hand over a concrete artifact**, not just prose. A user who asked "how do I segment dormant trial users" should leave with a predicate they can paste.
4. **Say where it goes and what happens next** (which screen / endpoint; whether it's view-only; whether a sync or refresh is needed).
5. **Flag the gotchas** that bite people — null handling in CEL, tracking that isn't actually on, suppression vs. undeliverable, sandbox limits.

## Product map — the four areas

Route the user to the right reference section below. Read the section before giving detailed guidance.

| If the user is asking about… | Area | Read |
| --- | --- | --- |
| Connecting AWS, the CloudFormation stack, verifying a domain/DKIM, getting out of the SES sandbox (production access), importing an existing ("brownfield") SES setup, channels = config sets, why a config set shows "tracking off" | **Setup & onboarding** | § Reference: Setup & onboarding |
| Contact Lists, Segments, CEL predicates, the attribute registry, who's in an audience and why, preview/dry-run, managed vs. git-backed definitions | **Lists & Segments** | § Reference: Lists & Segments |
| Deliverability / engagement / per-message / template reports, opens & clicks, ISP breakdown, the undeliverable list, the suppression list, bounces & complaints | **Reports & deliverability** | § Reference: Reports & deliverability |
| Email templates, the connected Git repo, the manifest, Handlebars, test sends, deploying templates to SES, exporting existing SES templates | **Templates** | § Reference: Templates |

If a request spans areas (common — e.g. "segment everyone who bounced, then stop emailing them"), name each area and read each section you need.

## Core vocabulary (so you never mislead the user)

- **Control plane, not proxy.** SendOps configures the customer's AWS account and reports on it. Sending happens via their SES. If SES is down or in sandbox, SendOps can't send around that.
- **Channel = SES configuration set**, exactly one-to-one. "Set up a channel" means "this maps to a SES config set."
- **AWS account is shared-capable.** Several SendOps orgs can link one AWS account: the first provisions the CloudFormation stack; later orgs *join* with no deploy. An identity or config set is "active" in at most one org at a time.
- **List ≠ Segment.** A **List** is membership you *assign* (static, explicit add/remove). A **Segment** is membership you *describe* with a CEL predicate (dynamic; contacts flow in and out as their data changes). This single distinction drives everything downstream — see the Lists & Segments reference.
- **Membership ≠ mailability.** Being in a List or Segment does not mean SendOps will email someone. Consent and suppression are enforced separately, at send time.
- **Undeliverable list ≠ Suppression list.** The **suppression list** mirrors the AWS SES suppression list. The **undeliverable list** is SendOps's own derived view based on configurable rules over bounce/complaint history. They answer different questions — see the reports reference.
- **Git is the source of truth for templates** (and, when a repo is connected, for segment/attribute *definitions* too). Data — list membership, attribute values — never lives in git; it lives in SendOps.
- **The audience UI is view-only in this release.** Lists, Segments, and attributes are *created and edited* via the API or a connected GitHub repo; the dashboard displays them, deep-links to GitHub, and lets you pause/resume a managed Segment — but there is no in-app editor. Don't tell a user to "click New Segment" in the UI — tell them how to author the definition. (Verify against their build if they say the UI differs; products move.)
- **Reports refresh manually**, not live. If numbers look stale, the user may need to hit refresh.

## Tone

Be the friendly expert. Lead with the answer, then the artifact, then where it goes. Prefer a worked example over abstract description. When you hit a genuine limitation of the current release (no in-app segment editor, public API is read-only, no dedicated-IP management), say so plainly and give the real path forward rather than pretending. Keep AWS/SES accuracy high — users trust this skill precisely because it tells them the truth about what's happening under the hood.

---

# Reference: Setup & onboarding — connecting AWS, domains, sandbox, channels, sync

This is everything about getting an AWS/SES account wired into SendOps and keeping it in sync. Read it before answering anything about onboarding, the CloudFormation stack, DKIM, production access, brownfield import, channels, or tracking.

## The mental model

SendOps runs a **control plane** in the customer's own AWS account. Onboarding is a **four-step sequential flow** (each step must pass before the next opens):

1. **Connect AWS** — cross-account IAM role + CloudFormation stack.
2. **Validate stack** — SendOps probes the deployed resources to confirm they work.
3. **Domain / identity** — verify a sending identity (domain via DKIM, or a single email).
4. **Test send** — prove an email actually goes out.

Steps are tracked in `org_onboardings` + `onboarding_steps`. Each step is `pending` → `in_progress` → `passed` / `failed`, evaluated **independently** (a later step can't pass until the earlier one has). When a step fails it stores a structured error code (e.g. `aws_connection_invalid`, `invalid_arn`, `invalid_region`).

## 1. Connecting AWS (assume-role + CloudFormation)

The user runs a **CloudFormation stack** in their AWS account from a SendOps-hosted template. That stack creates a cross-account IAM role (`SendOpsRole`) that SendOps assumes via STS, plus the event plumbing. The user then pastes the **role ARN** + **region** into SendOps, which validates by performing an `AssumeRole`.

What the CloudFormation stack provisions (current template **version 12** — 7 resources):

| Resource | What it is |
| --- | --- |
| `SendOpsRole` | Cross-account IAM role. Trust policy allows SendOps's AWS account to `AssumeRole` **only with the correct ExternalId**. Grants the SES / EventBridge / CloudFormation permissions SendOps needs. |
| `SendOpsConfigSet` (`sendops-events`) | An **infrastructure** SES configuration set used for stack validation. **Not a user-facing channel** — it never appears in the channel list. |
| `SendOpsEventDestination` | EventBridge destination on `sendops-events` capturing all 10 SES event types (send, reject, bounce, complaint, delivery, open, click, renderingFailure, deliveryDelay, subscription). |
| `SendOpsConnection` | EventBridge Connection holding the ingest API-key auth (`x-api-key`). |
| `SendOpsApiDestination` | EventBridge API destination → SendOps ingest webhook, rate-limited ~300/s. |
| `SendOpsEventBridgeRole` | IAM role that lets the EventBridge rule invoke the API destination. |
| `SendOpsEventRule` | EventBridge rule matching `source: aws.ses`, forwarding to the API destination. |

**ExternalId** is generated per-org at onboarding start and stored encrypted. It is what makes the role assumption safe (an attacker who learns the role ARN still can't assume it).

> **Gotcha — region whitelist.** The region must be one of SendOps's allowed regions. A region outside the whitelist fails with `invalid_region` before any AWS call.

> **What happens the moment Connect succeeds:** SendOps immediately enqueues a **full sync** (trigger `onboarding`) plus a contact-list sync. This is the brownfield import path — existing SES domains, identities, and config sets show up right away instead of waiting for the hourly scheduler.

### Shared AWS accounts — the "join" flow

One AWS account can back **several SendOps orgs**. The first org to connect **provisions** the CloudFormation stack. Any later org that pastes a role ARN from an **already-connected** account is offered a **join** instead — it reuses the existing stack with **no new CloudFormation deploy**.

- Before connecting, SendOps does a zero-AWS-call detection: *is this account already connected, and can you join it?* You can join only if you're an **owner or admin** of an org already on that account.
- SendOps also **proactively** lists joinable accounts on the Connect screen, so an admin can join without pasting an ARN at all.
- The joining org's region must **match** the account's stored region (else `join_region_mismatch`).
- One AWS account → many member orgs; each org → at most one AWS account.
- An identity or config set is **active in at most one org at a time** — the others see it as discovered/claimed elsewhere. This is how SendOps avoids two orgs fighting over the same SES resource.

When advising: if "this account is already connected" appears, the user almost certainly wants **Join**, not a second stack. Tell them to use the joinable-accounts offer.

## 2. Validate stack

After Connect, an async **stack validation** job probes the deployed resources: config set present, EventBridge rule + targets correct, SES account reachable, write access, channel-settings access, template write access. It also reads SES account status and records whether the account is **in the SES sandbox** (informational — validation still passes in sandbox).

> The current required CloudFormation **template version is 12**. If the user later tries to request production access on an older stack, SendOps returns `CF_TEMPLATE_OUTDATED` — they must update the stack first. If you see that error, the fix is "update your CloudFormation stack to the latest SendOps template," not anything in the form.

## 3. Domain verification & DKIM

Three identity paths:

- **`new_domain`** — SendOps calls SES to create a domain identity and returns **3 DKIM CNAME records** of the form `{token}._domainkey.{yourdomain}` → `{token}.dkim.amazonses.com`. The user adds these at their DNS provider. SendOps polls until DKIM status is `SUCCESS`.
- **`new_email`** — a single email identity. SES emails a verification link to that address. No DKIM, no polling job (it flips when they click the link).
- **Import existing** — for brownfield accounts, SendOps reconciles all existing SES identities; if a verified one exists, step 3 auto-advances (the user skips DNS entirely).

### DNS provider detection — guidance only

SendOps does an NS lookup and classifies the domain's DNS provider (Route 53, Cloudflare, GoDaddy, Namecheap, … 17 providers) to show **provider-specific setup guidance**.

> **Important truth to tell users:** SendOps does **not** create DNS records for them — not even on Route 53. Detection only tailors the instructions. The user (or their DNS admin) pastes the CNAMEs themselves. "Route 53 detected" means "here's how to add them in Route 53," not "done automatically."

### The DKIM polling schedule

The verification job re-checks on a decaying cadence and gives up after 7 days:

- First hour: every **2 minutes**
- Hours 1–24: every **10 minutes**
- After 24h: every **hour**
- Hard cutoff: **7 days** from start

When DKIM verifies, SendOps marks the domain verified, advances onboarding, and **provisions the default channels** (see below). If a user says "my domain still says pending," check (a) the CNAMEs are exact and not proxied/altered, and (b) it's within the 7-day window.

## 4. Leaving the SES sandbox (production access)

New SES accounts are **in the sandbox**: they can only send to verified identities and have tiny quotas. Production access is requested **through SendOps**, which calls SES `PutAccountDetails` on the customer's account.

- Permissions: `ses.production_access.view` (see status), `ses.production_access.submit` (request).
- The request form validates: **mail type** is `TRANSACTIONAL` or `PROMOTIONAL`; **website URL** is valid http(s); **use-case description** is **≥ 50 characters**; no other open request; and the **CloudFormation template is current (v12)**.
- After submit, SendOps polls SES for the outcome on a decaying schedule, timing out to `failed` after ~14 days.

> **Gotcha — AWS never signals "denied."** SES exposes "production access enabled = true/false" but no denial reason. So a granted request is detected; a **denied** one simply keeps reading as "under review" until the ~14-day timeout flips it to `failed`. If a user's request sits in review for days, that may be a silent denial — advise them to check the AWS Support case directly.

> **Sandbox + test send (step 4):** while in the sandbox, the test-send `to:` address must itself be a verified identity, or the send fails with `sandbox_email_not_verified` (the error lists the verified identities available).

## Brownfield import & "Adopt"

If the customer already uses SES ("brownfield"), connecting imports their existing setup. SendOps classifies the account's **maturity** (greenfield vs brownfield) from signals like: production access already granted, ≥1 existing config set, ≥3 verified identities, or ≥10 suppression entries.

**"Adopt"** is the specific action for an **existing config set** that sync found but isn't wired to SendOps yet (state `discovered`). Adopting it:

1. Adds a SendOps-shaped EventBridge **event destination** to that config set (Open/Click only if the channel's `tracking_enabled` is true).
2. Flips its state `discovered` → `active`.
3. **Touches nothing else** — TLS policy, suppression, max-delivery, and every other existing setting are preserved exactly.

> Discovered-but-not-adopted config sets keep receiving events but are **not** in the active pool used for sending. If a user "sees" a channel but "can't send through it," it likely needs **Adopt**. Permission: `channels.adopt`.

## Channels = SES configuration sets (1:1)

Every **channel** maps to exactly one SES **configuration set** (`ses_config_set_name`), and vice versa. After domain verification, SendOps provisions four defaults in the customer's SES account:

| Channel | Default suppression | Tracking |
| --- | --- | --- |
| `sesmail-default` | BOUNCE | off |
| `sesmail-transactional` | BOUNCE | off |
| `sesmail-marketing` | BOUNCE, COMPLAINT | **on** |
| `sesmail-onboarding` | BOUNCE | off |

(All default to TLS `REQUIRE`, sending enabled.) Provisioning is idempotent — it skips if channels already exist.

### The "tracking off = no event destination" trap

This is the single most-misread thing in setup. There are **two independent flags** on a channel:

- **`TrackingEnabled`** — true only if an **enabled** event destination targets the SendOps EventBridge bus **and** includes both `OPEN` and `CLICK`. Controls whether **open/click** analytics flow.
- **`NeedsEventDestination`** — true if **no** enabled destination targets the SendOps bus at all. When true, the channel sends **nothing** to SendOps — not even send/delivery/bounce/complaint.

So:

| State | Opens/clicks? | Bounces/deliveries? | Meaning |
| --- | --- | --- | --- |
| Tracking on | ✅ | ✅ | Fully wired |
| Tracking **off**, destination present | ❌ | ✅ | Working pipeline, engagement tracking disabled by choice |
| `NeedsEventDestination` = true | ❌ | ❌ | **Broken pipeline** — fix this |

> When a user says "my opens aren't tracked," distinguish the two: if **only** opens/clicks are missing, tracking is simply off (enable it on the channel). If **all** events are missing (no bounces, no deliveries either), the config set has no SendOps event destination — that's the real break, and **Adopt** (for a discovered set) or re-provisioning fixes it.

## Sync behavior

Sync **reads AWS state and mirrors it into SendOps** — it does not push local changes back to AWS. A full sync runs three syncers in order: **domains → email identities → channels**.

- **State machine:** present in AWS → `active` (or stays `discovered` if never adopted); absent from AWS → `detached`; reappears → `active`. Sync **never auto-activates** a discovered resource — adoption is an explicit user choice.
- **Drift detection:** sync compares stored channel settings (TLS, max-delivery, suppression, tracking redirect) and domain records (DKIM, SPF, DMARC) against live AWS and writes **drift audit rows** — it flags drift, it does **not** auto-correct it.
- **The `sendops-events` infra config set is always skipped** by the channel syncer — it's plumbing, not a channel.

When sync runs:

- **Onboarding** — immediately after Connect/Join.
- **Scheduled** — hourly, staggered across orgs so they don't all hit AWS at once.
- **On-demand** — `POST /api/v1/orgs/{slug}/sync` (permission `sync.trigger`), with a 5-minute cooldown to prevent hammering.
- **After mutations** — creating/deleting/editing a channel, adding/removing a domain, or importing identities all enqueue a sync.

> A per-org Redis mutex (5-min TTL) prevents concurrent syncs. If a user triggered a sync seconds ago and "nothing happened," they're likely inside the cooldown — tell them to wait.

## Quick navigation cheats

- Connect AWS / see onboarding: the onboarding flow (4 steps).
- Production access: status + submit under the deliverability/account area (perms `ses.production_access.*`).
- Channels: the channels area — look for `discovered` badges that need **Adopt**, and tracking on/off per channel.
- Force a refresh from AWS: trigger a sync (perm `sync.trigger`); respect the 5-min cooldown.

---

# Reference: Lists & Segments — audiences, CEL, attributes

Everything about building audiences in SendOps. Read it before drafting a List, a Segment predicate, or an attribute schema — the CEL grammar and the null contract are exact and very easy to get subtly wrong from memory.

## The one distinction that drives everything

- **List** = membership you **assign**. Static. You explicitly add/remove contacts; a contact stays in until removed (or deleted). Think "imported CSV," "beta testers," "everyone from the conference."
- **Segment** = membership you **describe**. Dynamic. You write **one CEL predicate**; a contact is a member iff the predicate is `TRUE` for them **right now**. As their data changes, they flow in and out automatically.

Both record membership as an **append-only event log** (no derived "current members" table). "Who's in it now" is computed at read time as the latest event per contact:

- Lists: latest `added`/`removed` event wins.
- Segments: latest `entered`/`exited` transition wins.

This is why **membership over time** is a first-class thing: SendOps keeps the full transition history, so future features (journeys, broadcasts) can ask "who entered this segment last week."

> **Membership ≠ mailability.** Being in a List/Segment does not mean SendOps will email the contact. Consent (topic preferences, unsubscribe) and suppression are enforced **separately at send time**.

## The current UI is view-only

In this release the audience dashboard is **read-only**:

- You can **view** Lists, Segments, their predicate, eval class, status, member count, and current member list.
- You can **pause/resume** a *managed* Segment (permission `segments.manage`). Git-backed segments have no pause button — you pause them by removing the source file from the repo.
- You **cannot** create or edit Lists/Segments/attributes in the UI. There is no in-app editor.

To create/edit, the user uses **either**:

1. **The session API** — `POST/PUT/DELETE /api/v1/orgs/{slug}/segments` (and the lists/attributes equivalents). Full CRUD. This is what the dashboard would call; it's dashboard-token auth.
2. **A connected GitHub repo** — definitions live as files; SendOps syncs them. (See "Managed vs git-backed.")

> The **Public API v1** (`/v1/...`, API-key auth) is **read-only** for audiences — `GET /v1/lists`, `GET /v1/segments`, `GET /v1/attributes`, plus `POST /v1/segments/preview`. There is **no** v1 write endpoint for lists/segments. So "create a segment via the public API" is not possible today — direct them to the session API or git.

So your job for any "I want an audience of X" request is to **hand them the artifact** (a `.cel` predicate, or a list definition) and tell them which of the two authoring paths to use.

## The CEL environment (the exact surface)

Segment predicates are written in **CEL** (Google's Common Expression Language, cel-go v0.21.0). A predicate must evaluate to a **bool**. The environment exposes exactly four namespaces plus a few built-ins — nothing else. Each saved segment is stamped with a **profile version** (currently `1`).

### `contact.<attr>` — your registered attributes

Per-org custom attributes from the **attribute registry** (see below). Reference as `contact.tier`, `contact.score`, `contact.signup_date`, etc. Attribute names match `^[a-z][a-z0-9_]{0,62}$`.

Typed by the registry. The compiler emits the right cast: numbers, booleans, datetimes, strings, enums (see Attribute registry). **Absent attributes are NULL — read the null contract below before relying on this.**

### `engagement.*` — six derived, live fields

Computed live from message events (not stored attributes). Contacts with no events get **0** (counters) or the unix epoch (timestamps) — **never null**.

| Field | Type | Meaning |
| --- | --- | --- |
| `engagement.opens_30d` | number | unique opens, rolling 30 days |
| `engagement.clicks_30d` | number | clicks, rolling 30 days |
| `engagement.sends_30d` | number | sends, rolling 30 days |
| `engagement.last_open_at` | datetime | last open |
| `engagement.last_click_at` | datetime | last click |
| `engagement.last_send_at` | datetime | last send |

### `consent.*` — four boolean fields

Mirrored from topic/unsubscribe state. Absent → **false** (never null).

| Field | Meaning |
| --- | --- |
| `consent.subscribed` | subscribed to at least one topic |
| `consent.opted_out` | opted out |
| `consent.unsubscribed_all` | global unsubscribe |
| `consent.suppressed` | on the suppression list |

### `list.member("<uuid>")` — cross-reference a List

Boolean function. The argument must be a **string-literal List UUID**: `list.member("0190aa...-...")`. True iff the contact is currently in that List. Lets a Segment build on a List.

### Built-ins for time

- `now()` — current time (folded to evaluation time). **Using `now()` makes the segment "sweep-class"** (see eval classes).
- `duration("720h")` — a Go duration string (720h = 30 days).
- `timestamp("2026-01-01T00:00:00Z")` — RFC 3339 literal.

Temporal arithmetic is allowed: `engagement.last_open_at < now() - duration("720h")`.

### Operators & methods available

- Comparison: `==`, `!=`, `<`, `<=`, `>`, `>=` (numeric comparisons work across int/double).
- Boolean: `&&`, `||`, `!`.
- Membership: `value in ["a", "b", "c"]` — RHS must be a **list literal**, max 100 elements.
- String methods (receiver form): `contact.email.endsWith("@acme.com")`, `.startsWith(...)`, `.contains(...)` — compile to SQL `LIKE`.
- Presence: `has(contact.tier)` — true iff the attribute is set. **This is the only macro enabled.**

### Explicitly NOT supported (don't draft these)

- `??` null-coalescing — **not supported** (fail closed).
- Ternary `cond ? a : b` — **not supported**.
- Comprehension macros (`.exists`, `.all`, `.map`, `.filter`) — **disabled**.
- Arbitrary function calls; map/struct literals; list literals anywhere except the RHS of `in`.
- Reserved roots that can't be attribute names: `contact`, `engagement`, `consent`, `list`, `now`, `duration`, `has`.

### Size governors (hard limits — exceeding any is a 422)

- ≤ 200 AST nodes total
- ≤ 20 nesting depth
- ≤ 100 elements in any `in [...]`
- ≤ 8 distinct `engagement.*` references
- ≤ 8 `list.member(...)` calls

If a predicate is rejected for size, simplify (fewer OR branches, smaller `in` lists, or split into two segments).

## ⚠️ The null-handling footgun (read this twice)

A contact is a member of a segment **iff the predicate evaluates to TRUE. NULL, FALSE, and error all exclude.** And **a missing attribute is NULL**, not a default value.

Concretely:

- `contact.tier != "free"` — a contact who has **no `tier` attribute at all** evaluates to NULL → **excluded**. Even though "not free" sounds like it should include them. This surprises everyone.
- `contact.score > 50` — contacts **without** a `score` are NULL → excluded (they are **not** treated as 0).

Why: each attribute access compiles to a guarded expression that yields SQL `NULL` when the key is absent, and ClickHouse three-valued logic (`NULL OR TRUE = TRUE`, `NULL AND FALSE = FALSE`) mirrors cel-go's Kleene logic exactly. The two engines agree; the result is "absent → excluded."

**How to write predicates that mean what the user wants:**

| User intent | Wrong (silently drops attribute-less contacts) | Right |
| --- | --- | --- |
| "everyone who isn't on the free plan, including unknowns" | `contact.tier != "free"` | `!has(contact.tier) \|\| contact.tier != "free"` |
| "anyone whose score is over 50" | `contact.score > 50` | `has(contact.score) && contact.score > 50` (explicit) |
| "people with no tier set" | — | `!has(contact.tier)` |

`has(...)`, `engagement.*`, and `consent.*` are **always** non-null, so they're safe to use bare. Only `contact.<attr>` can be null. When you draft any predicate that includes a `contact.*` comparison, **proactively decide and state** how attribute-less contacts should be treated.

## Attribute registry

Custom attributes must be **registered** before use. An attribute has:

- `name` — `^[a-z][a-z0-9_]{0,62}$`, not a reserved root.
- `type` — one of `string`, `number`, `bool`, `datetime`, `enum`.
- `enum_values` — required when type is `enum`; the allowed set is enforced **at write time**, not in predicates.
- `description` — optional.

Rules:

- **Type is immutable once values exist.** You can't change `score` from `string` to `number` after data has been written. Plan the type up front.
- Values are written per-contact via `PUT /api/v1/orgs/{slug}/contacts/{id}/attributes`; writes are validated against the registry and trigger incremental re-evaluation of any segment referencing that attribute.
- Read the registry via Public API: `GET /v1/attributes`, `GET /v1/attributes/{id}` (scope `api.attributes.view`). Management is session/git only.

An on-disk attribute schema (for git-backed definitions) is a JSON map:

```json
{
  "tier": { "type": "enum", "enum_values": ["free", "pro", "enterprise"], "description": "Billing tier" },
  "score": { "type": "number" },
  "signup_date": { "type": "datetime" }
}
```

## Preview / dry-run before you save

Always preview a non-trivial predicate before committing it.

- Session: `POST /api/v1/orgs/{slug}/segments/preview` (permission `segments.manage` — preview runs an arbitrary predicate, so it's write-class).
- Public API: `POST /v1/segments/preview` (scope `api.segments.view`).

Returns a **count**, the **eval class**, and a small **sample** of `{contact_id, email}`. No membership is written — it's a pure read. Use it to sanity-check size *before* you commit a definition to git or POST it.

## Managed vs git-backed definitions

Every Segment/attribute has an **origin**:

- **`managed`** — created/edited via API or dashboard. Mutable.
- **`git`** — synced from a connected GitHub repo. **Read-only in SendOps** (the repo is the source of truth). Trying to edit/delete a git-backed definition via API returns **409**.

The git layout lives under the repo's manifest (`sendops.json`), alongside templates:

```json
{
  "segments": {
    "high-value": { "path": "audience/high-value.cel", "name": "High value", "description": "Pro plan, engaged" }
  },
  "attributes": "audience/attributes.json"
}
```

Each `.cel` file contains **only the predicate string**, e.g. `audience/high-value.cel`:

```
contact.plan == "pro" && engagement.opens_30d > 5
```

Sync behavior:

- Triggered on push to the default branch, on GitHub connect, or `POST /api/v1/orgs/{slug}/definitions/sync`.
- **Attributes apply first**, then segments (so predicates can type-check against just-synced attributes).
- An **invalid** predicate is stored with `status=invalid` + a reason — **not dropped**. The user fixes the file and re-pushes.
- A segment **removed** from the repo becomes `archived` (its membership history is preserved). A removed attribute is hard-deleted.
- **Git wins:** a git definition supersedes a managed one with the same key and flips its origin to `git`.

> **Data never lives in git.** Only *definitions* (predicate strings, attribute schema) sync from the repo. Attribute **values**, list **membership**, and consent are all in SendOps. Never tell a user to commit contact data to the repo.

## Eval classes & freshness (so you can explain staleness)

A segment is classified by how it must be re-evaluated:

- **incremental** — re-evaluated per-contact when a referenced attribute changes (near-real-time).
- **sweep** — periodically re-evaluated for the whole population (segments using `now()`/time windows can only be kept fresh by a sweep). The sweep runs roughly every few hours.
- **both** — needs both.

So a "dormant 30 days" segment (uses `now()`) is sweep-class: membership updates on the sweep cadence, not instantly. If a user expects a time-window segment to update the instant a clock ticks, explain the sweep.

## Copy-paste recipe book

All of these are predicate strings — drop into a `.cel` file or the API body. Adjust attribute names to the user's registry.

**Engaged pro users**
```
contact.plan == "pro" && engagement.opens_30d > 0
```

**Dormant — no open in 30 days (and we have sent to them)**
```
engagement.sends_30d > 0 && engagement.last_open_at < now() - duration("720h")
```

**Trial users who never engaged**
```
contact.plan == "trial" && engagement.opens_30d == 0
```

**Not on the free plan — including contacts with no tier set**
```
!has(contact.tier) || contact.tier != "free"
```

**High score, explicitly require the attribute**
```
has(contact.score) && contact.score >= 80
```

**One of several tiers**
```
contact.tier in ["pro", "enterprise"]
```

**Corporate domain only**
```
contact.email.endsWith("@acme.com")
```

**Subscribed and not suppressed (mailable-ish — still re-checked at send)**
```
consent.subscribed && !consent.suppressed
```

**In an existing List and engaged**
```
list.member("PUT-LIST-UUID-HERE") && engagement.clicks_30d > 0
```

**Signed up in the last 7 days**
```
has(contact.signup_date) && contact.signup_date > now() - duration("168h")
```

When you hand any of these over: (1) confirm the referenced attributes are **registered** and the right **type**, (2) decide the null behavior explicitly, (3) tell them to **preview** it, then (4) save via API or commit the `.cel` to the repo.

---

# Reference: Reports & deliverability — reports, suppression vs undeliverable, hygiene

Everything about reading what happened to your mail and managing bad addresses. Read it before interpreting a report or badge, and **especially** before answering anything about the suppression list vs the undeliverable list — they are different things and conflating them misleads users badly.

## Report surfaces

All dashboard reports live under the reports area and read from **ClickHouse materialized views** (pre-aggregated), gated by `analytics.view` unless noted.

- **Deliverability** — overview KPIs (sends, deliveries, hard+soft bounces, complaints, rejects, delivery delays, rendering failures, unique opens, unique clicks); **bounce-reason** breakdown (by bounce type/sub-type); **complaint-reason** breakdown; **ISP breakdown** (per-provider sends/deliveries/bounces vs the prior period — top 8 providers, rest collapsed to "Other"); **delivery latency** (p50/p95/p99); **reputation** (read **live** from SES, not ClickHouse); **suppression list** (read **live** from SES, 60s Redis cache).
- **Messages** — paginated list of distinct messages with terminal status, sender, recipient, subject, channel, template. Per-message **event timeline** (every event for one message, opens/clicks enriched with geo).
- **Events** — flat event stream, filterable by type/channel, **exportable** as CSV/JSON (up to 10,000 rows).
- **Engagement** — open/click time-series, by-domain breakdown, by-template breakdown + per-template performance table.
- **Public API v1** (read-only, scope `api.reports.view`): `GET /v1/reports/deliverability`, `/v1/reports/engagement`, `/v1/reports/template-performance` (bucketed time-series).

> **Reputation and the suppression list are read live from SES**, so they reflect AWS right now. Everything else is from materialized views and reflects ingested events (which arrive via EventBridge moments after they happen).

## ⚠️ Suppression list vs Undeliverable list (the core distinction)

These answer **different questions** and live in **different places**.

|  | **Suppression list** | **Undeliverable list** |
| --- | --- | --- |
| Whose list is it? | **AWS SES's** — SendOps mirrors it | **SendOps's own** derived view |
| Where it lives | In SES (SendOps reads it, 60s cache) | ClickHouse aggregate + Postgres exclusions/rules |
| What populates it | SES auto-adds on **hard bounce / complaint** | SendOps **classification rules** over bounce/complaint **history** |
| Question it answers | "Who will SES **refuse to send to** right now?" | "Who has a **history** that says they're a bad target, per **my** policy?" |
| Reasons | `bounce`, `complaint`, `manual` | your rules (permanent bounce, complaint, reject, repeated transient, …) |
| Removing someone | `DELETE` → calls SES `DeleteSuppressedDestination` (**live SES write**) | `POST` an **exclusion** (Postgres only; does **nothing** to SES) |

**Plain-English version for users:** the **suppression list is the hard gate** — if an address is on it, SES itself blocks the send, full stop. The **undeliverable list is SendOps's advisory** — "based on your rules and this address's bounce/complaint history, you probably shouldn't email it" — but it does **not** by itself stop a send. They overlap (a hard-bounced address is usually on both) but they are not the same list, and clearing one does **not** clear the other.

### Suppression list details

- View: `GET /api/v1/orgs/{slug}/reports/deliverability/suppression` (perm `suppression.view`). Public API read-only: `GET /v1/suppressions`, `/v1/suppressions/{email}` (scope `api.suppressions.view`).
- Remove (re-allow): `DELETE .../suppression/{email}` (perm `suppression.manage`) — this is a **direct write to SES**; afterward SES will send to that address again. "Not found" is treated as success (idempotent).
- There is **no "add to suppression" API** — SES adds entries automatically. (You add to *undeliverable* indirectly via history; you don't manually suppress in SendOps.)

### Undeliverable list details

- View: `GET /api/v1/orgs/{slug}/undeliverable/` (perm `undeliverable.view`). Public API read-only: `GET /v1/undeliverable` (scope `api.undeliverable.view`).
- An address shows as `listed` (rules say undeliverable) or `excluded` (an operator cleared it).
- Built from a 365-day ClickHouse aggregate of bounce/complaint/reject events, reconciled with Postgres "exclusion" rows at query time.

## Classification rules (locked + tunable)

The undeliverable list is computed from a per-org rule set. Three rules are **locked** (always on, can't be disabled); three are **configurable**.

**Locked (fire on the first qualifying event, ever):**

| Rule | Fires on |
| --- | --- |
| `permanent_bounce` | any **Permanent** (hard) bounce |
| `complaint` | any complaint |
| `rejected` | any reject |

**Configurable (windowed — enable/disable, set `events` 1–100 and `window_days` 1–365):**

| Rule | Fires on |
| --- | --- |
| `repeated_transient` | N general **Transient** (soft) bounces within the window |
| `undetermined` | N **Undetermined** bounces within the window |
| `soft_bounce_accumulation` | N soft bounces of type MailboxFull / MessageTooLarge / ContentRejected / AttachmentRejected within the window |

(Capacity-type soft bounces — `ChannelLimitExceeded` — are deliberately **excluded** from accumulation; they're a sender-side issue, not a bad recipient.)

**Three presets:**

- **strict** — locked rules only.
- **standard** (default for new orgs) — locked rules + `repeated_transient` (3 events / 30 days).
- **aggressive** — all six enabled.
- Anything else reports as **custom**.

**Tuning:**

- `PUT /api/v1/orgs/{slug}/undeliverable/rules` (perm `undeliverable.configure`) — send the full rule object.
- `POST .../undeliverable/rules/preview` (perm `undeliverable.view`) — dry-run; returns current vs candidate counts and a diff (added/removed samples) **without persisting**. Always preview before changing rules so the user sees who flips.
- The rule set has a short **version** hash; the list response carries `X-SendOps-Rules-Version` so the UI can detect mid-session changes.

## Allow / re-list behavior

- **Undeliverable (SendOps side):**
  - Clear an address: `POST .../undeliverable/exclusions` `{email, reason?}` (perm `undeliverable.manage`). Writes a Postgres exclusion. **Does nothing to SES.**
  - Re-list (undo): `DELETE .../undeliverable/exclusions/{email}`. Removes the exclusion; the address re-surfaces if it still has qualifying events.
  - **Auto re-list:** if a cleared address bounces/complains **again** (a new event after the exclusion timestamp), SendOps re-lists it automatically on the next query — no operator action.
- **Suppression (SES side):** `DELETE .../suppression/{email}` — the only way to truly let SES send again.

> A frequent confusion: "I removed them from undeliverable but mail still bounces / still won't send." Clearing **undeliverable** is advisory and doesn't touch SES. If SES is blocking, the address is on the **suppression** list — that's a separate `DELETE`. And if the underlying address is genuinely dead, re-listing/auto-suppression will just happen again; the fix is data hygiene, not repeated un-suppression.

## Bounce & complaint hygiene — the event flow

Events reach SendOps from **AWS EventBridge** (not SNS): the customer's CloudFormation stack installs an EventBridge rule that POSTs SES events to SendOps's ingest endpoint. The pipeline: API-key → resolve account/org → parse → attribute to the owning org → resolve channel → **dedup** (24h) → buffer → batch-write to ClickHouse → `202 Accepted`.

Bounce taxonomy (verbatim from SES):

- **Hard bounce** = `Permanent` (sub-types like `NoEmail`, `Suppressed`, `OnAccountSuppressionList`). SES **auto-suppresses** these.
- **Soft bounce** = `Transient` (`General`, `MailboxFull`, `MessageTooLarge`, `ContentRejected`, `AttachmentRejected`). SES does **not** auto-suppress.
- **Undetermined** = SES couldn't classify. Not auto-suppressed.
- **Complaint** = recipient marked spam. SES auto-suppresses with reason `complaint`.

Hygiene guidance to give users:

- Hard bounces and complaints are **already** handled by SES suppression — don't keep mailing them. The undeliverable rules then surface the **pattern** (e.g. an address softly bouncing repeatedly) before it becomes a hard problem.
- Watch the **complaint rate** and **hard-bounce rate** — high values threaten SES reputation and production access. The deliverability report's reputation panel reads these live from SES.
- A spike in the dedup rate (duplicate inbound events) can fire a notification — usually a stack/config issue, not a recipient problem.

## The search permission gotcha

Searching reports/messages **by a specific recipient email** is gated by a **separate, narrower permission** than viewing reports:

- Dashboard: recipient search is `analytics.search.recipients` — **not** `analytics.view`. A user who can see aggregate reports may still be unable to search by recipient.
- Public API: the `recipient=` filter on `GET /v1/messages` and `GET /v1/recipients/{email}/messages` require the `api.messages.unmask_recipients` scope, which is only grantable by someone who holds `analytics.search.recipients`. Without it, recipient emails are **masked** and the recipient filter is unavailable.

This is deliberate PII protection. If a user says "I can see the reports but can't search for jane@example.com," the answer is "you need the `analytics.search.recipients` permission (or, for the API key, the `api.messages.unmask_recipients` scope)" — not a bug.

## Refresh behavior — reports are not live

Report data is fetched **on page load / on demand**, not streamed.

- There's a **manual refresh** button. An **optional auto-refresh** interval can be set (default off).
- Auto-refresh **pauses** when the tab is hidden, after ~10 min of inactivity, or after 3 consecutive failures (circuit breaker).
- Changing the date range or channel filter marks data stale and resets the countdown.

So "my numbers look old": if auto-refresh is off (the default), they're a snapshot from last load — hit refresh. Underlying event ingestion itself is near-real-time (events land seconds after they occur); the staleness is the report view, not the data.

## Quick interpretation cheats

- **Opens/clicks are zero but deliveries look fine** → tracking is off on that channel, or the pipeline has no event destination. See § Reference: Setup & onboarding → "tracking off = no event destination" trap.
- **"Why won't it send to this address?"** → check the **suppression** list (SES gate), not undeliverable.
- **"This address keeps coming back on the undeliverable list after I clear it"** → it's still bouncing; auto re-list is working as designed. Fix the data.
- **"Reputation looks bad / production access stuck"** → check hard-bounce + complaint rates in the reputation panel (live from SES).

---

# Reference: Templates — git source of truth, manifest, Handlebars, namespacing

Everything about email templates in SendOps. Read it before answering anything about the connected repo, the manifest, Handlebars, deploying to SES, test sends, or exporting existing templates.

## Code-first — there is no in-app editor

SendOps is **code-first**. The in-app template editor was **removed**. Templates are authored as code (React Email TSX rendered to HTML by CI, or plain `.hbs` files), committed to a connected **GitHub repo**, and synced into the customer's SES account. The dashboard **displays** templates (preview, validation status, git metadata, test-data profiles, test-send) but offers **no editing**.

So for "how do I change a template," the answer is always: **edit the file in the repo and push** — never "click edit in SendOps."

## Git as the source of truth

One **GitHub connection per org**. As of the latest release it lives under **Connections › GitHub** (it used to be under Templates). The same connection serves **both** templates **and** git-backed audience definitions (segments/attributes) — see § Reference: Lists & Segments.

Connection knobs that matter:

- **`path_prefix`** — scopes the connection to a subfolder. Empty = repo root. If set (e.g. `sendops/`), the manifest is read from `<path_prefix>/sendops.json` and all template paths are relative to it. Useful for monorepos.
- Auth is via a **GitHub App** installation (the user installs the app and authorizes the repo). If a user is stuck connecting, it's usually the GitHub App install/authorization step — point them to Connections › GitHub.

## The manifest — `sendops.json`

The repo declares its templates in **`sendops.json`** (at the repo root, or under `path_prefix`). Max 64 KiB. Shape:

```json
{
  "templates": {
    "welcome": {
      "path": "templates/welcome.hbs",
      "testdata": "fixtures/welcome/",
      "subject": "Welcome to {{productName}}"
    }
  },
  "images": ["static/logo.png"],
  "segments": { "high-value": { "path": "audience/high-value.cel", "name": "High value" } },
  "attributes": "audience/attributes.json"
}
```

Per template entry:

- **`path`** (required) — repo-relative path to the `.hbs` file. No `..` traversal.
- **`testdata`** (optional) — path to test data. Must end in `/` (a directory of `.json` files) or `.json` (a single file). Each file is one profile (a JSON object), or an array whose elements each carry a `_name`.
- **`subject`** (optional) — the SES `Subject`; may contain Handlebars placeholders. If omitted, SES Subject falls back to the template name.
- **`images`** (optional, top-level) — repo-relative images promoted to the Edge CDN.

The map **key** (`"welcome"`) is the **logical template name** — the display name and the base of the SES template name.

> If `sendops.json` is absent, SendOps falls back to **scanning** the legacy `template_path` directory for `.html` / `.hbs` / `.handlebars` files. Prefer an explicit manifest; tell users to add one.

## Handlebars — what's supported

Template files are Handlebars (`.hbs`, also `.html` / `.handlebars`). Supported syntax:

- `{{variable}}` (HTML-escaped) and `{{{variable}}}` (raw/unescaped).
- Block helpers — **exactly four**: `{{#if}}`, `{{#unless}}`, `{{#each}}`, `{{#with}}` (each with optional `{{else}}`).
- Partials `{{> name}}` (parsed; not resolved locally).
- Comments `{{! ... }}` / `{{!-- ... --}}`.

> **Any other block helper is a validation error** (`unknown_helper`). If a user wrote `{{#formatDate}}` or some custom helper, it will fail — they must precompute that value before render.

SES-layer validation also enforces: ≤ 500 KB, valid UTF-8, and at least one HTML tag after stripping Handlebars.

> **Block-helper survival (React Email authors):** if you author in React Email TSX, Handlebars block helpers like `{{#if}}` must survive the TSX→HTML CI render into the `.hbs` (authored directly or via an escape pattern). React Email's HTML renderer does not emit Handlebars; the pipeline stores the rendered `.hbs` source as-is and does not re-render JSX at deploy time.

## Per-org SES namespacing — `{ses_namespace}--{name}`

When a template deploys to SES, its SES name is **`{ses_namespace}--{logical_name}`** (e.g. org namespace `acme` + template `welcome` → SES template `acme--welcome`). Slashes become `--`; characters outside `[A-Za-z0-9._-]` are stripped.

- `ses_namespace` is derived from the org slug **at creation** and is **immutable thereafter** — deployed names are a customer-facing contract that must survive slug renames.
- Why: multiple orgs can share one AWS account (see § Reference: Setup & onboarding); namespacing prevents two orgs' `welcome` templates from colliding in SES's flat per-account namespace.

So when a user looks in the SES console and sees `acme--welcome` instead of `welcome`, that's correct and expected — not a bug. They should reference the **logical** name in SendOps; the namespacing is applied automatically on deploy.

## Test sends

From the template detail page (Preview tab renders in an iframe; Send tab sends a real email).

- The test goes through **SendOps's own SES account** (from `noreply@sendops.dev`), **not** the customer's SES — so a test send works even before the customer leaves their sandbox.
- The subject is prefixed with `[TEST] `.
- A send needs **test data**: either a stored **profile** (git-synced from the manifest `testdata`, or created in the UI) or **inline JSON**.
- Rate-limited per user and per org (hourly + daily) and per recipient; exceeding a limit returns 429.
- Every attempt is logged.

## Deploy & the transform pipeline

On sync, each template is validated then run through a **deploy-time transform pipeline** before being pushed to SES. The pipeline runs in **two phases**:

**Phase 1 — name transform** (runs first; fails closed and aborts the whole deploy if it fails):

1. **Namespacing** — rewrites the SES **template name** to `{ses_namespace}--{name}`. Touches only the name, not the body.

**Phase 2 — content transforms** (run in order, each rewriting the **HTML body**):

2. **Asset resolution** — rewrites `{{asset "path"}}` and repo-relative `src="..."` to content-addressed CDN URLs (only if the org has an asset map + CDN; fails open — unresolved refs pass through).
3. **Unsubscribe footer** — injects an unsubscribe link before `</body>` (only if the org configured an unsubscribe base URL; idempotent).
4. **Postal address** — injects a CAN-SPAM/CASL postal address (only if configured; idempotent).

If the name phase fails, **no content transforms run**. After each **content** transform the output is **re-validated** — a transform that produces invalid HTML/Handlebars fails the deploy. The result is stored as a **second artifact** (`deployed_content`) distinct from the raw repo source, so the dashboard can show **Deployed** and **Diff** tabs (what the repo says vs. the exact bytes sent to SES). A rebuild job can re-run the pipeline when SendOps-side config changes (e.g. the org sets a postal address) **without** a new git commit.

So if a user asks "why does the deployed email have a footer / different image URLs than my source," that's the transform pipeline — point them at the Deployed/Diff tabs.

## Brownfield export (existing SES templates → git)

For customers who already have templates in SES, SendOps **exports** them into a ready-to-commit zip (this is the migration path, related to **SND-769**):

- Lists the org's SES templates and fetches each one's content (rate-limited).
- Produces a zip:
  ```
  sendops-templates/
    sendops.json          ← manifest (path-only entries)
    templates/<name>.hbs  ← HTML body (text-only templates wrapped in <pre>)
    templates/<name>.txt  ← text body (when both exist)
    .gitignore
    README.md
  ```
- Names are sanitized (lowercased, hyphenated; collisions get `-2`, `-3`).
- The generated manifest fills **only `path`** — `testdata` and `subject` are **not** backfilled. It's a starting point, not a finished manifest; the user fleshes it out.

The end-to-end migration: **export → unzip → commit to a GitHub repo → connect the repo → SendOps syncs it back and deploys** (now with namespacing + transforms applied).

## SendOps dogfoods this

SendOps itself is the first customer of the code-first pipeline: its own 40+ transactional templates are authored as React Email TSX, rendered by CI, and committed as artifacts in the product's own monorepo. Mentioning this is a good trust signal — "we ship our own mail this exact way."

## Quick navigation cheats

- Connect/repair the repo: **Connections › GitHub**.
- Change a template: edit the file in the repo + push (no in-app editor).
- See what actually shipped to SES: template detail → **Deployed** / **Diff** tabs.
- Send yourself a test: template detail → **Send test** (uses SendOps's SES; works in sandbox).
- Migrate existing SES templates in: use **export**, commit the zip, connect the repo.
