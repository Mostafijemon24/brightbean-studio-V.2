# PRD Gap Analysis — BrightBean Studio vs. "AI-Powered Social Media SaaS" PRD

**Date:** 2026-10-18
**Scope:** Audit of the provided Product Requirements Document against the current
`brightbean-studio` codebase (branch `claude/happy-curie-w55te5`). This is a
read-only analysis — no code was changed.

---

## 0. Executive summary

BrightBean Studio is already a mature, self-hostable Django 5 social media
management platform. **Roughly 70% of the PRD is already built and production-grade.**
The core publishing, scheduling, calendar, inbox, approvals, analytics, media,
RBAC, and connected-account machinery is solid and well-tested.

The PRD's genuinely *new* demands cluster into a small number of areas, almost all
of which the repo's own roadmap already lists as "Phase 5 / Phase 6 / not yet built":

1. **AI Content Studio** (generative captions/titles/hashtags/CTAs) — **absent**
2. **AI Image Studio** (image generation from prompts) — **absent**
3. **AI provider abstraction / BYO key / model + cost config** — **absent**
4. **Engagement automation** (keyword comment triggers, comment-to-DM, auto-reply) — **absent**
5. **Contacts / Lead management (CRM-lite)** — **absent**
6. **Evergreen content library + recycling** — **absent** (fixed recurrence exists)
7. **Per-platform SEO / image-dimension validation** — **mostly absent** (char limits exist)
8. **Developer Super Admin panel** (feature flags, cost tracking, AI config) — **absent**
9. **First-class "Client" entity + 4-tier agency hierarchy** — **absent** (Workspace doubles as client)

Two cross-cutting decisions must be made before any build (see §3).

### Status legend

| Symbol | Meaning |
|--------|---------|
| ✅ **Built** | Implemented and production-grade |
| 🟡 **Partial** | Exists but incomplete vs. the PRD, or limited to some platforms |
| ❌ **Missing** | Not implemented anywhere |

---

## 1. What the codebase actually is

| Layer | Reality |
|-------|---------|
| Backend | Django 5.1, django-ninja (REST), MCP server |
| Frontend | **Django templates + HTMX + Alpine.js** (server-rendered) |
| CSS | Tailwind 4 via `django-tailwind` |
| Database | PostgreSQL 16+ (**also the job queue**) |
| Background jobs | **`django-background-tasks`** (Postgres-backed — no Redis/Celery). Worker = `python manage.py process_tasks` |
| Media | Local FS or S3-compatible (R2/S3/MinIO/B2); Pillow + FFmpeg |
| Providers | First-party API adapters in `providers/` (no aggregator) |
| Agent surface | REST API (`apps/api`) + OAuth 2.1 server (`apps/oauth_server`) + MCP server (`apps/mcp`) |
| Deploy | Docker + Heroku/Render/Railway one-click; Caddy + Gunicorn |

Multi-tenant model: **Organization → Workspace → SocialAccount**. Every tenant row is
scoped via opt-in managers (`apps/common/managers.py`). ~23 apps under `apps/`.

---

## 2. The two framing caveats (read before anything else)

### 2.1 The attached design tokens assume a different stack

The uploaded `tokens.css`, `tokens.json`, `tailwind.config.js`, and `components.md`
describe a **Next.js + shadcn/ui** component system (combobox, drawer, listbox
patterns, React-style primitives). The actual app is **Django templates + HTMX +
Alpine + Tailwind 4**. The *design tokens themselves* (color roles, type scale,
spacing, radius, the WCAG-passing palette, status-badge roles) are portable and
could be adopted as CSS custom properties in `theme/static_src/`. But the component
specs are written for a React/shadcn world and do **not** map 1:1 to this codebase.

**Decision needed:** adopt only the token layer (recommended, low-risk) vs. treat the
design files as a signal to migrate the frontend (large, separate project).

### 2.2 The PRD lists X/Twitter; the repo deliberately excludes it

PRD §3 requires **X (Twitter)**. The repo's own feature spec states X/Twitter is
*explicitly excluded* (`development_specs/feature-spec-social-media-management-v2.md`),
and there is **no `providers/twitter.py` / `x.py` adapter** and no registry entry.
Adding X means a new provider adapter *and* reversing a stated product decision
(X's API terms/pricing were the likely reason).

**Decision needed:** add X/Twitter, or formally drop it from the PRD.

---

## 3. Section-by-section gap matrix

### §3 — Supported platforms

| Platform | Status | Evidence |
|----------|--------|----------|
| Facebook Pages | ✅ | `providers/facebook.py` |
| Instagram Professional | ✅ | `providers/instagram.py`, `instagram_login.py` |
| LinkedIn profile + org | 🟡 | `providers/linkedin_company.py` (full), `linkedin_personal.py` (OIDC mode: posting works; inbox/first-comment/analytics `NotImplementedError`) |
| **X / Twitter** | ❌ | No adapter; deliberately excluded (see §2.2) |
| Pinterest + boards | 🟡 | `providers/pinterest.py` posts, but **no board picker** on connect |
| Google Business Profile + locations | 🟡 | `providers/google_business.py` posts, but **no location picker** on connect |

Bonus platforms beyond the PRD that already exist: **TikTok, YouTube, Threads,
Bluesky, Mastodon, DEV.to** — all with real `publish_post` adapters registered in
`providers/__init__.py:PROVIDER_REGISTRY`.

### §4 — Agency / multi-client / multi-account hierarchy

| Requirement | Status | Evidence / gap |
|-------------|--------|----------------|
| Agency workspace | ✅ | `apps/organizations/models.py::Organization` |
| **First-class Client entity** | ❌ | **No `Client` model.** "Client" is only a *role* (`WorkspaceMembership.CLIENT`) and the magic-link portal (`apps/client_portal/`). An agency must create **one Workspace per client**; nothing groups multiple accounts under a client within a workspace, nor multiple workspaces under a client. |
| 4-tier agency→clients→accounts→destinations | ❌ | Only 3 tiers: Org → Workspace → SocialAccount (`apps/social_accounts/models.py`) |
| Multiple clients per workspace | ❌ | Workspace holds a flat list of accounts, no client sub-grouping |
| Client switcher | 🟡 | Only an implicit **workspace** switcher (sidebar + `User.last_workspace_id`, `apps/members/middleware.py`). No client-level switcher because no client entity. |
| Multi-org per user | ❌ | Hardcoded "one org per user" in `apps/members/middleware.py` (`OrgMembership...first()`) |
| Connect once, select multiple destinations | 🟡 | **Meta/LinkedIn-Company only.** FB Pages / IG Business / LinkedIn Company get a destination picker (`apps/social_accounts/views.py::select_account` + `account_select.html`). Pinterest boards, GBP locations, YouTube, Mastodon connect as a single account with no picker. |

> This is the single biggest structural gap. The PRD treats "Client" as the central
> organizing object; the codebase treats "Workspace" as that object. See §5 for the
> remediation choice.

### §5 — Unified destination picker

🟡 **Partial.** A reusable multi-select destination picker exists but only for the
Meta/LinkedIn-Company connect flow. There is no single reusable component spanning
Client → Platform → Account → Page/Board/Location used across composer, bulk import,
scheduling, and automation targeting (because board/location destinations aren't
modeled uniformly). Composer does target multiple `PlatformPost` destinations per
`Post`, which is the publish-side half of this requirement. ✅

### §6 — AI Content Studio

❌ **Missing entirely.** No AI caption/title/hashtag/CTA generation, no brand voice,
no platform-specific AI formatting, no content variations/repurposing.
Confirmed by grep across `apps/composer`. The `apps/intelligence` app is **not** this
(see §19). The composer is otherwise strong: `Post`, `PlatformPost` per-platform
overrides, `PostVersion` history, `PostTemplate`, `ContentCategory`, `Tag`, Kanban
`Idea`/`IdeaGroup` board (`apps/composer/models.py`). Approval workflow ✅ (`apps/approvals`).

### §7 — AI Image Studio

❌ **Missing entirely.** Only image *thumbnailing/resizing* exists
(`apps/media_library/services.py::generate_image_thumbnail`, Pillow). No generation
from prompts, no platform presets, no brand-logo/overlay compositing, no aspect
validation.

### §8 — Platform-specific SEO & content validation

| Check | Status | Evidence |
|-------|--------|----------|
| Character-count / caption length | ✅ | `apps/social_accounts/models.py::PLATFORM_CHAR_LIMITS`, `PlatformPost.is_over_limit`, live preview endpoint |
| Media presence/type validation | ✅ | `providers/__init__.py::requires_media`/`is_video_only`, `apps/composer/views.py::_validate_required_media` |
| Video duration (TikTok) | ✅ | `providers/tiktok.py` |
| Upload file size/MIME | ✅ | `apps/media_library/validators.py` |
| **Per-platform image dimension / aspect-ratio** | ❌ | Not validated anywhere (crop can be *stored* in `PostMedia.platform_overrides`, never checked) |
| Hashtag / keyword guidance | ❌ | Static workspace default only; no suggestions/counting |
| Alt-text assistance | 🟡 | Stored (`PostMedia.alt_text`) but not required/suggested |
| Unified pre-publish checklist | 🟡 | Validation is scattered at save/publish time; no single consolidated checklist UI |

### §9 — Unified content calendar

✅ **Built, strong.** Month/week/day/list views, drag-drop reschedule, 11 per-platform
status states (incl. pending-client/approval), filters (status/platform/account/author/
category/tag/date), timezone support, recurring `PostingSlot`s, named `Queue`s with
auto-slot assignment. All in `apps/calendar/`. (Client-level filtering = workspace
scoping, so it inherits the §4 limitation.)

### §10 — Bulk CSV / Excel import

🟡 **Partial (CSV only).** Full 3-step CSV flow — upload → column-mapping → preview →
confirm — with per-row validation and scheduling (`apps/composer/views.py::csv_upload`/
`csv_preview`/`csv_confirm_import`, model `CSVImportJob`). **Gaps:** no Excel (.xlsx)
support (stdlib `csv` only); import is synchronous/session-based rather than run as a
background job; no AI caption/image generation from imported titles.

### §11 — Evergreen content automation

🟡→❌ **Fixed recurrence yes, evergreen no.** `RecurrenceRule` (daily/weekly/monthly,
`apps/calendar/models.py`) with a daily generator task (`apps/calendar/tasks.py::
generate_recurring_posts`). **Missing:** evergreen library, randomized selection,
min-interval-between-repeats, auto queue replenishment, content expiration,
cross-post duplicate prevention. The repo spec lists "content recycling / evergreen
queues" as **Phase 6 Post-Launch**.

### §12 — Comment-to-DM & engagement automation

❌ **Missing entirely.** No keyword trigger engine, no comment-to-DM workflow, no
auto-reply/auto-responder, no rule engine (grep: no `Automation`/`TriggerRule`/
`AutoReply`/`KeywordRule` classes). What exists: a manual unified inbox
(`apps/inbox`), canned `SavedReply` templates, keyword-based *sentiment tagging*
(classification only), and — importantly — a per-platform **messaging permission
layer** already built: `apps/inbox/reply_policy.py` enforces Meta's 24h/7day DM
windows and scope checks. That policy layer is the right foundation to validate any
future automation against.

### §13 — Contacts / Lead management

❌ **Missing entirely.** No `Contact`/`Lead`/CRM model anywhere. Sender identity is
only denormalized fields on `InboxMessage` (`sender_name`/`sender_handle`/`avatar`);
no persistent person entity, no identity resolution, no campaign attribution, no lead
pipeline.

### §14 — Simple main dashboard (≤6 nav items)

🟡 **Partial / needs review.** The app is server-rendered with a workspace sidebar.
Whether the current IA collapses to the PRD's six destinations (Overview, Content,
Calendar, Automations, Contacts, Settings) needs a UI pass — and "Automations" and
"Contacts" don't exist yet (§12, §13). Overview/dashboard real empty-states vs. fake
stats: needs verification against the PRD's "no fake data" rule.

### §15 — Workspace roles & team permissions

🟡 **Strong but role names differ.** Robust server-side RBAC: decorators in
`apps/members/decorators.py`, org roles (owner/admin/member) + workspace roles
(owner/manager/editor/contributor/client/viewer), a 13-key permission matrix, and
privilege-escalation guards (`apps/members/services.py`). **Gaps vs PRD:** no
"Publisher" or "Content Creator" *named* roles (closest: `editor`/`contributor` +
the `publish_directly` permission flag); `CustomRole` model exists but has **no
app-level create/assign UI** — Django-admin only, and invitation flows ignore it.

### §16 — Connected account management

✅ **Built, strong.** `apps/social_accounts/` list/reconnect/disconnect views,
`ConnectionStatus` enum, token-expiry warnings, missing-scope detection, webhook
health, `last_health_check_at`/`last_error`, sidebar unhealthy-channel surfacing.
Non-OAuth flows (Bluesky app-password, DEV.to key, Mastodon dynamic registration)
all present. The only gap is the §4 destination-discovery coverage.

### §17 — Separate API & background infrastructure

✅ **Built (different engine).** Publishing is **worker-driven, not browser-driven**
(`apps/publisher/tasks.py::run_publish_cycle` every 15s under `process_tasks`). The
PRD asks for "a reliable queue and worker system"; the repo uses
**`django-background-tasks` (Postgres-backed)** rather than Celery/Redis. That
satisfies the functional requirement; if the PRD specifically wants Celery, that's a
migration, not a gap. Clean separation of REST API / OAuth server / MCP server exists.

### §18 — Developer Super Admin panel

❌ **Missing.** No custom admin site or superuser console — only stock Django
`admin/` with minimal `ModelAdmin` registrations. No license/subscription management,
no AI-provider config, no feature flags, no global usage/cost tracking, no failed-job
inspection UI. (`apps/api_keys` has per-org key management + usage counters, but
scoped to the org, not a global system console.)

### §19 — AI provider & API cost management

❌ **Missing.** No LLM provider abstraction, no model selection, no usage quotas/cost
estimation/BYO-key for generative AI. **Important clarification:** `apps/intelligence`
is **not** a generative-AI layer — it is a Studio-side integration to an external,
Stripe-gated **YouTube packaging-analytics** SaaS (`score_packaging`,
`benchmark_channel`, etc.), with subscription plumbing (`IntelligenceSubscription`,
`IntelligenceUsageEvent`). It proves the billing/usage-audit *pattern* but provides no
content/image generation.

### §20 — Security & data isolation

🟡 **Strong, with one structural caveat.** ✅ Encrypted token/credential storage
(`apps/common/encryption.py`, `EncryptedTextField`/`EncryptedJSONField`), ✅ server-side
RBAC, ✅ allauth + Google SSO + TOTP 2FA, ✅ CSP/ratelimit/CSRF, ✅ privilege-escalation
guards, ✅ 14-day reversible org deletion. **Caveat:** tenant isolation uses **opt-in
scoped managers**, not enforced default/row-level scoping — any queryset that forgets
`.for_workspace()`/`.for_org()` can leak across tenants. The `RBACMiddleware` membership
check is the backstop. Worth a focused review, especially the separate
`client_portal` session path.

### §21 — Recommended technical architecture

✅ Already a modular monolith with Postgres-backed workers — closely matches the PRD's
"modular monolith + background workers, avoid microservices" recommendation. The
*only* divergence is the PRD's suggested Next.js/Tailwind/shadcn + managed-auth stack
vs. the repo's Django/HTMX/allauth reality (see §2.1).

### §22 — Core database entities

Most PRD entities exist (Users, Workspaces, Memberships, Roles, SocialAccounts,
Credentials, Content items/variants, Media, Calendar entries, Publishing results,
Notifications, API keys/audit). **Missing entities:** `Client`, `PublishingDestination`
(as first-class, beyond SocialAccount), `AIGenerationRecord`, `EvergreenRule`,
`AutomationRule`, `Contact`, `Campaign`, `License`/`Subscription` (for the core product),
global `AuditEvent`, `FeatureFlag`.

### §23 — Reliability & publishing workflow

✅ **Built, excellent.** One `Post` fans out to N `PlatformPost` children, **each with
independent status + retry budget + result** — exactly the PRD's "independent publishing
job per destination" requirement. Duplicate prevention via
`transaction.atomic()` + `select_for_update()` status-claim (`apps/publisher/engine.py`),
backoff retries, retryable-vs-permanent classification, `PublishLog` 90-day audit,
`RateLimitState` per account. A partial-destination failure does **not** fail the whole
post. The only reliability gap vs §18 is no user-facing failed-job dashboard
(visibility is per-post + Sentry today).

### §24 — Marketplace-ready delivery

✅ **Largely built.** Dockerfile, docker-compose (dev/prod), Caddyfile, Heroku/Render/
Railway configs, `.env.example`, `README`, `CONTRIBUTING`, `SECURITY`, AGPL-3.0 license,
pytest suite, CI. **Note on resale:** the project is **AGPL-3.0**, which materially
affects the PRD's "license and resell the source repeatedly" business model — AGPL
requires derivative/hosted works to publish source under the same terms. **This needs
legal review before adopting the PRD's commercial-resale plan.**

---

## 4. Prioritized build order (the genuinely new work)

Ordered by dependency and value. Effort is rough (S/M/L/XL). Everything here is
*additive* to the existing platform.

### Decision gate 0 — resolve before building (blocking)

- **D1. Client model vs. Workspace-as-client (§4).** Either (a) adopt "Workspace = Client"
  and reframe the PRD's agency hierarchy onto it (cheapest, no migration), or
  (b) introduce a first-class `Client` entity grouping accounts within a workspace
  (large migration touching isolation, RBAC, calendar filters, publishing). **Recommend (a)**
  unless multiple distinct clients per workspace is a hard requirement.
- **D2. X/Twitter — add or drop (§2.2).**
- **D3. Design tokens — adopt token layer only, or migrate frontend (§2.1).** Recommend token-only.
- **D4. AGPL vs. resale license model (§24).** Legal review.

### Phase 1 — AI foundation (unlocks §6, §7, §19) — **L**
1. **AI provider abstraction layer** + BYO-key + model/quota/cost config
   (new app, e.g. `apps/ai`). Reuse the `apps/intelligence` billing/usage-audit pattern
   and `EncryptedTextField` for keys. **M**
2. **AI Content Studio** — caption/title/hashtag/CTA generation, brand voice,
   per-platform variants, inside the existing composer. Add `AIGenerationRecord`. **L**
3. **AI Image Studio** — prompt→image via the provider layer, platform presets,
   brand-logo overlay, write into `media_library`. **L**

### Phase 2 — validation & quality (§8) — **M**
4. **Per-platform image dimension/aspect validation** + unified **pre-publish checklist**
   component (consolidate the scattered save/publish checks). **M**
5. **Hashtag/keyword guidance** (counting + suggestions, optionally AI-assisted). **S**

### Phase 3 — engagement automation + CRM (§12, §13) — **L**
6. **Contacts/Lead model** (`apps/contacts`): person entity, source platform, tags,
   lead status, interaction history, identity resolution over `InboxMessage`. **M**
7. **Keyword automation engine** (`apps/automations`): comment triggers → auto-reply
   and comment-to-DM, validated against the existing `reply_policy.py` permission layer.
   Run on the background-task worker. **L**

### Phase 4 — evergreen + bulk (§10, §11) — **M**
8. **Evergreen library + recycling rules** (randomized selection, min-interval,
   expiration, auto queue replenishment, duplicate prevention), extending `apps/calendar`. **M**
9. **Excel import** + move bulk import to a real background job writing `CSVImportJob`. **S**

### Phase 5 — admin & ops (§18, §22) — **M**
10. **Super Admin console** (feature flags, global usage/cost tracking, failed-job
    inspection over the background-task tables, AI-provider config surface). **M**
11. **Destination-picker uniformity (§4/§5):** Pinterest board + GBP location pickers
    on connect, and a single reusable destination-picker component. **M**

### Phase 6 — stack/brand decisions (optional, large)
12. Adopt the design-token layer in `theme/static_src/` (**S**) — or, if chosen,
    a frontend migration (**XL**, separate initiative).
13. Add X/Twitter adapter if D2 = add (**M**).

---

## 5. One-paragraph verdict

BrightBean Studio already *is* most of this PRD — the hard, reliability-critical parts
(multi-tenant publishing engine, per-destination jobs, calendar, inbox, approvals,
analytics, RBAC, connected-account health, deployment) are built and good. The PRD's
new surface area is concentrated and buildable on top of the existing architecture:
a generative-AI layer (Content + Image Studio + provider/cost management), an
engagement-automation engine with a lightweight CRM, evergreen recycling, richer
per-platform validation, and a developer admin console. Before building, settle the
four decision-gate items — the **Client-entity question** and the **AGPL-vs-resale
license question** are the two that could otherwise force expensive rework.
