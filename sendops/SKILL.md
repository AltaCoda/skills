---
name: sendops
description: >-
  Your in-agent guide to SendOps (sendops.dev), the email-infrastructure control
  plane on AWS SES. Use this skill whenever the user works with SendOps or wants
  help with: connecting an AWS/SES account, verifying domains and DKIM, leaving
  the SES sandbox (production access), building Contact Lists or Segments,
  writing SendQL segment predicates, defining contact attributes, building Drip
  Workflows / automated journeys in the `.flow` (SendFlow) language, reading
  deliverability / engagement / per-message reports, the undeliverable list vs.
  the suppression list, bounce and complaint hygiene, open/click tracking, or
  git-synced email templates. Trigger it even when the user never says "SendOps"
  by name — "why aren't my opens being tracked", "segment trial users who
  haven't opened in 30 days", "send a welcome drip when someone signs up", "my
  domain won't verify", or "suppression list vs undeliverable list" all mean
  this skill should be consulted. It makes a complex product feel simple:
  explains the concepts, drafts the exact SendQL, lists, attribute schemas, and
  `.flow` workflows, interprets reports, and says exactly where to click.
---

# SendOps Co-pilot

SendOps ([sendops.dev](https://sendops.dev)) is an **email-infrastructure control plane** on AWS SES. It sets up SES, EventBridge, and supporting resources **inside the customer's own AWS account** (via CloudFormation) and then manages templates, audiences, reporting, deliverability, and analytics on top. **It is not a proxy** — email always sends through the customer's SES directly. Positioning: *"Send more. Pay the same."*

This skill exists because SendOps has grown powerful enough that a normal user can feel lost — between SES setup, Lists and Segments, reports, and templates there's a lot of surface area. Your job is to be the calm expert sitting next to them: translate what they want into the right SendOps concept, hand them the exact thing to create (a SendQL predicate, a list, an attribute, a DNS record), and point them to the precise screen or endpoint. You make the product feel small.

This skill is a **single self-contained file**: a short router (this part) followed by five in-depth reference sections (Setup & onboarding, Lists & Segments, Drip Workflows, Reports & deliverability, Templates). Everything you need is below — read the matching section before giving detailed guidance.

## What this skill does and doesn't do

This is a **guidance skill**. It does not call the SendOps API, hold credentials, or change anything in the user's account on its own. It helps the user *think, draft, and navigate*:

- **Explain** any SendOps concept in plain terms and how it maps to AWS SES underneath.
- **Draft** the exact artifact the user needs — a SendQL segment predicate, a List definition, an attribute-registry entry, a template manifest snippet, DNS records to paste — ready for them to apply.
- **Interpret** what a report or a status badge is telling them, and what to do about it.
- **Navigate** — tell them which page, tab, button, or endpoint to use.

When something requires an action in SendOps, say so explicitly and give the path. Don't imply you performed it. Typical phrasing: *"Here's the predicate — paste it into the Segment editor (Audience → Segments → New segment), save it via `POST /api/v1/orgs/{slug}/segments`, or commit it to your connected repo as a `.sendql` file."* If the user has the SendOps Public API or its MCP server connected as a tool in this session, you may additionally read live data through it — but never assume that; the baseline is advice, not execution.

## How to help — the loop

1. **Locate the request on the product map** (below). Most confusion is really "which of these five areas am I in?" Name it for the user.
2. **Read the matching reference section below** before answering anything non-trivial — the details (SendQL grammar, classification rules, sync lifecycle) are exact and easy to get subtly wrong from memory.
3. **Hand over a concrete artifact**, not just prose. A user who asked "how do I segment dormant trial users" should leave with a predicate they can paste.
4. **Say where it goes and what happens next** (which screen / endpoint; whether it's view-only; whether a sync or refresh is needed).
5. **Flag the gotchas** that bite people — absence handling in SendQL, tracking that isn't actually on, suppression vs. undeliverable, sandbox limits.

## Product map — the five areas

Route the user to the right reference section below. Read the section before giving detailed guidance.

| If the user is asking about… | Area | Read |
| --- | --- | --- |
| Connecting AWS, the CloudFormation stack, verifying a domain/DKIM, getting out of the SES sandbox (production access), importing an existing ("brownfield") SES setup, channels = config sets, why a config set shows "tracking off" | **Setup & onboarding** | § Reference: Setup & onboarding |
| Contact Lists, Segments, SendQL predicates, the attribute registry, who's in an audience and why, preview/dry-run, managed vs. git-backed definitions | **Lists & Segments** | § Reference: Lists & Segments |
| Drip Workflows, automated journeys/"drips", the `.flow` (SendFlow) language, enroll/trigger a contact, wait/branch/split, send steps, enrollment scope, re-entry, per-workflow frequency cap, manual send approval | **Drip Workflows** | § Reference: Drip Workflows |
| Deliverability / engagement / per-message / template reports, opens & clicks, ISP breakdown, the undeliverable list, the suppression list, bounces & complaints | **Reports & deliverability** | § Reference: Reports & deliverability |
| Email templates, the connected Git repo, the manifest, Handlebars, test sends, deploying templates to SES, exporting existing SES templates | **Templates** | § Reference: Templates |

If a request spans areas (common — e.g. "segment everyone who bounced, then stop emailing them"), name each area and read each section you need.

## Core vocabulary (so you never mislead the user)

- **Control plane, not proxy.** SendOps configures the customer's AWS account and reports on it. Sending happens via their SES. If SES is down or in sandbox, SendOps can't send around that.
- **Channel = SES configuration set**, exactly one-to-one. "Set up a channel" means "this maps to a SES config set."
- **AWS account is shared-capable.** Several SendOps orgs can link one AWS account: the first provisions the CloudFormation stack; later orgs *join* with no deploy. An identity or config set is "active" in at most one org at a time.
- **List ≠ Segment.** A **List** is membership you *assign* (static, explicit add/remove). A **Segment** is membership you *describe* with a SendQL predicate (dynamic; contacts flow in and out as their data changes). This single distinction drives everything downstream — see the Lists & Segments reference.
- **Membership ≠ mailability.** Being in a List or Segment does not mean SendOps will email someone. Consent and suppression are enforced separately, at send time.
- **Undeliverable list ≠ Suppression list.** The **suppression list** mirrors the AWS SES suppression list. The **undeliverable list** is SendOps's own derived view based on configurable rules over bounce/complaint history. They answer different questions — see the reports reference.
- **Git is the source of truth for templates** (and, when a repo is connected, for segment/attribute *definitions* too). Data — list membership, attribute values — never lives in git; it lives in SendOps.
- **The audience UI is writable — the surface differs by object.** **Lists** are fully managed in the dashboard: create, edit, delete, and add/remove members (permission `lists.manage`). **Segments** have a full in-app editor (`segments.manage`): create and edit the SendQL definition with live validation and preview, pause/resume, and version history — or author via the API / a connected GitHub repo instead. Git-backed segments are read-only in the app (edit them in the repo). **Attribute definitions** are managed via the API (writable) or git, not a dashboard form. (Verify against their build if they say the UI differs; products move.)
- **Reports refresh manually**, not live. If numbers look stale, the user may need to hit refresh.

## Tone

Be the friendly expert. Lead with the answer, then the artifact, then where it goes. Prefer a worked example over abstract description. When you hit a genuine limitation of the current release (no v1 write endpoint for lists/segments, no dedicated-IP management), say so plainly and give the real path forward rather than pretending. Keep AWS/SES accuracy high — users trust this skill precisely because it tells them the truth about what's happening under the hood.

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

What the CloudFormation stack provisions (current template **version 13** — 7 resources; v13 added the `ses:CreateContactList` permission so SendOps can provision the org's contact list):

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

> The current required CloudFormation **template version is 13**. If the user later tries to request production access on an older stack, SendOps returns `CF_TEMPLATE_OUTDATED` — they must update the stack first. If you see that error, the fix is "update your CloudFormation stack to the latest SendOps template," not anything in the form.

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
- The request form validates: **mail type** is `TRANSACTIONAL` or `PROMOTIONAL`; **website URL** is valid http(s); **use-case description** is **≥ 50 characters**; no other open request; and the **CloudFormation template is current (v13)**.
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

> **The names in the table are the logical channel names.** The actual SES **configuration-set name is namespaced per workspace** as `sesmail-{ses_namespace}--{slug}` — e.g. the marketing channel for org namespace `acme` is the SES config set `sesmail-acme--marketing`. The namespace comes from the org and prevents two orgs that share one AWS account from colliding on SES's flat per-account config-set namespace (same reasoning as template namespacing). So if a user sees `sesmail-acme--transactional` in the SES console, that's the `transactional` channel — correct and expected, not a duplicate.

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

# Reference: Lists & Segments — audiences, SendQL, attributes

Everything about building audiences in SendOps. Read it before drafting a List, a Segment predicate, or an attribute schema — the SendQL grammar and the absence contract are exact and very easy to get subtly wrong from memory.

## The one distinction that drives everything

- **List** = membership you **assign**. Static. You explicitly add/remove contacts; a contact stays in until removed (or deleted). Think "imported CSV," "beta testers," "everyone from the conference."
- **Segment** = membership you **describe**. Dynamic. You write **one SendQL predicate**; a contact is a member iff the predicate is `TRUE` for them **right now**. As their data changes, they flow in and out automatically.

Both record membership as an **append-only event log** (no derived "current members" table). "Who's in it now" is computed at read time as the latest event per contact:

- Lists: latest `added`/`removed` event wins.
- Segments: latest `entered`/`exited` transition wins.

This is why **membership over time** is a first-class thing: SendOps keeps the full transition history, so future features (journeys, broadcasts) can ask "who entered this segment last week."

> **Membership ≠ mailability.** Being in a List/Segment does not mean SendOps will email the contact. Consent (topic preferences, unsubscribe) and suppression are enforced **separately at send time**.

## What you can author in the UI vs. the API

The audience dashboard is **partly writable**, and the surface differs by object:

- **Lists** — full CRUD in the dashboard: create, edit, delete, and add/remove members (permission `lists.manage`). Members can be added by **email** (the contact is auto-created if new) or by existing contact id, with per-row results.
- **Segments** — full authoring in the dashboard (`segments.manage`): a SendQL editor with live validation, diagnostics, dry-run preview, and version history, plus pause/resume. Or author via the API or git instead. Git-backed segments are read-only in the app (the repo is ground truth) and have no pause button; you pause them by removing the source file from the repo.
- **Attribute definitions** — managed via the **API** (now writable, see below) or a connected repo. No dashboard form for the registry.

To author Segments and attribute definitions, the user uses one of:

1. **The dashboard editor** (Segments only) — Audience → Segments → New segment.
2. **The session API** — `POST/PUT/DELETE /api/v1/orgs/{slug}/segments` (and the lists/attributes equivalents). Full CRUD; dashboard-token auth.
3. **A connected GitHub repo** — definitions live as files; SendOps syncs them. (See "Managed vs git-backed.")

> **Public API v1** (`/v1/...`, API-key auth) — read vs. write differs by object:
> - **Lists & Segments are read-only**: `GET /v1/lists` (+ `/{id}`, `/{id}/members`), `GET /v1/segments` (+ `/{id}`, `/{id}/members`), plus the read-class `POST /v1/segments/preview`. There is **no** v1 write endpoint — "create a segment via the public API" is not possible today; direct them to the session API or git.
> - **Contacts are full read + write** (SND-907) — the contact-roster write surface. Reads (`GET /v1/contacts`, `GET /v1/contacts/{email}`) use `api.contacts.view`; writes (`POST /v1/contacts`, `PUT`/`PATCH`/`DELETE /v1/contacts/{email}`, and async bulk `POST /v1/contacts/bulk` → `202` + `job_id`, polled via `GET /v1/contacts/bulk/{job_id}`) use scope **`api.contacts.manage`**. Contacts are keyed by **email** (URL-encode `@` as `%40`); `POST` creates, `PUT` replaces (omitted fields cleared), `PATCH` diff-merges (a `null` attribute clears that key), `DELETE` archives (idempotent). Writes accept an **`Idempotency-Key`** header (24 h replay) and can manage **static-List** membership inline via a `lists` directive (`add`/`remove`/`set` on POST/PATCH, an exact array on PUT). Attribute values are validated against the registry — an unknown key or type mismatch is `422 attribute_validation_failed`; in bulk that fails only the offending row, not the batch.
> - **Attribute definitions are writable** — the first v1 write surface (SND-906): `POST /v1/attributes`, `PUT /v1/attributes/{id}`, `DELETE /v1/attributes/{id}`, and the zero-write impact preview `POST /v1/attributes/{id}/preview`, all under scope **`api.attributes.manage`** (reads use `api.attributes.view`).
> - **Broadcasts are full read + write** — you can create and send a broadcast over the API. Reads (`GET /v1/broadcasts` (+ `/{id}`, `/{id}/results`) and `POST /v1/broadcasts/{id}/preview`) use `api.broadcasts.view`; the writes — `POST /v1/broadcasts` (create, in template or inline-HTML mode), `POST /v1/broadcasts/{id}/send` (send now or schedule), `POST /v1/broadcasts/{id}/test` (test send), `DELETE /v1/broadcasts/{id}` — use scope **`api.broadcasts.manage`**. Consent and suppression are enforced at send time. (Edit/reschedule/cancel-after-send remain dashboard-only.)
> - **Activities are read + write** — the custom-behavior ingest surface. Ingest via `POST /v1/activities` (single or a batch of ≤ 1000 events) under scope **`api.activities.write`** — a write-only append (unknown identities get a stub contact, which carries no consent); reads (`GET /v1/activities`, `GET /v1/contacts/{id}/activities`, `.../activities/summary`) use `api.activities.view`. Ingest can carry an idempotency key and a `dedup_mode` (`retry` vs `once`) to control whether a repeated signal is collapsed — but the real duplicate-send guard is a workflow's re-entry policy, not this (see § Reference: Drip Workflows).
> - **Workflows are read + dry-run only**: `GET /v1/workflows` (+ `/{id}`, `/{id}/runs`, `/exits`, `/funnel`, `/runs/{runId}/timeline`) and the simulate-only `POST /v1/workflows/{id}/dry-run`, all under scope **`api.workflows.view`**. There is **deliberately no workflows *manage* scope** — you cannot create or edit a workflow over the public API; author it in the dashboard or git. (The `.flow` source itself isn't exposed over the API — metadata, run counts, and analytics only.)
> - **Assets are read-only**: `GET /v1/assets` (+ `/{id}`) under scope **`api.assets.view`** — the org's git-sourced image library (source path, CDN URL, content hash, dimensions). Assets are managed via the connected git repo, not the API.

So your job for any "I want an audience of X" request is to **hand them the artifact** (a SendQL predicate, or a list definition) and tell them which authoring path to use.

## The SendQL language (the exact surface)

Segment predicates are written in **SendQL**, SendOps's segment-query language. A predicate is one expression that is true or false per contact. Each saved segment is stamped with a **profile version** (currently `1`).

### Structure

Terms combine with `and`, `or`, `not`, and parentheses. Precedence, loosest to tightest: `or` → `and` → `not` — so `a or b and c` means `a or (b and c)`; use parentheses when in doubt.

### `attr.<name>` — your registered attributes

Per-org custom attributes from the **attribute registry** (see below). Names match `^[a-z][a-z0-9_]{0,62}$`; always written with the `attr.` prefix, so registry names can never collide with keywords.

| Form | Example |
| --- | --- |
| Comparison | `attr.score >= 10` (`=`, `!=`, `<`, `<=`, `>`, `>=`) |
| In a set | `attr.country in ["US", "CA", "MX"]` |
| String match | `attr.email ends with "@acme.io"`, `attr.name starts with "A"`, `attr.title contains "VP"` |
| Presence | `has attr.company`, or `attr.tier is known` / `attr.tier is unknown` |
| Age (date attrs) | `now - attr.signup_date > 7d`, `now - attr.signup_date between 3d and 14d` |

Typed by the registry; both sides of a comparison must be the same type (`attr.score > "ten"` is rejected at validation). **Absent attributes are unknown — read the absence contract below before relying on this.**

### Event terms — first-class engagement history

SendQL selects on the **raw event stream** (no fixed rollups). Events: `send`, `delivery`, `open`, `click`, `bounce`, `complaint`, `reject`, `delivery_delay`. Four shapes:

| Shape | Meaning | Example |
| --- | --- | --- |
| `count(<event>) <op> N` | how many times it happened | `count(open within 30d) >= 3` |
| `exists(<event>)` | at least once | `exists(click within 7d)` |
| `sum\|avg\|min\|max(<field> of <event>) <op> N` | aggregate a numeric event field | `sum(processing_ms of delivery within 7d) < 10000` |
| `last\|first(<event>) <op> <time>` | most recent / earliest occurrence | `last(open) < now - 14d` |

Narrow any event with:

- `where <condition>` — filter on event properties, e.g. `click where url contains "/pricing"` (properties include `subject`, `template`, `sender_domain`, `url` on click, `type`/`sub_type` on bounce).
- `within <duration>` — rolling window, e.g. `open within 30d`.
- `between <date> and <date>` — absolute window.

### Consent and membership terms

| Term | Selects contacts who… |
| --- | --- |
| `subscribed to "<topic>"` | are opted in to a named topic |
| `opted out of "<topic>"` | have opted out of a named topic |
| `unsubscribed from all` | are globally unsubscribed |
| `suppressed` | are on the suppression list |
| `in list "<key>"` | are a member of a named List (by its stable key) |
| `in segment "<key>"` | are a member of another Segment |

### Durations, dates, and `now`

- Durations: number + unit, no space — `30s`, `15m`, `24h`, `7d`, `2w`, `6mo`, `1y`.
- Dates: `YYYY-MM-DD`.
- `now` is evaluation time; `now - 14d` is a point in the past. **Any time-relative term (`now`, `within`, `last`/`first`, age) makes the segment "sweep-class"** (see eval classes).

### Reserved words

Keywords can't be bare event/property names: `all and avg between contains count ends exists first from has in is known last list max min not now of opted or out segment starts subscribed sum suppressed to true false unknown unsubscribed where within with`. Attribute names are always safe behind the `attr.` prefix.

### Size governors (hard limits — exceeding any is a 422)

- ≤ 200 AST nodes total
- ≤ 20 nesting depth
- ≤ 100 elements in any `in [...]`
- ≤ 8 event terms
- ≤ 8 `in list` / `in segment` terms

If a predicate is rejected for size, simplify (fewer OR branches, smaller `in` lists, or split into two segments).

## ⚠️ The absence footgun (read this twice)

A contact is a member of a segment **iff the predicate evaluates to TRUE — unknown and false both exclude.** A missing attribute is **unknown**, not a default value; a never-occurred event has **no** `last`/`first` time.

Concretely:

- `attr.tier != "free"` — a contact with **no `tier` attribute at all** is unknown → **excluded**. Even though "not free" sounds like it should include them. This surprises everyone.
- `attr.score > 50` — contacts **without** a `score` are excluded (they are **not** treated as 0).
- `last(open) < now - 14d` — a contact who **never opened** is excluded (their last-open isn't "long ago", it's unknown). To include never-openers, add `or count(open) = 0`.

Counts are the safe exception: `count(open within 30d)` is `0` for a contact with no events, so `count(open) = 0` genuinely matches never-openers.

**How to write predicates that mean what the user wants:**

| User intent | Wrong (silently drops the unknowns) | Right |
| --- | --- | --- |
| "everyone who isn't on the free plan, including unknowns" | `attr.tier != "free"` | `attr.tier != "free" or attr.tier is unknown` |
| "anyone whose score is over 50" | — | `attr.score > 50` (explicitly excludes no-score contacts — say so) |
| "people with no tier set" | — | `attr.tier is unknown` |
| "went quiet — no open in 30 days, ever-openers only" | — | `count(open) > 0 and last(open) < now - 30d` |

The editor's **absence lint** flags exactly these spots with a non-blocking warning and the guard to add. When you draft any predicate with a `!=`, a `not`, an age term, or `last`/`first`, **proactively decide and state** how contacts missing that data should be treated.

## Attribute registry

Custom attributes must be **registered** before use. An attribute has:

- `name` — `^[a-z][a-z0-9_]{0,62}$`, not a reserved root.
- `type` — one of `string`, `number`, `bool`, `datetime`, `enum`.
- `enum_values` — required when type is `enum`; the allowed set is enforced **at write time**, not in predicates.
- `description` — optional.

Rules:

- **The registry is mutable and authoritative — edits always apply, never blocked.** Name, type, and enum changes go through, but each edit is **classified** by its effect: **free** (no data risk — e.g. string⇄enum), **safe-but-stale** (values keep their meaning), or **disruptive** (a rename or a cast-class change that can break predicates). A **disruptive** edit raises a standing per-segment `eval_warning` and triggers best-effort re-evaluation — it does *not* quarantine or roll back. So type is **no longer immutable**; just **preview the impact first** (see below) before a rename or cast-class change. (Git-origin definitions are the exception: the repo is ground truth, so editing one via API returns **409**.)
- Values are written per-contact via `PUT /api/v1/orgs/{slug}/contacts/{id}/attributes`; writes are validated against the registry and trigger incremental re-evaluation of any segment referencing that attribute.
- **Manage the registry via Public API** (scope `api.attributes.manage`): `POST /v1/attributes`, `PUT /v1/attributes/{id}`, `DELETE /v1/attributes/{id}`, and the dry-run `POST /v1/attributes/{id}/preview`. Reads use `api.attributes.view` (`GET /v1/attributes`, `GET /v1/attributes/{id}`). The session API and a connected repo remain valid authoring paths too.

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

The same discipline applies to **attribute-definition edits**: `POST /v1/attributes/{id}/preview` (scope `api.attributes.manage`) returns the edit's **class** (free / safe-but-stale / disruptive) and, for a disruptive change, the affected segments and projected drop-outs — all without writing. Run it before any rename or type change.

## Managed vs git-backed definitions

Every Segment/attribute has an **origin**:

- **`managed`** — created/edited via API or dashboard. Mutable.
- **`git`** — synced from a connected GitHub repo. **Read-only in SendOps** (the repo is the source of truth). Trying to edit/delete a git-backed definition via API returns **409**.

The git layout lives under the repo's manifest (`sendops.json`), alongside templates:

```json
{
  "segments": {
    "high-value": { "path": "audience/high-value.sendql", "name": "High value", "description": "Pro plan, engaged" }
  },
  "attributes": "audience/attributes.json"
}
```

Each `.sendql` file contains **only the predicate string**, e.g. `audience/high-value.sendql`:

```
attr.plan = "pro" and count(open within 30d) > 5
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
- **sweep** — periodically re-evaluated for the whole population (segments using `now`, `within` windows, or `last`/`first` can only be kept fresh by a sweep). The sweep runs roughly every few hours.
- **both** — needs both.

So a "dormant 30 days" segment (time-relative) is sweep-class: membership updates on the sweep cadence, not instantly. If a user expects a time-window segment to update the instant a clock ticks, explain the sweep.

## Copy-paste recipe book

All of these are predicate strings — paste into the Segment editor, a `.sendql` file, or the API body. Adjust attribute names to the user's registry.

**Engaged pro users**
```
attr.plan = "pro" and count(open within 30d) > 0
```

**Dormant — sent to but no open in 30 days (ever-openers only)**
```
count(send within 30d) > 0 and count(open) > 0 and last(open) < now - 30d
```

**Trial users who never engaged**
```
attr.plan = "trial" and count(open within 30d) = 0
```

**Not on the free plan — including contacts with no tier set**
```
attr.tier != "free" or attr.tier is unknown
```

**High score (contacts without a score are excluded)**
```
attr.score >= 80
```

**One of several tiers**
```
attr.tier in ["pro", "enterprise"]
```

**Corporate domain only**
```
attr.email ends with "@acme.com"
```

**Subscribed and not suppressed (mailable-ish — still re-checked at send)**
```
subscribed to "product-updates" and not suppressed
```

**In an existing List and clicked recently**
```
in list "beta-cohort" and count(click within 30d) > 0
```

**Signed up in the last 7 days**
```
now - attr.signup_date < 7d
```

**Clicked the pricing page this week**
```
exists(click where url contains "/pricing" within 7d)
```

When you hand any of these over: (1) confirm the referenced attributes are **registered** and the right **type**, (2) decide the absence behavior explicitly, (3) tell them to **preview** it, then (4) save in the editor / via API, or commit the `.sendql` to the repo.

---

# Reference: Drip Workflows — automated journeys, the .flow language, enrollment, sends & approval

Everything about **Drip Workflows** — automated, multi-step email journeys. Read it before drafting a workflow, explaining enrollment or re-entry, or answering anything about the `.flow` language, send approval, or per-workflow frequency caps. The `.flow` grammar is exact and easy to get subtly wrong from memory.

## What they are

A **Drip Workflow** (a "drip") is an automated journey: a contact **enrolls** on a trigger, then the workflow **waits, branches, and sends** over time — each contact on their own durable timeline — until they **exit**. Welcome sequences, onboarding nudges, cart abandonment, renewal countdowns, win-backs are all drips.

- **Authored in the `.flow` DSL** — a small language called **SendFlow**. It has sequence, branching, bounded loops and waits — everything a journey needs and nothing that would break the auto-laid-out canvas (there is no goto and no unbounded loop, so every workflow renders as a clean top-to-bottom flowchart and provably terminates).
- **Two live views of one definition** — the dashboard **canvas** and the **source editor** edit the same workflow: change either and the other follows. Comments survive the round-trip.
- **Same managed-vs-git model as segments.** A workflow is either **managed** (edited in the dashboard, with version history and restore) or **git-backed** (a `.flow` file in the connected repo, declared under a **`workflows`** key in `sendops.json`). Git-backed workflows are **read-only in the app** — editing one via the API returns **409**; the repo is the source of truth. (Same GitHub connection that serves templates and segment/attribute definitions.)

## The `.flow` shape (compact reference)

A workflow is `workflow "<name>" v1 { <settings> <steps> }`. **Settings come first** (all optional), then the ordered **steps**. Conditions after `where` / `when` / `until` and inside `if` are ordinary **SendQL** predicates (same attributes, events, and `activity.<name>` terms as segments — see § Reference: Lists & Segments), evaluated against the one contact at that moment.

**Settings (before any step):**

| Setting | Meaning |
| --- | --- |
| `enter on <trigger> [where <condition>]` | The one enrollment trigger. `<trigger>` is a **segment** (`segment "trial-started"`), an **event** (`event click`), a custom **activity** (`activity.purchase`), or a **date** relative to a datetime attribute (`3 days before attr.renewal_date`, also `after`). Optional `where` narrows enrollment to contacts who also match a condition. |
| `exit "<name>" when <condition>` | A named exit, checked **before every step**; first match wins. Exit names drive the conversion breakdown on the detail page (e.g. `exit "converted" when attr.plan != "trial"`). |
| `send window <from>-<to> in contact timezone` | Defer any send that comes due outside the window to the next in-window moment, per each contact's timezone (e.g. `9am-6pm`). |
| `reentry once \| on rematch \| per occurrence` | Re-entry policy (default `once`). |
| `frequency cap <N> per <duration>` \| `frequency cap off` | Per-workflow frequency-cap override (e.g. `frequency cap 4 per 7 days`); `off` exempts this workflow. Undeclared → the org-wide default applies. |
| `enroll forward \| existing [since <duration>]` | Enrollment scope on activation (see below). |

**Steps (in order):**

| Statement | Meaning |
| --- | --- |
| `send "<template>" [via topic "<name>"] [transactional]` | Send a deployed template. Bare `send "welcome"` inherits the template's default consent lane; `via topic "product-updates"` sends in the marketing lane under a named topic; `transactional` sends in the topic-exempt lifecycle lane. (`via topic` and `transactional` are mutually exclusive; a topic name is a quoted string, though a single bare word like `via topic onboarding` is also accepted.) |
| `wait <N> <unit>` | Pause for a duration (`wait 2 days`). |
| `wait until <date or attr.<name>>` | Pause until an absolute date or a contact's datetime attribute. |
| `wait up to <N> <unit> until <condition> { timeout: … }` | Wait for a condition with a hard deadline; the optional `timeout:` arm runs if the condition never becomes true before the deadline. |
| `if <condition> { … } [else if … { … }] [else { … }]` | Branch; all arms **rejoin** after the block. |
| `split { 50%: { … } 50%: { … } }` | Stable random cohorts (weights total 100). |
| `hold out <N>%` | The named percentage exits here (a control/holdout group); everyone else continues. |
| `repeat up to <N> every <duration> [until <condition>] { … }` | The only loop — always bounded, with a delay before each pass. |
| `set attr.<name> = <value>` | Write a contact attribute (respects the registered type; an unknown enum value is rejected at validation). |
| `add to list "<key>"` | Add the contact to a static List. |
| `exit` | End the journey here. |

> **Two duration styles.** Workflow-level durations are written in **words** (`wait 2 days`, `every 24 hours`). Inside a condition, SendQL's compact form still applies (`open within 7d`). Don't mix them up.

A worked example:

```text
workflow "Trial onboarding" v1 {
  enter on segment "trial-started"
  exit "converted" when attr.plan != "trial"

  send "welcome" transactional
  wait 2 days
  if not exists(open where template = "welcome") {
    send "welcome-reminder" via topic "onboarding"
    wait 2 days
  }
  send "activation-tips" via topic "onboarding"
  wait up to 7 days until count(activity.login within 7d) >= 2 {
    timeout: send "need-a-hand" via topic "onboarding"
  }
}
```

## Enrollment scope — who joins when you activate

A workflow is **draft** until you **activate** it; only active workflows enroll and step contacts. A trigger enrolls contacts *going forward* — but at activation there may already be contacts who match (people already in the segment, contacts whose `renewal_date` is next week). The `enroll` setting decides whether those are pulled in:

- **`enroll forward`** — only contacts who match from activation onward. Existing matches are left alone.
- **`enroll existing`** — also **back-enroll** contacts who already match (a one-time backfill that runs once at activation). `enroll existing since <duration>` bounds how far back it reaches (e.g. `since 30 days`).

> **The undeclared default depends on the trigger.** Leave `enroll` off and each trigger keeps its natural default: a **segment** trigger **back-enrolls** existing members; **event**, **activity**, and **date-relative** triggers are **forward-only** (there's no sensible "replay every past click"). Forward-only enrollment keys on **when the event actually occurred**, not when SendOps received it — so a late or backfilled historical event, one whose timestamp predates activation, won't enroll a forward-only workflow. A date-relative `enroll existing` **must** carry a `since` window (an unbounded look-back would enroll every contact with any past anchor date).

In the dashboard, `enroll` is a control in the **Flow settings** panel, and the editor previews roughly how many contacts a backfill would enroll (and asks you to confirm) before you activate. When `enroll` is undeclared, the Flow settings card shows the **effective** mode it resolves to — a **Forward only** or **Backfill** badge — so the behavior is visible at a glance.

## Re-entry — can a contact go through twice

By default a contact goes through a workflow **once, ever** (`reentry once`) — a durable, permanent record of "has this contact ever run this workflow," checked every time. To allow re-enrollment after a prior run **finished**, opt into the relaxed mode, which has two spellings that select the **same** behavior and differ only in which trigger reads naturally:

- **`reentry on rematch`** — reads naturally for a **segment** trigger: re-enroll after leaving the trigger segment and matching it again.
- **`reentry per occurrence`** — reads naturally for an **event** or **activity** trigger: re-enroll on a later occurrence.

Two guardrails always apply regardless of mode: **one live journey per contact per workflow** (a contact already moving through won't start a second concurrent journey), and **converts aren't re-enrolled** (a contact who left through a named exit isn't pulled back in while they still match the trigger).

> **`reentry once` — not activity dedup — is the real duplicate-send guard.** On an `activity.<name>` trigger, `reentry once` is what actually stops a duplicate activity (a retried request, a milestone re-emitted by a sync) from producing a duplicate send. Ingest-level `dedup_mode` (see the API inventory) reduces how often a duplicate activity happens at all, but the re-entry policy is the guarantee that a contact isn't emailed twice. Worst case with `reentry once`: a duplicate *timeline row*, never a duplicate *send*.

## Sends, consent & approval

Every `send` step runs the **same send pipeline as a Broadcast** — the customer's SES, the deployed template, per-contact merge data, one-click List-Unsubscribe — just fired when the contact reaches the step rather than all at once. Two safety gates and one operational switch matter:

- **Frequency cap runs first, then consent.** A `send` is checked against the frequency cap *before* consent is evaluated. A capped send is **skipped, not queued** — the journey moves on.
- **Consent is re-checked at send time.** A contact who unsubscribes mid-journey is quietly skipped for that send while their journey continues.

> **`transactional` bypasses the consent filter but is STILL subject to the frequency cap.** This trips people up: the topic-exempt lifecycle lane skips the consent/opt-out gate, but a capped-out contact skips a `transactional` step exactly like a marketing one. If a lifecycle workflow (e.g. onboarding nudges) must never be capped, give it its own `frequency cap off`.

> **A per-workflow `frequency cap` REPLACES the org default — it doesn't add to it.** The workflow's own threshold and window are used instead of the org's. But the **count** being compared is still **cross-workflow**: a send here counts against, and alongside, sends from every other workflow to the same contact — only the threshold/window are workflow-specific.

**Manual send approval** — for workflows where a person should sign off before mail goes out, turn on **Hold sends for approval**. This is an **operational switch**, not part of the `.flow` file — it takes effect immediately, creates no new version, and **works on git-backed workflows too**. While on, every due send enters an **awaiting approval** state and collects in the workflow's **Approvals** panel, where a person acts per row or in bulk:

| Action | Effect |
| --- | --- |
| **Approve** | Send now — still honoring the send window, consent, and frequency cap |
| **Skip** | Don't send this email; the contact's journey continues to the next step |
| **Cancel** | Don't send; end this contact's journey |

Workflows with sends waiting show an amber count badge on the list and the detail header. The per-contact **run timeline** surfaces exactly which step a journey took and **why a send was skipped** (capped, unsubscribed, held).

## Public API — read + dry-run only

The Public API can **read** workflows and **dry-run** a contact through one, but **cannot create or edit** them (there is deliberately no workflows *manage* scope — see the API inventory in § Reference: Lists & Segments). `POST /v1/workflows/{id}/dry-run` evaluates every condition against a real contact so you see exactly which path they'd take — nothing is sent or written. To author or change a workflow, use the dashboard or git.

## Where to click

- **Author / edit:** the **Workflows** area → the workflow's **canvas** and **source** tabs (two views of the same `.flow`).
- **Enrollment scope & frequency cap & approval hold:** the **Flow settings** panel on the workflow.
- **Review held sends:** the workflow's **Approvals** panel.
- **See what happened:** the detail page — run counts by status, the per-node **funnel** on the canvas, the **exit breakdown**, and a per-contact **timeline**.
- **Move a managed workflow into git** (or a git one back to managed): **Promote** opens a PR into the repo; **Adopt** brings a git-backed workflow under dashboard editing.

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
  "segments": { "high-value": { "path": "audience/high-value.sendql", "name": "High value" } },
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
