# Contributing to grantdesk

Thanks for being here. This document is short on ceremony and specific about the two or three things that will actually get a pull request merged or closed.

grantdesk is the largest of the five repositories in the Egeria program and the only one with meaningful ongoing maintenance cost. That is not incidental. An application with user accounts, a database, and a compliance calendar people rely on carries an obligation that a data-processing script does not. Most of the rules below exist to keep that obligation bounded.

---

## Before you write code

**Read [`docs/NON-GOALS.md`](docs/NON-GOALS.md) first.** It is not a formality. Roughly half of well-intentioned feature contributions to a tool like this one are for features this tool has already decided not to have, and there is no worse experience in open source than spending a weekend on something that gets closed with a link. If your idea is on that list and you think the list is wrong, open an issue arguing that case before you build. Changing a non-goal is a legitimate thing to propose. Doing an end-run around it in a pull request is not.

**Open an issue for anything that changes the schema, adds a dependency, or adds a route.** Small fixes, documentation, tests, and template contributions can go straight to a pull request.

---

## Local setup

Requires Node 20 or newer and pnpm 9 or newer.

```bash
git clone https://github.com/egeria-corporation/grantdesk
cd grantdesk
pnpm install
cp .env.example .env          # set SESSION_SECRET to anything for local work
pnpm db:migrate:local         # applies migrations to the local emulated D1
pnpm seed:demo                # loads the demo workspace and three client orgs
pnpm dev                      # http://localhost:8787
```

`pnpm dev` runs the Workers target under Wrangler's local emulation. `pnpm dev:node` runs the same application on the Node target used by the Docker path. **If you change anything in the data or platform layer, run both.** That is the single most common way a change breaks half the users.

Before you push:

```bash
pnpm check      # biome lint + format check + tsc --noEmit
pnpm test       # vitest, both targets
```

CI runs exactly these. A red CI badge on the README is a bug in its own right.

---

## The rules that are actually enforced

### 1. Tenancy is not negotiable

Every domain table carries `workspace_id` and `org_id`. Every read and every write goes through the scoped repository layer, which takes a tenancy context and applies the predicate itself. Application code does not hand-write a `WHERE org_id = ?`.

There is a test that walks the schema and fails if a domain table is missing either column. There is a second test that fails if a raw query in `src/` references a domain table outside the repository layer. Do not disable these. If you have a case they get wrong, say so in an issue and we will fix the test rather than the invariant.

A pull request that can leak one client's data into another client's view will be closed, not reviewed. This is the failure mode that ends a tool like this, because a consultant who shows the wrong client's declined proposal to a board has a professional problem, not a software problem.

### 2. Both deployment targets, always

Cloudflare Workers with D1, and Node with SQLite in Docker. One schema, one migration set, one test suite that runs against both.

Anything Workers-specific goes behind the interfaces in `src/platform/`: scheduled work, object storage, key-value caching, outbound email. Each has a Workers implementation and a Node implementation. `env.MY_KV.get(...)` in a route handler is a bug even when it works.

### 3. Migrations are forward-only and never edited after they ship

Numbered SQL files under `migrations/`. Once a migration is on `main`, it is immutable. If it was wrong, add another one. People are running this against real client records and there is no re-run.

Every migration needs a test that applies it to a fixture database and asserts the resulting shape. Destructive migrations, meaning anything that drops or narrows a column, need an explicit note in the pull request describing what data is lost and what the operator should back up first.

### 4. Server-rendered HTML, minimal JavaScript

The UI is Hono JSX rendered on the server. Forms are HTML forms that POST. There is a small amount of vanilla JavaScript for keyboard shortcuts and inline status changes, and it is progressive enhancement: every action has a no-JavaScript path that works.

We are not adding a client-side framework. This is not aesthetic preference. A grant consultant needs to open this on hotel wifi the night before a deadline, and a server-rendered page is the version that loads. It also means the whole application is legible to someone reading the source, which is most of what makes an open-source tool maintainable by strangers.

### 5. Tests use realistic fixtures

Fixtures use real funder names, real assistance listing numbers, real award shapes, and dates that exercise the parts that break: quarter boundaries, leap days, fiscal years that are not calendar years, awards that end mid-quarter, no-cost extensions that move a period end after obligations have already been generated.

Do not add real client data to fixtures, ever, including your own clients and including data you have permission to use. Use the fictional organizations already in `test/fixtures/`: Riverside Housing Trust, Bay Area Literacy Project, Cascade Watershed Alliance.

The recurrence engine in particular is tested as a pure function with table-driven cases. If you touch it, add cases before you add behavior.

### 6. Compliance templates need citations

A pull request adding or changing a built-in obligation template must include, for every obligation it generates, the citation it derives from (a CFR section, a standard form's instructions, or a named agency policy document) and the date the contributor verified it. Templates without citations get closed.

The reason is direct: someone will rely on a generated deadline. A template that says "final report due 120 days after the period of performance" and cites 2 CFR 200.344 can be checked and, when the rule changes, found and fixed. One that says it because a contributor remembered it that way cannot.

Templates describing a specific funder's actual requirements are welcome and useful. Templates that are guesses about a funder's requirements are worse than nothing.

---

## Pull requests

- Branch from `main`, one logical change per pull request.
- Conventional commit prefixes: `feat:`, `fix:`, `docs:`, `test:`, `refactor:`, `chore:`.
- Describe what changed and, for anything touching the schema or the recurrence engine, what an existing operator will experience on upgrade.
- New user-facing behavior needs a `CHANGELOG.md` entry.
- Screenshots for UI changes, since we have no visual regression testing and are not adding it.

Reviews aim for a response within a week. A pull request that adds a dependency should say what it replaces and why the alternative of writing thirty lines is worse. The dependency budget for this repository is deliberately tight, because every dependency is a thing an operator running a five-year-old deployment has to worry about.

---

## Reporting bugs

Include the deployment target (Workers or Docker), the version, what you expected, and what happened. For anything in the compliance calendar, include the rule configuration and the award dates, because almost every bug in that area is a date-boundary bug and the dates are the reproduction.

**Do not paste real client data into an issue.** Reduce it to fictional organizations and shifted dates that still reproduce the problem. If you genuinely cannot reproduce it without real data, say so and we will find another way.

Security issues go to the address in [`SECURITY.md`](SECURITY.md), not to the public issue tracker. Anything touching tenancy isolation, session handling, or the webhook signature check is a security issue.

---

## Upstream first

Where grantdesk would need a fix or an extension in a project it depends on, we open the pull request upstream first and note it in [`docs/research/prior-art.md`](docs/research/prior-art.md). We do not fork or re-implement something a community project already does well in order to own it. This community's endorsement is worth more than the code.

---

## Code of conduct and licensing

Participation is governed by [`CODE_OF_CONDUCT.md`](CODE_OF_CONDUCT.md). Contributions are accepted under the Apache License 2.0, as stated in that license's Section 5. There is no separate contributor license agreement to sign.
