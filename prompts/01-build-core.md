# Claude Code kickoff: build grantdesk core

You are building `grantdesk`, an open-source pipeline, compliance calendar, and win/loss ledger for grant consultants. This document is your brief. Read it fully before writing code.

This is roughly eight to ten weeks of work for one engineer. It is sequenced into milestones that each ship something usable. Do not attempt to build it in one pass, and do not skip ahead: the tenancy spine in Milestone 0 is load-bearing for everything after it, and the recurrence engine in Milestone 4 is the reason the project exists.

---

## 1. Mission

A grant consultant runs their practice in a spreadsheet. It works for the chase and it fails completely after the award, which is where the expensive mistakes live: an interim report missed by two weeks, a budget modification filed after the window closed, a final report that costs a client their renewal.

Build a thin, opinionated tool that does three things:

1. **Pipeline** — opportunities by stage, deadline, owner, and client, with the letter-of-inquiry / full-proposal branch that grant work actually has.
2. **Compliance calendar** — once an award is logged, every reporting obligation, drawdown window, and modification deadline is tracked and surfaced ahead of time, with recurring obligations generated from a rule rather than entered one at a time. **This is the differentiator. If you have to cut something, cut anywhere else.**
3. **Win/loss ledger** — every submission with amount requested, amount awarded, funder type, and outcome, which exports as a segmented grant professional portfolio.

Success looks like: a consultant with seven clients imports their spreadsheet in an hour, logs a three-year federal award in three minutes, and gets eighteen correctly dated obligations without typing eighteen rows.

---

## 2. Read these first, in this order

1. `docs/program/CONVENTIONS.md` — binding for the whole program. Apache-2.0, `NOTICE`, dual CLI and MCP interface, optional OpenGrants integration that degrades silently, data honesty, the required disclosure text.
2. `docs/program/RESEARCH.md` — verified facts. Do not re-derive them, and do not invent competitor pricing or API details that are not in it.
3. `docs/program/HOSTING.md` — Cloudflare rationale and the caching strategy.
4. `docs/NON-GOALS.md` in this repository. **Read this twice.** It is the scope contract and it is the single most likely thing to get violated by a well-meaning implementation decision.
5. `docs/research/data-sources.md` — the domain model. The federal reporting rules, the anchoring vocabulary, and the stage sequence in that file are requirements, not background.
6. `docs/research/prior-art.md` and `docs/research/competitive.md` — why the shape is what it is, including why RFC 5545 `RRULE` was studied and rejected as the internal recurrence representation.
7. `README.md` — the promises already made to users. Do not break them.

---

## 3. Hard constraints

These are not preferences. A design that violates one of them is wrong even if it works.

### 3.1 Multi-tenancy from the first migration

Consultants have multiple clients. Every nonprofit tool that fails consultants fails here, because it assumed one organization per install and could never afford to change.

- Two levels. **`workspace`** is the consulting practice, the top-level tenant. **`organization`** is a client. A workspace has many organizations.
- **Every domain table carries both `workspace_id` and `org_id`**, denormalized on purpose. Do not derive tenancy through a join. A single indexed column that is always present is a predicate you cannot forget, and a join you can get wrong.
- **All reads and writes go through a scoped repository layer** in `src/db/repo/`. It takes a `TenancyContext { workspaceId, orgIds, userId, role }` and applies the predicate itself. Route handlers, services, the CLI, and the MCP server never write `WHERE org_id = ?` by hand.
- **Two tests enforce this and must exist by the end of Milestone 0:**
  - `test/tenancy/schema-invariant.test.ts` reads the schema, enumerates domain tables from an explicit list, and fails if any lacks `workspace_id` or `org_id`.
  - `test/tenancy/no-raw-queries.test.ts` scans `src/` outside `src/db/repo/` and `migrations/` and fails on any reference to a domain table name in a raw SQL string or a Drizzle query builder call.
- **A cross-tenant leak is a release blocker**, not a bug to schedule.

Some tables are workspace-scoped without an organization: `organization` itself, `user`, `membership`, `funder`, `api_token`, `session`. Those are listed explicitly in the invariant test's exemption list, and the exemption list is short and does not grow without a comment explaining each entry.

### 3.2 Scope discipline

Build a thin, opinionated tool, not a CRM. The moment this grows contact management, email synchronization, donation processing, or a form builder, it becomes a product with a support burden instead of a repository.

Specifically, and non-exhaustively: no contacts, no inbox sync, no send-from-app, no calendar write-back (one-way iCal export only), no task management, no payment processing, no accounting, no form builder, no proposal editor, no filing on the user's behalf, no portal scraping, no grantmaker-side features, no plugin system, no user-defined automation, no custom fields, no client portal, no mobile app, no SAML. The full list with reasoning is `docs/NON-GOALS.md`.

When you find yourself designing something that is not in this prompt, stop and check that file. If it is on the list, do not build it. If it is not on the list and you think it is necessary, it goes on the stop-and-ask list in section 13.

### 3.3 Two deployment targets at parity

Cloudflare Workers with D1, and Node with SQLite in Docker Compose. One schema, one migration set, one test suite that runs against both.

- Data layer: **Drizzle ORM, SQLite dialect**, one schema file. `drizzle-orm/d1` on Workers, `drizzle-orm/better-sqlite3` on Node.
- No D1-only SQL and no SQLite extensions the other target lacks.
- Everything platform-specific goes behind an interface in `src/platform/`, each with a Workers implementation and a Node implementation: `Scheduler`, `ObjectStore`, `KeyValue`, `Mailer`, `Clock`, `Random`.
- `env.SOMETHING` accessed anywhere outside `src/platform/` and `src/config.ts` is a bug.
- Note the one real divergence and handle it explicitly: **D1 does not support interactive transactions.** Use `db.batch()` for multi-statement atomicity on both targets, exposed through a single `runBatch()` helper. Do not write code that depends on `BEGIN ... COMMIT`.

### 3.4 Server-rendered, minimal JavaScript

Hono JSX rendered server-side. HTML forms that POST. Progressive enhancement only: every action must work with JavaScript disabled. No client framework, no build step for the UI beyond TypeScript compilation, one hand-maintained CSS file.

A consultant opens this on hotel wifi the night before a deadline. Server-rendered HTML is the version that loads.

### 3.5 Deadlines are dates, not instants

Store `due_on` as a `TEXT` date in `YYYY-MM-DD`, never as a timestamp. Compliance deadlines are calendar dates in a jurisdiction, and storing them as instants produces the classic bug where a report due 2027-01-30 renders as January 29 for a user in a different timezone.

Timezone lives on the award (`award.timezone`, defaulting to the workspace default) and is used only to decide what "today" is when computing overdue status and when to send a digest. All date arithmetic in the recurrence engine is calendar arithmetic on plain dates, with no `Date` object timezone semantics anywhere in it. Write the date helpers yourself as pure functions over `{ y, m, d }`; do not reach for a date library for this, and do not use `new Date(...)` inside the engine.

### 3.6 Optional OpenGrants, never required

Every feature works with no `OPENGRANTS_API_KEY`. Enrichment is wrapped so a network failure or an expired key degrades silently to the un-enriched result. Enriched values carry a `— live from OpenGrants` marker in the interface. No key means no nag: the key is mentioned once in the README and never in application output.

### 3.7 The required disclosure

Per `docs/program/CONVENTIONS.md`, this tool reports on compliance posture, so this text appears in the application footer, in every digest email, at the foot of the portfolio export, and in `--json` CLI output as a `disclosure` field:

> This is informational only, derived from public data on the dates shown. It is not an eligibility determination, and not legal, tax, or accounting advice. Verify against the official source before relying on it.

---

## 4. Stack and repository layout

TypeScript strict, pnpm, biome, vitest, Hono. Node 20+, pnpm 9+.

```
grantdesk/
├── README.md  LICENSE  NOTICE  CONTRIBUTING.md  CODE_OF_CONDUCT.md  SECURITY.md
├── CHANGELOG.md  .env.example  .gitignore  biome.json  tsconfig.json
├── package.json  pnpm-lock.yaml  wrangler.jsonc  drizzle.config.ts
├── Dockerfile  docker-compose.yml
├── migrations/                 0001_init.sql, 0002_..., forward-only
├── src/
│   ├── config.ts               env parsing and validation, one place
│   ├── db/
│   │   ├── schema.ts           Drizzle schema, single source of truth
│   │   ├── client.ts           driver selection, runBatch()
│   │   └── repo/               scoped repositories, the ONLY place with queries
│   ├── domain/
│   │   ├── dates.ts            pure calendar arithmetic, no Date objects
│   │   ├── holidays.ts         US federal holiday table + business-day rules
│   │   ├── recurrence.ts       THE ENGINE. pure, no I/O
│   │   ├── templates/          built-in obligation templates, one file each
│   │   ├── stages.ts           stage machine and legal transitions
│   │   └── analytics.ts        win/loss and portfolio computation, pure
│   ├── services/               orchestration; uses repo + domain
│   ├── http/
│   │   ├── app.ts              Hono app assembly
│   │   ├── routes/             ui/, api/, auth/, webhooks/, ical/, mcp/
│   │   ├── middleware/         session, tenancy, csrf, security-headers
│   │   └── views/              Hono JSX components
│   ├── integrations/opengrants/
│   ├── platform/               Scheduler, ObjectStore, KeyValue, Mailer, Clock
│   ├── mcp/                    MCP server, thin adapter over services
│   ├── cli/                    CLI, thin adapter over services
│   ├── worker.ts               Workers entry: fetch + scheduled
│   └── node.ts                 Node entry: @hono/node-server + node-cron
├── public/styles.css
├── test/
│   ├── fixtures/               realistic committed fixtures
│   ├── tenancy/                the two invariant tests
│   ├── domain/                 table-driven recurrence tests
│   └── e2e/                    runs against BOTH targets
└── .github/workflows/ci.yml
```

**Business logic in a route handler is a bug.** Core logic lives in `src/domain/` and `src/services/`. The web UI, the JSON API, the CLI, and the MCP server are four thin adapters over the same services.

Scripts in `package.json`:

```
dev            wrangler dev
dev:node       tsx watch src/node.ts
build          tsc -b && (worker bundle via wrangler)
check          biome check . && tsc --noEmit
test           vitest run
test:node      vitest run --project node
test:workers   vitest run --project workers
db:generate    drizzle-kit generate
db:migrate:local   wrangler d1 migrations apply grantdesk --local
db:migrate:remote  wrangler d1 migrations apply grantdesk --remote
seed:demo      tsx src/cli/index.ts seed demo
```

---

## 5. Data model

Full DDL. This is the schema to implement, expressed as SQL for clarity; define it in `src/db/schema.ts` with Drizzle and generate the migration from it so the two cannot drift.

Conventions: ids are `TEXT` ULIDs (26 chars, lexicographically sortable, generated in `src/platform/random.ts`). Timestamps are `TEXT` ISO-8601 UTC with `Z`. Calendar dates are `TEXT` `YYYY-MM-DD`. Money is `INTEGER` cents; never a float. Enumerations are `TEXT` with `CHECK` constraints, because SQLite has no enum type and a `CHECK` is the only thing that stops a typo becoming a permanent data problem.

```sql
-- ===========================================================================
-- TENANCY
-- ===========================================================================

CREATE TABLE workspace (
  id            TEXT PRIMARY KEY,
  name          TEXT NOT NULL,
  slug          TEXT NOT NULL UNIQUE,
  timezone      TEXT NOT NULL DEFAULT 'America/Los_Angeles',
  lead_days     TEXT NOT NULL DEFAULT '30,14,7,1',
  created_at    TEXT NOT NULL,
  updated_at    TEXT NOT NULL
);

CREATE TABLE user (
  id            TEXT PRIMARY KEY,
  email         TEXT NOT NULL UNIQUE,   -- stored lowercased
  name          TEXT,
  timezone      TEXT,
  digest_hour   INTEGER NOT NULL DEFAULT 7,
  digest_enabled INTEGER NOT NULL DEFAULT 1,
  created_at    TEXT NOT NULL,
  last_seen_at  TEXT
);

CREATE TABLE membership (
  id            TEXT PRIMARY KEY,
  workspace_id  TEXT NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
  user_id       TEXT NOT NULL REFERENCES user(id) ON DELETE CASCADE,
  role          TEXT NOT NULL CHECK (role IN ('owner','admin','member','viewer')),
  all_orgs      INTEGER NOT NULL DEFAULT 1,  -- 0 = restricted, see org_access
  created_at    TEXT NOT NULL,
  UNIQUE (workspace_id, user_id)
);

-- Only consulted when membership.all_orgs = 0. A subcontract writer on one
-- engagement sees exactly the orgs listed here.
CREATE TABLE org_access (
  id            TEXT PRIMARY KEY,
  workspace_id  TEXT NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
  user_id       TEXT NOT NULL REFERENCES user(id) ON DELETE CASCADE,
  org_id        TEXT NOT NULL REFERENCES organization(id) ON DELETE CASCADE,
  created_at    TEXT NOT NULL,
  UNIQUE (workspace_id, user_id, org_id)
);

CREATE TABLE organization (            -- a CLIENT
  id                   TEXT PRIMARY KEY,
  workspace_id         TEXT NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
  name                 TEXT NOT NULL,
  slug                 TEXT NOT NULL,
  ein                  TEXT,           -- 'NN-NNNNNNN', normalized on write
  uei                  TEXT,           -- SAM.gov Unique Entity ID
  sam_expires_on       TEXT,           -- date; drives the federal gate warning
  fiscal_year_end_month INTEGER CHECK (fiscal_year_end_month BETWEEN 1 AND 12),
  entity_type          TEXT CHECK (entity_type IN
                         ('501c3','501c4','government','school_district',
                          'tribal','fiscally_sponsored','for_profit','other')),
  timezone             TEXT,
  notes                TEXT,
  archived_at          TEXT,
  created_at           TEXT NOT NULL,
  updated_at           TEXT NOT NULL,
  UNIQUE (workspace_id, slug)
);

-- ===========================================================================
-- FUNDERS  (workspace-scoped, shared across clients, deliberately not org-scoped)
-- ===========================================================================

CREATE TABLE funder (
  id            TEXT PRIMARY KEY,
  workspace_id  TEXT NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
  name          TEXT NOT NULL,
  funder_type   TEXT NOT NULL CHECK (funder_type IN
                  ('federal','state','local_government','private_foundation',
                   'family_foundation','community_foundation','corporate_foundation',
                   'corporate_giving','united_way_or_federated','individual_or_other')),
  ein           TEXT,
  website       TEXT,
  portal_kind   TEXT CHECK (portal_kind IN
                  ('grants_gov','assist','research_gov','foundant','fluxx',
                   'submittable','sm_apply','blackbaud','email','other','none')),
  portal_url    TEXT,
  opengrants_funder_id TEXT,
  notes         TEXT,
  created_at    TEXT NOT NULL,
  updated_at    TEXT NOT NULL
);
CREATE INDEX idx_funder_ws ON funder(workspace_id, name);

-- ===========================================================================
-- PIPELINE
-- ===========================================================================

CREATE TABLE opportunity (
  id             TEXT PRIMARY KEY,
  workspace_id   TEXT NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
  org_id         TEXT NOT NULL REFERENCES organization(id) ON DELETE CASCADE,
  funder_id      TEXT REFERENCES funder(id) ON DELETE SET NULL,
  title          TEXT NOT NULL,
  external_ref   TEXT,          -- NOFO number, e.g. 'HRSA-26-042'
  assistance_listing TEXT,      -- CFDA, e.g. '93.243'
  track          TEXT NOT NULL DEFAULT 'foundation'
                   CHECK (track IN ('federal','foundation','state_local','corporate')),
  stage          TEXT NOT NULL CHECK (stage IN
                   ('forecast','identified','qualifying','inquiry',
                    'loi_drafting','loi_submitted','invited',
                    'proposal_drafting','proposal_submitted','site_visit',
                    'under_review','awarded','declined','not_invited',
                    'withdrawn','not_pursued')),
  stage_entered_at TEXT NOT NULL,
  owner_user_id  TEXT REFERENCES user(id) ON DELETE SET NULL,
  amount_requested_cents INTEGER,
  deadline_is_hard INTEGER NOT NULL DEFAULT 0,   -- federal deadlines are hard
  source         TEXT NOT NULL DEFAULT 'manual'
                   CHECK (source IN ('manual','csv_import','opengrants_search',
                                     'opengrants_alert','opengrants_webhook')),
  opengrants_grant_id  TEXT,
  saved_search_id      TEXT REFERENCES saved_search(id) ON DELETE SET NULL,
  last_enriched_at     TEXT,
  tags           TEXT,          -- JSON array of strings
  notes          TEXT,
  archived_at    TEXT,
  created_at     TEXT NOT NULL,
  updated_at     TEXT NOT NULL
);
CREATE INDEX idx_opp_tenant   ON opportunity(workspace_id, org_id, stage);
CREATE INDEX idx_opp_owner    ON opportunity(workspace_id, owner_user_id);
CREATE INDEX idx_opp_og       ON opportunity(opengrants_grant_id);

-- The LOI / full-proposal split lives here. An opportunity has MANY dates.
-- There is deliberately no single `deadline` column on opportunity.
CREATE TABLE opportunity_deadline (
  id             TEXT PRIMARY KEY,
  workspace_id   TEXT NOT NULL,
  org_id         TEXT NOT NULL,
  opportunity_id TEXT NOT NULL REFERENCES opportunity(id) ON DELETE CASCADE,
  kind           TEXT NOT NULL CHECK (kind IN
                   ('loi_due','full_proposal_due','submission_target',
                    'docket_cutoff','site_visit','decision_expected',
                    'inquiry_by','other')),
  due_on         TEXT NOT NULL,              -- YYYY-MM-DD
  label          TEXT,
  source         TEXT NOT NULL DEFAULT 'manual'
                   CHECK (source IN ('manual','template','opengrants')),
  completed_on   TEXT,
  created_at     TEXT NOT NULL,
  updated_at     TEXT NOT NULL
);
CREATE INDEX idx_oppdl ON opportunity_deadline(workspace_id, org_id, due_on);

CREATE TABLE stage_transition (
  id             TEXT PRIMARY KEY,
  workspace_id   TEXT NOT NULL,
  org_id         TEXT NOT NULL,
  opportunity_id TEXT NOT NULL REFERENCES opportunity(id) ON DELETE CASCADE,
  from_stage     TEXT,
  to_stage       TEXT NOT NULL,
  user_id        TEXT,
  occurred_at    TEXT NOT NULL
);

-- ===========================================================================
-- WIN / LOSS LEDGER
-- ===========================================================================
-- One row per thing actually SENT. An LOI and the full proposal that follows
-- are two rows on the same opportunity, and they are never blended into one
-- conversion rate.

CREATE TABLE submission (
  id              TEXT PRIMARY KEY,
  workspace_id    TEXT NOT NULL,
  org_id          TEXT NOT NULL,
  opportunity_id  TEXT NOT NULL REFERENCES opportunity(id) ON DELETE CASCADE,
  funder_id       TEXT REFERENCES funder(id) ON DELETE SET NULL,
  submission_type TEXT NOT NULL CHECK (submission_type IN
                    ('loi','full_proposal','renewal','continuation','report_only')),
  submitted_on    TEXT NOT NULL,
  method          TEXT CHECK (method IN
                    ('grants_gov','assist','research_gov','portal','email','mail','other')),
  confirmation_ref TEXT,
  amount_requested_cents INTEGER,
  is_competitive  INTEGER NOT NULL DEFAULT 1,
  lead_writer_user_id TEXT REFERENCES user(id) ON DELETE SET NULL,
  outcome         TEXT NOT NULL DEFAULT 'pending' CHECK (outcome IN
                    ('pending','awarded_full','awarded_partial','declined',
                     'not_invited','invited','withdrawn','no_response',
                     'ineligible_discovered_late')),
  outcome_on      TEXT,
  amount_awarded_cents INTEGER,
  decline_reason  TEXT CHECK (decline_reason IN
                    ('not_a_fit','funding_exhausted','too_competitive',
                     'budget_concerns','capacity_concerns','incomplete_application',
                     'missed_deadline','unknown')),
  decline_note    TEXT,
  created_at      TEXT NOT NULL,
  updated_at      TEXT NOT NULL
);
CREATE INDEX idx_sub_tenant ON submission(workspace_id, org_id, submitted_on);
CREATE INDEX idx_sub_writer ON submission(workspace_id, lead_writer_user_id, submitted_on);

-- ===========================================================================
-- AWARDS
-- ===========================================================================

CREATE TABLE award (
  id                 TEXT PRIMARY KEY,
  workspace_id       TEXT NOT NULL,
  org_id             TEXT NOT NULL,
  opportunity_id     TEXT REFERENCES opportunity(id) ON DELETE SET NULL,
  submission_id      TEXT REFERENCES submission(id) ON DELETE SET NULL,
  funder_id          TEXT REFERENCES funder(id) ON DELETE SET NULL,
  name               TEXT NOT NULL,
  award_number       TEXT,                    -- e.g. '5R01GM123456-03'
  noa_date           TEXT,                    -- Notice of Award date
  amount_cents       INTEGER NOT NULL,
  -- Period of performance. NOT the budget period. See docs/research/data-sources.md
  period_start       TEXT NOT NULL,
  period_end         TEXT NOT NULL,
  budget_period_end  TEXT,                    -- current budget period, if multi-year
  is_federal         INTEGER NOT NULL DEFAULT 0,
  assistance_listing TEXT,
  pass_through_entity TEXT,                   -- set when this is a subaward
  payment_method     TEXT CHECK (payment_method IN
                       ('reimbursement','advance_drawdown','tranche','lump_sum','other')),
  indirect_rate_bp   INTEGER,                 -- basis points, 1000 = 10.00%
  portal_url         TEXT,
  program_officer    TEXT,                    -- a name and an email. NOT a contact record.
  timezone           TEXT,
  status             TEXT NOT NULL DEFAULT 'active'
                       CHECK (status IN ('active','closing','closed','terminated')),
  closed_at          TEXT,
  notes              TEXT,
  created_at         TEXT NOT NULL,
  updated_at         TEXT NOT NULL
);
CREATE INDEX idx_award_tenant ON award(workspace_id, org_id, status);

-- ===========================================================================
-- COMPLIANCE CALENDAR
-- ===========================================================================

CREATE TABLE obligation_rule (
  id             TEXT PRIMARY KEY,
  workspace_id   TEXT NOT NULL,
  org_id         TEXT NOT NULL,
  award_id       TEXT NOT NULL REFERENCES award(id) ON DELETE CASCADE,
  template_key   TEXT,          -- which built-in template produced this, if any
  kind           TEXT NOT NULL CHECK (kind IN
                   ('financial_report','performance_report','payment_report',
                    'payment_request','drawdown_window','final_financial_report',
                    'final_performance_report','closeout','budget_modification',
                    'prior_approval_deadline','no_cost_extension_deadline',
                    'property_report','subaward_report','audit_submission',
                    'registration_renewal','renewal_application','site_visit',
                    'acknowledgment','custom')),
  title_template TEXT NOT NULL, -- 'SF-425 Federal Financial Report, {period_label}'
  -- What the periods are anchored to:
  anchor         TEXT NOT NULL CHECK (anchor IN
                   ('award_start','award_end','budget_period_end',
                    'federal_fiscal_quarter','calendar_quarter','calendar_year_end',
                    'org_fiscal_year_end','fixed_date')),
  anchor_date    TEXT,          -- required when anchor = 'fixed_date'
  frequency      TEXT NOT NULL CHECK (frequency IN
                   ('once','monthly','quarterly','semiannual','annual')),
  -- Days from the PERIOD END to the DUE DATE. NEGATIVE means due BEFORE the
  -- period ends, which is how NIH RPPR and NSF annual reports actually work.
  offset_days    INTEGER NOT NULL DEFAULT 0,
  business_day_rule TEXT NOT NULL DEFAULT 'none'
                   CHECK (business_day_rule IN ('none','next_business_day','previous_business_day')),
  first_period_start TEXT,      -- defaults to the anchor's natural start
  last_period_end    TEXT,      -- defaults to award.period_end
  max_occurrences    INTEGER,
  lead_days      TEXT,          -- override of workspace default, e.g. '45,14,7'
  assignee_user_id TEXT REFERENCES user(id) ON DELETE SET NULL,
  citation       TEXT,          -- '2 CFR 200.328'
  citation_checked_on TEXT,     -- date the contributor verified it
  is_pattern_not_requirement INTEGER NOT NULL DEFAULT 0,  -- true for foundation patterns
  active         INTEGER NOT NULL DEFAULT 1,
  created_at     TEXT NOT NULL,
  updated_at     TEXT NOT NULL
);
CREATE INDEX idx_rule_award ON obligation_rule(workspace_id, award_id, active);

CREATE TABLE obligation (
  id             TEXT PRIMARY KEY,
  workspace_id   TEXT NOT NULL,
  org_id         TEXT NOT NULL,
  award_id       TEXT NOT NULL REFERENCES award(id) ON DELETE CASCADE,
  rule_id        TEXT REFERENCES obligation_rule(id) ON DELETE SET NULL,
  -- Idempotency key for generation: rule_id + period_start. A regeneration run
  -- that produces the same key updates in place instead of duplicating.
  generation_key TEXT,
  kind           TEXT NOT NULL,      -- same CHECK list as obligation_rule.kind
  title          TEXT NOT NULL,
  period_start   TEXT,
  period_end     TEXT,
  due_on         TEXT NOT NULL,
  status         TEXT NOT NULL DEFAULT 'upcoming' CHECK (status IN
                   ('upcoming','due','overdue','submitted','accepted',
                    'waived','not_applicable')),
  assignee_user_id TEXT REFERENCES user(id) ON DELETE SET NULL,
  submitted_on   TEXT,
  submitted_by_user_id TEXT REFERENCES user(id) ON DELETE SET NULL,
  external_ref   TEXT,               -- confirmation number from the portal
  evidence_url   TEXT,
  citation       TEXT,
  citation_checked_on TEXT,
  is_pattern_not_requirement INTEGER NOT NULL DEFAULT 0,
  -- Set to 1 the moment a human edits a generated row. Regeneration NEVER
  -- overwrites a row with user_modified = 1 or a status past 'due'.
  user_modified  INTEGER NOT NULL DEFAULT 0,
  linked_opportunity_id TEXT REFERENCES opportunity(id) ON DELETE SET NULL,
  notes          TEXT,
  created_at     TEXT NOT NULL,
  updated_at     TEXT NOT NULL,
  UNIQUE (rule_id, generation_key)
);
CREATE INDEX idx_obl_due    ON obligation(workspace_id, due_on, status);
CREATE INDEX idx_obl_org    ON obligation(workspace_id, org_id, due_on);
CREATE INDEX idx_obl_assign ON obligation(workspace_id, assignee_user_id, due_on);
CREATE INDEX idx_obl_award  ON obligation(workspace_id, award_id, due_on);

-- Built-in and workspace-authored template definitions. Built-ins ship in code
-- under src/domain/templates/ and are mirrored here on first use so that a
-- template edit in one workspace never affects another.
CREATE TABLE obligation_template (
  id            TEXT PRIMARY KEY,
  workspace_id  TEXT,          -- NULL = built-in, visible to all workspaces
  key           TEXT NOT NULL, -- 'federal.hhs.discretionary.quarterly-425'
  name          TEXT NOT NULL,
  description   TEXT,
  applies_to    TEXT NOT NULL CHECK (applies_to IN ('federal','foundation','state_local','any')),
  rules_json    TEXT NOT NULL, -- array of rule specs, validated by zod on load
  citation      TEXT,
  citation_checked_on TEXT,
  is_pattern_not_requirement INTEGER NOT NULL DEFAULT 0,
  created_at    TEXT NOT NULL,
  updated_at    TEXT NOT NULL,
  UNIQUE (workspace_id, key)
);

-- ===========================================================================
-- INTEGRATION, AUTH, AUDIT
-- ===========================================================================

CREATE TABLE saved_search (
  id            TEXT PRIMARY KEY,
  workspace_id  TEXT NOT NULL,
  org_id        TEXT NOT NULL,
  label         TEXT NOT NULL,
  query_json    TEXT NOT NULL,
  auto_create   INTEGER NOT NULL DEFAULT 0,  -- 0 = review queue, 1 = straight to pipeline
  last_synced_at TEXT,
  last_error    TEXT,
  created_by    TEXT REFERENCES user(id) ON DELETE SET NULL,
  created_at    TEXT NOT NULL,
  updated_at    TEXT NOT NULL
);

CREATE TABLE webhook_event (
  id             TEXT PRIMARY KEY,
  provider       TEXT NOT NULL DEFAULT 'opengrants',
  external_id    TEXT NOT NULL,
  event_type     TEXT NOT NULL,
  workspace_id   TEXT,
  signature_ok   INTEGER NOT NULL,
  payload_json   TEXT NOT NULL,
  received_at    TEXT NOT NULL,
  processed_at   TEXT,
  error          TEXT,
  UNIQUE (provider, external_id)
);

CREATE TABLE session (
  id            TEXT PRIMARY KEY,    -- opaque, 32 random bytes, base64url
  user_id       TEXT NOT NULL REFERENCES user(id) ON DELETE CASCADE,
  workspace_id  TEXT NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
  created_at    TEXT NOT NULL,
  expires_at    TEXT NOT NULL,
  last_used_at  TEXT,
  user_agent_hash TEXT,
  revoked_at    TEXT
);

CREATE TABLE login_token (
  id            TEXT PRIMARY KEY,
  email         TEXT NOT NULL,
  token_hash    TEXT NOT NULL UNIQUE,   -- SHA-256(token + pepper)
  workspace_id  TEXT,
  purpose       TEXT NOT NULL CHECK (purpose IN ('login','invite')),
  expires_at    TEXT NOT NULL,
  consumed_at   TEXT,
  created_at    TEXT NOT NULL
);

CREATE TABLE api_token (
  id            TEXT PRIMARY KEY,
  workspace_id  TEXT NOT NULL REFERENCES workspace(id) ON DELETE CASCADE,
  user_id       TEXT NOT NULL REFERENCES user(id) ON DELETE CASCADE,
  name          TEXT NOT NULL,
  token_hash    TEXT NOT NULL UNIQUE,
  prefix        TEXT NOT NULL,          -- first 8 chars, shown in the UI
  scopes        TEXT NOT NULL,          -- JSON array: ['read'] or ['read','write']
  org_ids       TEXT,                   -- JSON array; NULL = all orgs the user can see
  last_used_at  TEXT,
  expires_at    TEXT,
  revoked_at    TEXT,
  created_at    TEXT NOT NULL
);

CREATE TABLE ical_feed (
  id            TEXT PRIMARY KEY,
  workspace_id  TEXT NOT NULL,
  user_id       TEXT NOT NULL REFERENCES user(id) ON DELETE CASCADE,
  token_hash    TEXT NOT NULL UNIQUE,
  org_ids       TEXT,                   -- JSON array; NULL = everything the user can see
  created_at    TEXT NOT NULL,
  last_fetched_at TEXT,
  revoked_at    TEXT
);

CREATE TABLE activity (
  id            TEXT PRIMARY KEY,
  workspace_id  TEXT NOT NULL,
  org_id        TEXT NOT NULL,
  entity_type   TEXT NOT NULL,
  entity_id     TEXT NOT NULL,
  user_id       TEXT,
  kind          TEXT NOT NULL,   -- 'note' | 'status_change' | 'created' | ...
  body          TEXT,
  created_at    TEXT NOT NULL
);
CREATE INDEX idx_activity ON activity(workspace_id, entity_type, entity_id, created_at);

CREATE TABLE audit_log (
  id            TEXT PRIMARY KEY,
  workspace_id  TEXT,
  actor_user_id TEXT,
  actor_token_id TEXT,
  action        TEXT NOT NULL,
  entity_type   TEXT,
  entity_id     TEXT,
  changes_json  TEXT,            -- field names and old/new; never full record bodies
  ip_hash       TEXT,
  created_at    TEXT NOT NULL
);

CREATE TABLE attachment (
  id            TEXT PRIMARY KEY,
  workspace_id  TEXT NOT NULL,
  org_id        TEXT NOT NULL,
  entity_type   TEXT NOT NULL,
  entity_id     TEXT NOT NULL,
  kind          TEXT NOT NULL CHECK (kind IN ('link','file')),
  url           TEXT,            -- for kind='link'
  object_key    TEXT,            -- for kind='file': 'ws/<wsid>/org/<orgid>/<ulid>'
  filename      TEXT,
  content_type  TEXT,
  bytes         INTEGER,
  sha256        TEXT,
  uploaded_by   TEXT,
  created_at    TEXT NOT NULL
);
```

### Relationship summary

```
workspace 1─n organization 1─n opportunity 1─n opportunity_deadline
                                        │
                                        └─n submission ──1 award ──n obligation_rule
                                                                │        │
                                                                └─n obligation ←┘
workspace 1─n funder            (shared across clients)
workspace 1─n user via membership, optionally narrowed by org_access
award.period_end moves ⇒ obligations anchored to award_end move, EXCEPT any
  already submitted or user_modified
```

Two modeling decisions worth stating because they will be questioned:

**`funder` is workspace-scoped, not org-scoped.** A consultant approaches the same community foundation for four clients. Duplicating the funder per client destroys the segmentation the portfolio export depends on. Funder rows carry `workspace_id` only and are exempt from the `org_id` invariant, which is one of the short list of exemptions.

**`submission` is the ledger, `opportunity` is the pipeline.** An opportunity that never got submitted has no submission row and correctly does not appear in any hit rate. This is what keeps the denominator honest.

---

## 6. The recurrence engine

`src/domain/recurrence.ts`. Pure functions, no I/O, no `Date` objects, fully unit tested. This is the highest-value and highest-risk code in the repository.

### Signature

```ts
export type PlainDate = { y: number; m: number; d: number };  // m is 1-12

export interface RuleSpec {
  kind: ObligationKind;
  titleTemplate: string;
  anchor: Anchor;
  anchorDate?: PlainDate;
  frequency: Frequency;
  offsetDays: number;               // negative = due BEFORE period end
  businessDayRule: BusinessDayRule;
  firstPeriodStart?: PlainDate;
  lastPeriodEnd?: PlainDate;
  maxOccurrences?: number;
}

export interface AwardContext {
  periodStart: PlainDate;
  periodEnd: PlainDate;
  budgetPeriodEnd?: PlainDate;
  orgFiscalYearEndMonth?: number;
}

export interface Occurrence {
  periodStart: PlainDate;
  periodEnd: PlainDate;
  dueOn: PlainDate;
  periodLabel: string;              // 'Q2 FY2027', 'Year 1', 'Final'
  generationKey: string;            // `${anchor}:${iso(periodStart)}`
}

export function expand(
  rule: RuleSpec,
  award: AwardContext,
  horizonEnd: PlainDate,
): Occurrence[];
```

### Algorithm

1. **Determine the period grid** from `anchor` and `frequency`.
   - `federal_fiscal_quarter`: quarters ending Dec 31, Mar 31, Jun 30, Sep 30. Federal fiscal year runs Oct 1 to Sep 30, so the quarter ending Dec 31 2026 is Q1 FY2027. Label accordingly, because that is what the form asks for.
   - `calendar_quarter`: quarters ending Mar 31, Jun 30, Sep 30, Dec 31, labeled by calendar year.
   - `award_start`: periods of the given frequency starting on `award.periodStart` and stepping by anniversary. A period starting 2026-02-15 with annual frequency ends 2027-02-14.
   - `budget_period_end`: periods ending on `budgetPeriodEnd` and its anniversaries.
   - `org_fiscal_year_end`: periods ending on the last day of `orgFiscalYearEndMonth`.
   - `calendar_year_end`: Dec 31.
   - `award_end` with `frequency: 'once'`: a single period equal to the whole period of performance. This is the shape of every final report and every closeout obligation.
   - `fixed_date`: a single period ending on `anchorDate`. Used for SAM renewal and one-off dates.
2. **Clip the grid.** Drop periods ending before `firstPeriodStart ?? award.periodStart`. Drop periods starting after `lastPeriodEnd ?? award.periodEnd`. Stop at `maxOccurrences`. Stop at `horizonEnd`. **Partial first and last periods are kept, not dropped**, because a federal award starting 2026-11-03 owes an SF-425 for the stub quarter ending 2026-12-31.
3. **Compute the due date:** `dueOn = addDays(periodEnd, offsetDays)`.
4. **Apply the business-day rule** using `src/domain/holidays.ts`: US federal holidays with the observed-day rule (a holiday falling on Saturday is observed Friday, on Sunday is observed Monday), plus weekends. `next_business_day` is the correct default for most federal reporting; `none` is correct where the award text names a hard calendar date.
5. **Label the period** for the title template: `Q1 FY2027`, `Year 2`, `Jan–Jun 2027`, `Final`.
6. **Compute `generationKey`** deterministically from the anchor and the period start. Never from the due date, which moves when a holiday rule changes.

### Materialization, and the rules that make it safe

`src/services/obligations.ts` turns occurrences into `obligation` rows.

- **Idempotent.** Upsert on `(rule_id, generation_key)`. Running expansion twice changes nothing.
- **Never clobber human work.** A row with `user_modified = 1`, or with `status` in `('submitted','accepted','waived','not_applicable')`, is left exactly as it is. Regeneration only touches rows that are still `upcoming` or `due` and untouched.
- **Lazy horizon.** Expand only to `OBLIGATION_HORIZON_DAYS` ahead (default 540). A five-year award does not create sixty rows on day one. The hourly scheduled job extends the horizon as time passes.
- **Award date changes cascade correctly.** When `award.period_end` moves, which is what a no-cost extension does, re-expand every active rule for that award. Obligations already filed keep their dates. Future unmodified obligations move. Obligations whose period no longer exists, because the award shrank, are marked `not_applicable` with a note rather than deleted, so the history survives.
- **Deactivating a rule** sets `active = 0`, leaves history intact, and marks future unmodified rows `not_applicable`.
- **Preview before commit.** Every path that creates or edits a rule shows the expansion first: the exact dated list the user is about to create. `preview_obligation_rule` in the MCP server and `grantdesk rules preview` in the CLI expose the same dry run. Users will not trust generated deadlines they cannot see before they exist, and they are right not to.

### Status computation

Status is derived at read time from `due_on` and today's date in the award's timezone, not only written by the scheduler. `upcoming` becomes `due` when today is within the first lead-day threshold, and `due` becomes `overdue` when today is past `due_on`. The scheduler persists the same values so queries and digests are cheap, but **a scheduler that has not run must never hide a deadline.** Compute, then persist; do not depend on the persist.

### Built-in templates

Under `src/domain/templates/`, one file each, validated by zod, each carrying `citation` and `citation_checked_on`. Ship these:

| Key | Shape |
|---|---|
| `federal.generic.quarterly-425` | Quarterly SF-425, federal fiscal quarter anchor, +30 days; annual performance report, +90; final financial and final performance at award end +120; no-cost extension reminder at award end −30; prior approval reminder at award end −60. Citations: 2 CFR 200.328, 200.329, 200.344, 200.308. |
| `federal.generic.annual-425` | Annual SF-425 at +90 instead of quarterly, same closeout set. |
| `federal.nih.rppr` | RPPR due the first of the month preceding the budget period end (non-SNAP), or −45 days (SNAP); final RPPR and final SF-425 at +120. Marked **VERIFY** until checked against the current NIH Grants Policy Statement. |
| `federal.nsf.annual` | Annual project report at budget period end −90; final project report and Project Outcomes Report at award end +120. Marked **VERIFY**. |
| `federal.compliance.sam-renewal` | Annual, anchored to `sam_expires_on`, reminder at −60 days. |
| `federal.compliance.single-audit` | Annual, anchored to the organization's fiscal year end, due at +270 days, generated only when the operator confirms federal expenditures are at or above $1,000,000. |
| `foundation.interim-final-12mo` | Interim narrative and financial at the 6-month mark; final report at grant end +30. Marked pattern, not requirement. |
| `foundation.final-only` | Single final report at grant end +30. Pattern. |
| `foundation.multiyear-annual` | Annual report at each year end −30, because the report releases the next payment. Pattern. |
| `foundation.tranche-on-report` | Interim report at the midpoint plus a linked payment-expected obligation 30 days after. Pattern. |

Applying a template creates `obligation_rule` rows the user can then edit. Templates are a starting point, never a black box, and the preview appears before anything is created.

---

## 7. Authentication and authorization

**Approach: emailed magic link plus server-side sessions.** No passwords, no OAuth provider, no SAML in version one. Rationale is in `docs/NON-GOALS.md` section 10: the initial user is a solo consultant or a six-person firm, and an identity provider integration is a barrier rather than a feature.

**Sign-in flow.** POST an email to `/login`. If a `membership` exists for it, generate 32 random bytes, store `SHA-256(token + pepper)` in `login_token` with a 15-minute expiry, and email `${APP_URL}/login/verify?token=...`. Consuming it creates a `session` row and sets the cookie. **Respond identically whether or not the address has an account**, so the endpoint is not an account-existence oracle. Rate limit by email and by IP.

**Accounts are never created by sign-in.** Only by invitation from a workspace owner or admin, or by first-run setup. An exposed instance is not an open door.

**Cookie:** `HttpOnly`, `Secure` in production, `SameSite=Lax`, `Path=/`, absolute expiry 30 days, sliding refresh on use. The cookie holds the opaque session id only, never a token containing claims.

**Authorization**, evaluated by middleware into a `TenancyContext` and consulted by the repository layer:

| Role | Can |
|---|---|
| `owner` | Everything, including workspace deletion, billing-equivalent settings, and member management |
| `admin` | Everything except workspace deletion and owner transfer |
| `member` | Create and edit records for organizations they can access |
| `viewer` | Read only |

Orthogonally: `membership.all_orgs = 0` narrows the accessible organization set to the rows in `org_access`. The repository layer applies `org_id IN (...)` from the context, always, with no way to opt out.

**CSRF:** every state-changing form carries a token bound to the session; the middleware also checks `Origin` against `APP_URL` on all POSTs.

**API tokens:** `gd_` prefix plus 32 random bytes, shown once at creation, stored hashed. Scopes are `read` or `read,write`, and a token may be narrowed to a subset of organizations. Every use is recorded in `audit_log` with the token id.

**Security headers** on every response: `Content-Security-Policy` with no inline script (hash the small amount of JavaScript you ship), `X-Content-Type-Options: nosniff`, `Referrer-Policy: same-origin`, `Strict-Transport-Security` in production, and `X-Robots-Tag: noindex` plus `Cache-Control: private, no-store` on everything under `/app`, `/api`, `/mcp`, and `/ical`.

---

## 8. API surface

REST under `/api/v1`, JSON in and out, zod-validated, authenticated by session cookie or `Authorization: Bearer gd_...`. Errors are `{ error: { code, message, details? } }` with correct status codes. Every list endpoint paginates with `?limit=&cursor=` and returns `{ data, next_cursor }`.

**Every request is workspace-scoped by the credential.** The workspace is never a parameter. Organization is a filter: `?org=riverside-housing` or `?org=all` where the caller's access permits.

```
GET    /api/v1/me
GET    /api/v1/organizations
POST   /api/v1/organizations
GET    /api/v1/organizations/:idOrSlug
PATCH  /api/v1/organizations/:id

GET    /api/v1/funders
POST   /api/v1/funders

GET    /api/v1/opportunities            ?org= &stage= &owner= &due_before= &q=
POST   /api/v1/opportunities
GET    /api/v1/opportunities/:id
PATCH  /api/v1/opportunities/:id
POST   /api/v1/opportunities/:id/stage        { to_stage, note? }
POST   /api/v1/opportunities/:id/deadlines    { kind, due_on, label? }
DELETE /api/v1/opportunities/:id/deadlines/:dlId

GET    /api/v1/submissions              ?org= &since= &outcome= &writer=
POST   /api/v1/submissions
PATCH  /api/v1/submissions/:id                 outcome, amounts, decline reason

GET    /api/v1/awards                   ?org= &status=
POST   /api/v1/awards
GET    /api/v1/awards/:id
PATCH  /api/v1/awards/:id                      moving period_end re-expands rules
POST   /api/v1/awards/:id/apply-template       { template_key, dry_run? }

GET    /api/v1/awards/:id/rules
POST   /api/v1/awards/:id/rules                { ...RuleSpec, dry_run? }
PATCH  /api/v1/rules/:id
DELETE /api/v1/rules/:id                       deactivates; never hard-deletes history
POST   /api/v1/rules/preview                   pure expansion, creates nothing

GET    /api/v1/obligations              ?org= &from= &to= &status= &assignee= &kind=
POST   /api/v1/obligations                     ad-hoc, no rule
PATCH  /api/v1/obligations/:id                 sets user_modified = 1
POST   /api/v1/obligations/:id/submit          { submitted_on, external_ref?, evidence_url? }

GET    /api/v1/analytics/winloss        ?org= &from= &to= &segment_by=funder_type|size_band
GET    /api/v1/analytics/portfolio      ?user= &from= &to=      the career export
GET    /api/v1/export/portfolio.csv
GET    /api/v1/export/portfolio.pdf

GET    /api/v1/saved-searches
POST   /api/v1/saved-searches
POST   /api/v1/saved-searches/:id/sync

POST   /api/v1/import/opportunities            multipart CSV, dry-run by default

POST   /webhooks/opengrants                    HMAC-SHA256 verified
GET    /ical/:token                            read-only RFC 5545 feed
```

**The `dry_run` parameter is not optional decoration.** Anything that creates more than one record supports it and returns exactly what would be created.

---

## 9. UI surface

Hono JSX, server-rendered. One CSS file at `public/styles.css`, roughly 400 lines, no framework, no build step. Dark mode via `prefers-color-scheme`. All actions work without JavaScript.

Screens, in build order:

**`/app` — Today.** The landing screen and the one that has to earn its place in five seconds. Across every client the user can see: obligations overdue, obligations due in the next 14 days, opportunity deadlines in the next 30 days, and anything blocked (a federal opportunity whose client's SAM registration expires before the deadline). Grouped by urgency, not by client.

**`/app/calendar` — Compliance calendar.** List and month views. Filters: client, kind, assignee, status, date range. Every row shows client, award, obligation, due date, days remaining, assignee, and the citation. One-click "mark submitted" with a confirmation number field. A prominent "download calendar feed" that produces the iCal URL with an explanation of exactly what it exposes.

**`/app/pipeline` — Pipeline.** Board grouped by stage, and a table view because a table is what people actually filter and sort. Columns: client, funder, amount requested, next date and its kind, owner, days in stage. Stage advance is a form POST with a `<select>`. Table view is the default on narrow screens.

**`/app/opportunities/:id`.** Detail, deadline list, stage history, notes, the link to the submission it produced, and the OpenGrants panel when enriched (marked `— live from OpenGrants`, with a proposed-change row when the upstream deadline differs).

**`/app/awards/:id`.** Award facts, obligations grouped by year, the rules that generate them with an inline preview of the next six occurrences, applied template with citation, portal link, and program officer as two plain fields.

**`/app/awards/:id/rules/new`.** The rule builder. Kind, anchor, frequency, offset, business-day rule, date range, assignee. **A live preview panel showing the exact dated list, updating on submit, before anything is created.** This screen is the product. Spend the time here.

**`/app/ledger` — Win/loss.** Every submission, filterable, with segmented rates in the header rather than one blended number.

**`/app/portfolio` — The career export.** Segmented by funder type and award size band, denominators visible, small segments labeled insufficient, LOI-to-invitation and invitation-to-award separated, multi-year trend, per-writer selector, "as of" date, the disclosure. Export as CSV and as a printable HTML page that prints to PDF cleanly. Do not add a PDF library for this; a print stylesheet is the correct amount of engineering.

**`/app/clients`, `/app/clients/:id`.** Client list with counts. Client detail showing EIN, UEI, SAM expiry with a warning when it is inside 90 days, fiscal year end, and links to the same EIN on `check.opengrants.io` and `funders.opengrants.io`.

**`/app/settings/*`.** Workspace, members and invitations, per-org access, API tokens, iCal feeds, OpenGrants key status (present or absent, never a nag), import and export.

**Keyboard shortcuts** as pure progressive enhancement: `g t` Today, `g c` calendar, `g p` pipeline, `/` search. Under thirty lines of JavaScript.

---

## 10. MCP server and CLI

Both are thin adapters over `src/services/`. Per `docs/program/CONVENTIONS.md`, business logic in a command handler or a tool handler is a bug.

### MCP

`npx grantdesk mcp --token gd_...` over stdio, and `POST /mcp` over HTTP with a bearer token on the hosted deployment. Use the official TypeScript SDK.

Tools:

| Tool | Notes |
|---|---|
| `list_organizations` | The agent's way to discover valid `org` values |
| `list_opportunities` | Filters: org, stage, owner, due_before |
| `get_opportunity` | |
| `create_opportunity` | |
| `advance_opportunity_stage` | Validates the transition against `stages.ts` |
| `list_obligations` | org, window, status, assignee. The most-used tool. |
| `get_obligation` | |
| `mark_obligation_submitted` | |
| `create_award` | |
| `list_obligation_templates` | Returns citations and check dates |
| `preview_obligation_rule` | **Pure. Creates nothing.** |
| `apply_obligation_template` | Requires `confirm: true` after a preview |
| `create_obligation_rule` | Requires `confirm: true` after a preview |
| `record_submission` | |
| `record_submission_outcome` | |
| `winloss_summary` | Segmented, never a single blended rate |
| `portfolio_summary` | The career export, as structured data |
| `sync_saved_search` | Only when an OpenGrants key is configured |

**Every tool that touches client data takes an explicit `org` argument. There is no implicit current client and no default.** An agent guessing wrong about which client it is acting on is the one failure this tool cannot have. Tool descriptions say so, and the server returns a clear error naming the available organizations when `org` is missing or ambiguous.

Anything that creates records requires an explicit `confirm: true`, and the preview tool is the intended precursor.

### CLI

`npx grantdesk <command>`. Human-readable by default, `--json` for machines, `--json` output always includes the disclosure field.

```
grantdesk demo [--reset]              zero-config demo on :8787, seeded
grantdesk serve                       run the Node target
grantdesk migrate [--local|--remote]
grantdesk seed demo
grantdesk import opportunities <csv> --org <slug> [--dry-run]
grantdesk import awards <csv> --org <slug> [--dry-run]
grantdesk export portfolio [--user <email>] [--since <date>] [--json|--csv]
grantdesk export all --out <dir>      full workspace export, CSV per table
grantdesk rules preview --award <id>  dry-run expansion, printed as a table
grantdesk obligations due [--days 30] [--org <slug>]
grantdesk token create --name "laptop" [--scopes read]
grantdesk mcp --token <token>
```

`grantdesk demo` is the 60-second quickstart and it must need nothing: no account, no key, no database setup. It creates a temp SQLite file, applies migrations, seeds the demo workspace, opens a browser, and prints the URL. **Test this on a clean machine. It is the first thing every evaluator will run, and if it needs a second command the README's central promise is false.**

---

## 11. Migrations

- Numbered SQL files in `migrations/`, applied by `wrangler d1 migrations apply` on Workers and by an equivalent runner on Node that shares the same directory and the same tracking table.
- **Forward-only. A migration on `main` is immutable.** People are running this against real client records.
- Generate from the Drizzle schema with `drizzle-kit generate`, then read the generated SQL before committing it. Do not commit generated SQL you have not read.
- Every migration gets a test that applies it to a fixture database and asserts the resulting shape.
- Destructive migrations need a note in the pull request describing what data is lost and what to back up.
- `0001_init.sql` contains the whole schema in section 5. Do not dribble the schema out across ten migrations during initial development; squash before the first tag.

---

## 12. Milestones

Each milestone ends with something a real user could use, tests passing on both targets, and a `CHANGELOG.md` entry. Do not start the next one until the current one is green.

### M0 — Skeleton and tenancy spine (week 1)

Repository scaffolding, `package.json`, `tsconfig.json` strict, `biome.json`, `vitest.config.ts` with both projects, `wrangler.jsonc`, `Dockerfile`, `docker-compose.yml`, CI workflow. Drizzle schema for the full model in section 5. `0001_init.sql`. Driver selection and `runBatch()`. The `src/platform/` interfaces with both implementations. The scoped repository layer. Magic-link auth, sessions, the tenancy middleware. First-run setup creating a workspace and its owner. **Both tenancy invariant tests.** A single page listing the workspace's organizations.

*Ships:* you can sign in and create clients. Nothing else works, and the foundation is correct.

### M1 — Pipeline (week 2)

Funders. Opportunities with stages and the stage machine in `stages.ts`, including legal-transition validation and `stage_transition` history. `opportunity_deadline` with all its kinds. Pipeline table and board views, opportunity detail, notes. CSV import for opportunities with a dry run. Today screen showing opportunity deadlines only.

*Ships:* a consultant can move their spreadsheet in and run their pipeline. This alone is worth using.

### M2 — Submissions and the ledger (week 3)

Submissions, LOI and full proposal as separate rows, outcomes and decline reasons. Ledger screen with filters. First segmented analytics: hit rate by funder type, LOI-to-invitation reported separately from invitation-to-award, denominators visible, small segments labeled. CSV export of the ledger.

*Ships:* the win/loss record exists and is honest.

### M3 — Awards and manual obligations (week 4)

Awards with period of performance and budget period modeled correctly. Manually created obligations. Calendar screen, list and month views, filtered across all clients. Mark-submitted flow. Status computation at read time. Award detail.

*Ships:* the compliance calendar works, entered by hand. Already better than the spreadsheet.

### M4 — The recurrence engine (weeks 5 and 6)

`src/domain/dates.ts` and `holidays.ts` as pure functions with exhaustive tests. `recurrence.ts` implementing every anchor and frequency in section 6, including negative offsets. The materialization service: idempotent upsert, never clobbering human work, lazy horizon, cascade on award date change. The rule builder UI with live preview. The built-in template library with citations and check dates. `rules preview` in the CLI.

*Ships:* the differentiator. This is the milestone the project is for. Do not compress it.

### M5 — Surfacing (week 6, overlapping)

Lead-time ladder and status transitions. Scheduled jobs on both platforms: hourly re-expansion and status recompute, daily digest. Digest email, one per user per day, grouped by client, with the disclosure. iCal feed with a revocable token. Today screen completed with obligations. SAM expiry warnings on federal opportunities.

*Ships:* the calendar now tells you before, not after. This is what makes it a tool rather than a record.

### M6 — Portfolio export (week 7)

`analytics.ts` completed: size bands, multi-year trend, per-writer views, insufficient-denominator labeling. The portfolio screen. CSV export and the print stylesheet. The "as of" stamp and the disclosure on every export.

*Ships:* the career-development payoff.

### M7 — OpenGrants integration (weeks 7 and 8)

The API client with silent degradation, rate-limit header handling, and KV caching keyed on the query and not on the tenant. Saved searches, a review queue, optional auto-create. Deadline refresh as a proposed change, never a silent overwrite. Webhook receiver with HMAC-SHA256 verification over the raw body, idempotency on `(provider, external_id)`, and fast 2xx. The `— live from OpenGrants` marker everywhere enriched data appears.

*Ships:* the funnel from an alert to a pipeline item to an award to a compliance calendar, end to end.

### M8 — MCP, CLI, API tokens (week 8)

The full MCP tool set. API tokens with scopes and per-org narrowing. The CLI commands in section 10. `grantdesk demo` working on a clean machine. The JSON API documented.

*Ships:* the agent-facing and machine-facing surfaces.

### M9 — Shipping (week 9)

`grantdesk demo` verified on a clean machine on macOS, Linux, and Windows Subsystem for Linux. Deploy button flow tested end to end from a fresh Cloudflare account. Docker Compose path tested from a clean clone. Full workspace export and delete. `CHANGELOG.md`, `SECURITY.md`, `CODE_OF_CONDUCT.md`, `LICENSE`, generated `THIRD-PARTY-LICENSES.txt`. Tag `v0.1.0`.

---

## 13. Acceptance criteria

Checkable. A milestone is not done until its items pass.

**Tenancy**

- [ ] `test/tenancy/schema-invariant.test.ts` passes and fails when `org_id` is removed from any domain table
- [ ] `test/tenancy/no-raw-queries.test.ts` passes and fails when a query is added outside `src/db/repo/`
- [ ] A test creates two workspaces with similar data and asserts that every list endpoint, every export, the iCal feed, and every MCP tool returns only the caller's rows
- [ ] A test asserts a `member` with `all_orgs = 0` and access to one organization cannot read, write, or enumerate a second organization by direct id, and receives 404 rather than 403 so the endpoint is not an existence oracle

**Recurrence engine**

- [ ] A three-year federal award starting 2026-10-01 with `federal.generic.quarterly-425` produces exactly 12 quarterly SF-425 obligations, 3 annual performance reports, 1 final financial report, 1 final performance report, 1 no-cost extension reminder, and 1 prior approval reminder
- [ ] The first quarterly obligation for that award covers 2026-10-01 to 2026-12-31, is labeled **Q1 FY2027**, and computes a raw due date of **2027-01-30**; with the template's `next_business_day` rule it lands on **2027-02-01**, because 2027-01-30 is a Saturday
- [ ] The final financial report for an award ending 2029-09-30 is due **2030-01-28** (120 days after)
- [ ] The no-cost extension reminder for that award is due **2029-08-31** (30 days before period end)
- [ ] An award starting 2026-11-03 with a federal-fiscal-quarter rule produces a stub first period of 2026-11-03 to 2026-12-31, not a dropped period
- [ ] A rule with `offsetDays: -45` produces due dates before the period end
- [ ] `next_business_day` moves a due date of 2027-01-30 (a Saturday) to 2027-02-01, and correctly observes a federal holiday falling on a weekend
- [ ] Running expansion twice produces zero new rows and zero updates
- [ ] Editing an obligation sets `user_modified = 1`, and a subsequent expansion leaves it untouched
- [ ] Extending `award.period_end` by 12 months moves future unmodified obligations, leaves submitted ones untouched, and moves the closeout set
- [ ] Shrinking `award.period_end` marks orphaned future obligations `not_applicable` and deletes nothing
- [ ] A leap-day case: an annual rule anchored to 2028-02-29 produces 2029-02-28
- [ ] `preview_obligation_rule` and `POST /api/v1/rules/preview` create zero rows, asserted by a row count before and after

**Pipeline and ledger**

- [ ] Stage transitions are validated, and an invalid transition returns 422 with the legal transitions listed
- [ ] An opportunity can be created directly at `invited` or `proposal_drafting` without passing through earlier stages
- [ ] LOI-to-invitation and invitation-to-award rates are computed separately, and no endpoint or screen returns a single blended win rate
- [ ] A segment with fewer than 5 submissions renders as "insufficient data" rather than a percentage, in the UI, in the CSV, and in the JSON
- [ ] The portfolio export carries a generation date, the submission date range, and the disclosure

**Platform parity**

- [ ] `pnpm test:workers` and `pnpm test:node` both pass, running the same e2e suite
- [ ] No `env.` reference exists outside `src/platform/` and `src/config.ts`, asserted by a test
- [ ] The Docker Compose path comes up from a clean clone with only `SESSION_SECRET` set

**OpenGrants**

- [ ] With no key set, every screen, endpoint, CLI command, and MCP tool works, and no output mentions the key
- [ ] With an invalid key, the same, and the failure appears in logs only
- [ ] A webhook with a bad signature returns 401 and is not processed
- [ ] A redelivered webhook with the same `external_id` is a no-op
- [ ] Enriched values are visually marked, asserted by a rendering test

**The quickstart promise**

- [ ] `npx grantdesk demo` on a machine with only Node 20 gives a working instance with seeded data in under 60 seconds, with no account, no key, and no second command

**Hygiene**

- [ ] `pnpm check` clean, `tsc --noEmit` clean with strict on
- [ ] CI green on push and pull request, badge in the README
- [ ] Every built-in template has a `citation` and a `citation_checked_on`, asserted by a test
- [ ] The disclosure string appears in the app footer, the digest email, the portfolio export, and `--json` output, asserted by tests
- [ ] `Cache-Control: private, no-store` and `X-Robots-Tag: noindex` on every response under `/app`, `/api`, `/mcp`, `/ical`, asserted by a test that enumerates the router

---

## 14. Verification steps

Run these by hand at the end of each milestone. Do not rely only on the suite.

```bash
# Clean setup
pnpm install && pnpm check && pnpm test

# Both targets
pnpm db:migrate:local && pnpm seed:demo
pnpm dev        # http://localhost:8787
pnpm dev:node   # same app, SQLite

# The 60-second promise, from a clean machine
npx grantdesk demo

# The engine, without touching the database
npx grantdesk rules preview \
  --award-start 2026-10-01 --award-end 2029-09-30 \
  --template federal.generic.quarterly-425
# Expect: 12 quarterly SF-425 rows; the first covers 2026-10-01 to 2026-12-31,
# is labeled Q1 FY2027, computes 2027-01-30 and shifts to 2027-02-01 under
# next_business_day; final financial report 2030-01-28; NCE reminder 2029-08-31.

# Tenancy, by hand
curl -H "Authorization: Bearer $TOKEN_WS_A" \
  "http://localhost:8787/api/v1/opportunities?org=all" | jq '.data | length'
curl -H "Authorization: Bearer $TOKEN_WS_A" \
  "http://localhost:8787/api/v1/opportunities/$AN_ID_FROM_WORKSPACE_B"
# Expect 404, not 403, and nothing in the body that confirms the id exists.

# Idempotency
sqlite3 .wrangler/state/.../db.sqlite "select count(*) from obligation;"
curl -XPOST .../api/v1/awards/$AWARD/apply-template \
  -d '{"template_key":"federal.generic.quarterly-425"}'
curl -XPOST .../api/v1/awards/$AWARD/apply-template \
  -d '{"template_key":"federal.generic.quarterly-425"}'
sqlite3 ... "select count(*) from obligation;"   # unchanged after the second call

# Degradation
unset OPENGRANTS_API_KEY && pnpm test
OPENGRANTS_API_KEY=invalid pnpm test

# Docker, from a clean clone
docker compose up -d && curl -sf http://localhost:8787/healthz

# Headers
curl -sI http://localhost:8787/app | grep -Ei 'cache-control|x-robots-tag'
```

Manual checks that no test covers:

1. Open the calendar with 200 obligations across 7 clients. If you cannot find what is due next week in five seconds, the screen is wrong.
2. Log a three-year federal award from scratch, start to finish. If it takes more than three minutes, the form is wrong.
3. Read the rule builder preview as a grant consultant would. If the dated list is not immediately obviously correct, the labeling is wrong.
4. Print the portfolio export. If it does not look like something a person would attach to a job application, it is not finished.

---

## 15. Stop and ask the human

Do not decide these alone. Stop, write down the options and your recommendation, and wait.

**Scope**

1. Any feature not described in this prompt, especially anything resembling an item in `docs/NON-GOALS.md`. If a design pulls you toward contacts, email, tasks, payments, forms, or a client portal, stop.
2. Any change to the tenancy model, including "just this one table does not need `org_id`."
3. Adding a fourth top-level concept alongside pipeline, compliance, and ledger.

**Correctness with real consequences**

4. Any obligation template you cannot cite. A remembered deadline does not ship.
5. Any due-date rule where the regulation is ambiguous or where sources disagree, particularly the 90-versus-120-day closeout question and the NIH and NSF timings marked **VERIFY** in `docs/research/data-sources.md`.
6. Any behavior that would silently overwrite a user-entered date, including OpenGrants refresh.
7. Anything that deletes an obligation rather than marking it inapplicable.

**Security and data**

8. Any relaxation of the repository-layer tenancy predicate, for performance or anything else.
9. Anything that puts client data in a shared cache, a log line, an error message, an external service, or a URL that could be shared.
10. Any change to session handling, token hashing, or webhook signature verification.
11. Any telemetry, analytics, or phone-home. The default is none.

**Public claims**

12. Any competitor price or feature claim in user-facing copy. Re-verify on the vendor's own page and date-stamp it. `docs/program/RESEARCH.md` figures are research, not publishable copy.
13. Any wording that could read as a compliance determination or as legal, tax, or accounting advice.
14. Any change to the required disclosure text.

**Engineering**

15. Any new runtime dependency. Say what it replaces and why thirty lines of your own is worse.
16. Anything that breaks parity between the Workers and Docker targets.
17. Editing a migration that is already on `main`.
18. Adding a client-side framework or a build step for the UI.

---

## 16. Definition of done for v0.1.0

- All acceptance criteria in section 13 pass
- CI green, badge in the README
- `npx grantdesk demo` works on a clean machine with no account and no key
- The Deploy to Cloudflare button works from a fresh Cloudflare account
- `docker compose up` works from a clean clone with only `SESSION_SECRET` set
- Every built-in template carries a citation and a verification date
- The disclosure appears on every surface that reports compliance posture
- A workspace owner can export everything and delete the workspace, from the interface, without asking anyone
- `docs/NON-GOALS.md` is still accurate, and nothing you built contradicts it
