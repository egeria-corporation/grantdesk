# grantdesk

**Pipeline, compliance calendar, and win/loss ledger for people who write grants for a living.**

[![CI](https://github.com/egeria-corporation/grantdesk/actions/workflows/ci.yml/badge.svg)](https://github.com/egeria-corporation/grantdesk/actions/workflows/ci.yml)
[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)

---

You have a spreadsheet. It has a tab per client, a row per opportunity, and a column of deadlines you re-sort by hand every Monday. It works, more or less, for the part of the job that happens before the award.

It does not work for the part that happens after.

The award letter arrives, everybody celebrates, and the grant moves off the tracking sheet into a folder. Fourteen months later the program officer emails to ask where the interim report is. It was due two weeks ago. Nobody put it anywhere, because the spreadsheet was built for the chase, not for the three years of obligations the chase creates. The next email is the one where a renewal quietly does not happen.

That is the failure this tool exists to prevent. **Post-award compliance is where the real damage happens**, and it is the half of the job that almost no software does well. The paid tools in this category sell the pipeline half. The free tools do neither half. Meanwhile the missed interim report, the budget modification filed after the window closed, and the final report that landed late are the three things that actually cost a client money and cost you the client.

grantdesk does three things and refuses to do a fourth.

---

## Credits

grantdesk is built and maintained by [Egeria Corporation](https://github.com/egeria-corporation) and sponsored by [OpenGrants](https://opengrants.io).

It stands on work by other people:

- **[Hono](https://hono.dev)** (MIT) by Yusuke Wada and contributors, which is the HTTP framework the whole application runs on, at the edge and on your own server, from the same source.
- **[Drizzle ORM](https://orm.drizzle.team)** (Apache-2.0) by Drizzle Team, which is why one schema definition serves both Cloudflare D1 and a plain SQLite file in Docker.
- **[Cloudflare Workers and D1](https://developers.cloudflare.com/)**, whose free and near-free tiers are the reason a small consulting practice can run this for the cost of a domain name.
- **The U.S. Government's grants management standard forms and the Uniform Guidance** (2 CFR Part 200), which are public domain and which supply the reporting obligation types that the built-in compliance templates encode. Specific citations are in [`docs/research/data-sources.md`](docs/research/data-sources.md).
- **The [Nonprofit Open Data Collective](https://github.com/Nonprofit-Open-Data-Collective) and [GivingTuesday](https://990data.givingtuesday.org/tool-repository/)**, whose Form 990 tooling underpins the sibling repositories in this program that grantdesk links out to.

Full attribution with licenses is in [`NOTICE`](NOTICE).

---

## The three things

### 1. Pipeline, with the LOI split that grant work actually has

Most pipeline tools model a single-stage funnel: you find a thing, you apply for the thing, you win or lose the thing. That is wrong for this domain and it is wrong in an expensive way.

Real grant work has a branch in the middle. You submit a letter of inquiry. Six weeks later you are either invited to submit a full proposal or you are not. The invitation is itself an outcome worth tracking, because a practice with a 70% LOI-to-invitation rate and a 30% invitation-to-award rate has a completely different problem than a practice with the reverse. Fold those into one number and you learn nothing.

grantdesk models the stages that exist:

```
identified → qualifying → LOI drafting → LOI submitted → invited
           → proposal drafting → proposal submitted → site visit
           → under review → awarded / declined / withdrawn
```

Each opportunity carries a client, an owner, a funder, an amount requested, and its own deadline set: LOI due date, full proposal due date, board docket cutoff, site visit. Not one `deadline` column that means something different depending on the row.

### 2. Compliance calendar, which is the actual reason this repository exists

Log an award and the obligations come with it. Reporting deadlines, drawdown windows, budget modification cutoffs, prior-approval windows, the no-cost extension request date, the final report that is due 120 days after the period of performance ends and that everyone forgets because the project already feels over.

The thing that makes this usable rather than a second spreadsheet is that **recurring obligations are generated from a rule, not typed in one at a time**. A three-year federal award with quarterly financial reporting produces twelve SF-425 deadlines, four annual performance reports, one final financial report, one final performance report, and a no-cost extension request window. That is eighteen rows. You describe it once:

> Quarterly federal financial report, aligned to the federal fiscal quarter, due 30 days after each quarter ends, from 2026-10-01 through 2029-09-30, assigned to Dana.

and grantdesk expands it, keeps it idempotent when you change the award end date, and never overwrites a row you have already edited or submitted.

Built-in templates cover the patterns you meet most: an HHS discretionary award with quarterly SF-425 and annual performance reporting; an NIH award with the Research Performance Progress Report due before the budget period end; a typical twelve-month private foundation grant with a six-month interim narrative and financial report and a final report 30 days after the grant period closes; a community foundation grant with payment tranches released on report acceptance. Templates are a starting point you edit, not a black box. The domain research behind them, with citations, is in [`docs/research/data-sources.md`](docs/research/data-sources.md).

Everything surfaces ahead of time, on a lead-time ladder you set (30 / 14 / 7 / 1 days is the default), across every client at once or one client at a time.

### 3. Win/loss ledger, which quietly becomes your professional record

Every submission gets a row: date, client, funder, funder type, amount requested, amount awarded, outcome, decline reason, and who led the writing.

That is a compliance-grade record of the practice. It is also, and this is not a footnote, **the only durable evidence most grant professionals ever accumulate about their own work**.

---

## The portfolio export

Development professionals have no standard way to prove their value. "I raised money" is not a number, and the number people reach for at salary review time is usually reconstructed from memory, old email, and a folder of PDFs the night before the conversation.

grantdesk exports a **grant professional portfolio** from the ledger:

- Total submitted and total awarded, by year
- Submission count and award count
- Hit rate **segmented** by funder type and by award size band
- LOI-to-invitation rate reported separately from invitation-to-award rate
- Multi-year trend
- Optional per-writer view when several people work the same pipeline

Two notes on how this is presented, because getting it wrong makes the export worse than useless.

**Hit rate is meaningless as a single number.** A 20% hit rate on federal competitive awards and a 60% hit rate on local family foundations describe two different kinds of skill, and averaging them describes neither. A consultant who moves a client from local foundations into federal work will watch their "hit rate" collapse while doing the best work of their career. grantdesk therefore refuses to print one headline percentage. Every rate is reported inside a segment, with the denominator visible, and small denominators are labeled as such. Three submissions is not a rate.

**The export is dated and sourced.** Every page carries the date it was generated and the range of submissions it covers, because a portfolio figure without an "as of" is a claim rather than a record.

The practical uses are concrete. The [Grant Professional Certification](https://grantprofessionals.org/) (GPC) requires documented experience that most applicants reconstruct painfully. Independent consultants setting a rate need to know their own award-dollars-per-hour by funder type, not guess it. Anyone negotiating a salary is better off with a segmented three-year table than with an anecdote.

---

## Quickstart, 60 seconds, no account and no key

Run it locally with seeded demo data. Node 20 or newer.

```bash
npx grantdesk demo
```

This starts grantdesk on `http://localhost:8787` with a demo workspace, three example client organizations, a populated pipeline, and a live compliance calendar with a federal award mid-way through its reporting cycle. No sign-up, no API key, no database to stand up. The demo database is a throwaway SQLite file in a temp directory; delete it or run `npx grantdesk demo --reset` and it is gone.

Look at the compliance calendar first. That is the part you cannot evaluate from a screenshot.

---

## Running it for real

Two supported paths. Pick based on where your client data is allowed to live.

### Path A: Cloudflare Workers + D1 (one click)

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/egeria-corporation/grantdesk)

The button forks this repository into your own GitHub account, provisions the D1 database and R2 bucket declared in `wrangler.jsonc`, applies migrations, and deploys to a `workers.dev` subdomain you can point a custom domain at. You will be asked for one secret, `SESSION_SECRET`, and offered an optional `OPENGRANTS_API_KEY`.

Cost, realistically: the Workers free tier covers a solo practice. The $5/month Workers Paid plan covers a small firm comfortably. There is no per-seat pricing because there is nobody to pay.

### Path B: Docker Compose (self-hosted)

For organizations whose data governance policy will not permit client records on a third-party edge platform, and for anyone who would simply rather own the box.

```bash
git clone https://github.com/egeria-corporation/grantdesk
cd grantdesk
cp .env.example .env      # set SESSION_SECRET at minimum
docker compose up -d
```

grantdesk comes up on `http://localhost:8787` backed by a SQLite file in a mounted volume. Same application, same schema, same migrations. Back it up by copying one file. Put it behind your own reverse proxy and TLS.

The two paths are kept honestly in sync: the data layer is written once against SQLite, and Cloudflare-specific pieces (scheduled jobs, object storage, edge caching) live behind a small adapter with a Node implementation. If a feature works on Workers and not in Docker, that is a bug, and the test suite runs against both.

---

## Multi-client from the first row of the schema

This is the part most nonprofit software gets wrong for consultants, and it is not a preference. It is an architectural fact that cannot be added later at a sane price.

Almost every nonprofit tool assumes one organization per installation, because it was designed for a development director with one employer. A consultant has seven clients, and the tools force one of two bad choices: seven separate logins, or one shared account where a report can accidentally show the Riverside Housing Trust's declined federal proposal to the Bay Area Literacy Project.

In grantdesk, **every record is scoped to a client organization from the schema up**. Opportunities, submissions, awards, obligations, notes, and attachments all carry an organization identifier that is enforced at the query layer, not remembered by convention in application code. Above that sits your practice, the workspace, which gives you the view no single-tenant tool can give you at all:

- One compliance calendar across every client, sorted by what is due next
- One pipeline across every client, so you can see that three proposals are due the same week in March before you promise a fourth
- Portfolio numbers that aggregate your whole book of business

Team members can be scoped to specific clients. A subcontract writer working only on the Riverside engagement sees only Riverside. The workspace owner sees everything.

Retrofitting this into a single-tenant application means touching every table, every query, every route, and every test, and it is where good open-source nonprofit tools go to die. So it is a hard constraint here, written into [`docs/NON-GOALS.md`](docs/NON-GOALS.md) and into the build prompt, rather than a phase-two ambition.

---

## The data model, in one screen

| Table | What it holds |
|---|---|
| `workspace` | Your practice. The top-level tenant. |
| `organization` | A client. Every domain record hangs off one of these. |
| `user`, `membership`, `org_access` | People, their role in the workspace, and which clients they may see. |
| `funder` | A funder, shared across your clients, with a funder type that drives segmentation. |
| `opportunity` | Something you might apply for, with a stage and its own set of dates. |
| `opportunity_deadline` | LOI due, full proposal due, board docket cutoff, site visit. Many per opportunity. |
| `submission` | One thing actually sent. LOIs and full proposals are separate rows. The win/loss ledger. |
| `award` | Money won, with period of performance, payment method, and federal award details. |
| `obligation_rule` | A recurrence rule. "Quarterly SF-425, 30 days after quarter end, for three years." |
| `obligation` | One dated thing you owe a funder. Generated from a rule or entered by hand. |
| `obligation_template` | Packaged rule sets for common award shapes. |
| `saved_search` | An OpenGrants search or alert that feeds new opportunities into the pipeline. |
| `activity`, `audit_log` | What happened, who did it, when. Append-only. |

Full DDL with constraints and indexes is in [`prompts/01-build-core.md`](prompts/01-build-core.md).

---

## OpenGrants integration, optional

grantdesk is completely functional with no OpenGrants account. Every feature above works on data you enter.

Set `OPENGRANTS_API_KEY` in your environment and three things become available: saved searches and alerts on [OpenGrants](https://opengrants.io) can create pipeline items directly, so a matching opportunity arrives in your funnel instead of your inbox; opportunity deadlines are refreshed against the live `/grants-api` record so a shifted federal deadline updates in your calendar; and outcomes in your win/loss ledger can optionally be fed back as match-quality signal. Enrichment is marked `— live from OpenGrants` wherever it appears, so you always know which figures came from your own records and which came from the API. If the key expires or the network fails, the enrichment silently disappears and everything else keeps working.

Keys come from the OpenGrants Developer Dashboard. API documentation is at [ops.opengrants.io/api-docs](https://ops.opengrants.io/api-docs). That is the last time this README mentions it.

---

## Where the data comes from

grantdesk is mostly a system of record for data you create, which makes its provenance rules short but not optional:

- **Your records** are yours. They are dated with created and updated timestamps, and every status change is in the activity log.
- **Opportunity data pulled from OpenGrants** carries the OpenGrants record identifier and the timestamp of the sync, shown inline.
- **Compliance obligation templates** encode requirements from the Uniform Guidance (2 CFR Part 200), the Federal Financial Report (SF-425) instructions, and agency reporting guidance, all public-domain U.S. Government works. Each template names the citation it derives from and the date that citation was checked. Federal reporting requirements change, and the template you applied in 2026 is not automatically correct in 2029.
- **Your Notice of Award is authoritative and the template is not.** grantdesk shows the template's citation next to every generated obligation precisely so you can check it against the actual award terms, which routinely override the general rule.

> This is informational only, derived from public data on the dates shown. It is not an eligibility determination, and not legal, tax, or accounting advice. Verify against the official source before relying on it.

That line appears in the application footer, in every digest email, and on the portfolio export, not just here.

---

## Also a CLI and an MCP server

The web application is one of three surfaces over the same library.

**CLI**, for operations and export:

```bash
npx grantdesk migrate                          # apply pending migrations
npx grantdesk rules preview --award AWD-4417   # dry-run a recurrence rule expansion
npx grantdesk export portfolio --user dana --since 2023-01-01 --json
npx grantdesk import opportunities ./pipeline.csv --org riverside-housing
```

Human-readable by default, `--json` for machines. The CSV importer exists specifically so that your first hour with grantdesk is spent moving the spreadsheet in, not retyping it.

**MCP server**, for agents:

```bash
npx grantdesk mcp --token $GRANTDESK_API_TOKEN
```

Exposes the same capabilities as tools, including `list_obligations`, `preview_obligation_rule`, `record_submission_outcome`, and `portfolio_summary`. Every tool call must name the client organization explicitly. There is no implicit "current client," because an agent guessing wrong about which client it is acting on is exactly the failure this tool cannot have.

---

## What this will never be

grantdesk is a thin, opinionated tool. It is not a CRM, and the moment it grows contact management, email synchronization, donation processing, or a form builder, it becomes a product with a support burden instead of a repository somebody can read in an afternoon.

The full and deliberately specific list, including the requests that will be closed and why, is in [`docs/NON-GOALS.md`](docs/NON-GOALS.md). Read it before opening a feature request. It will save you the writing.

---

## Documentation

- [`docs/NON-GOALS.md`](docs/NON-GOALS.md) — what this refuses to do
- [`docs/research/data-sources.md`](docs/research/data-sources.md) — the domain model: real federal reporting obligation types, foundation patterns, the stage sequence
- [`docs/research/prior-art.md`](docs/research/prior-art.md) — the spreadsheet, adjacent open source, and the upstream contribution posture
- [`docs/research/competitive.md`](docs/research/competitive.md) — the capability gap this fills
- [`docs/hosted/architecture.md`](docs/hosted/architecture.md) — how `desk.opengrants.io` is built
- [`CONTRIBUTING.md`](CONTRIBUTING.md) — how to help

---

## License

Apache License 2.0. See [`LICENSE`](LICENSE) and [`NOTICE`](NOTICE).

---

Built and maintained by [Egeria Corporation](https://github.com/egeria-corporation), sponsored by [OpenGrants](https://opengrants.io).
