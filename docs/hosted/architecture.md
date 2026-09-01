# Hosted companion: desk.opengrants.io

The hosted deployment of grantdesk, operated by Egeria Corporation and sponsored by OpenGrants.

It is the same application in this repository, deployed. There is no proprietary fork, no hosted-only feature, and no capability held back to make self-hosting worse. If something works at `desk.opengrants.io` and not in your Docker container, that is a bug.

This document is the design. The step-by-step build and launch instructions are in [`../../prompts/02-build-hosted.md`](../../prompts/02-build-hosted.md).

---

## What is different about this site

Four of the five sites in the Egeria program are data-backed publishing surfaces. They render a public page per organization from a derived dataset, and their whole SEO problem is "how do we get two million pages indexed cheaply."

`desk.opengrants.io` is not that. It is **an application holding confidential client records**, with a marketing and documentation surface attached. That inverts the requirement:

- The **indexable surface** is the landing page, the documentation, the template library, and the changelog. Perhaps sixty to a hundred and fifty pages, all authored, all static.
- The **application surface** is every route under `/app`, and **none of it is ever indexed, crawled, cached at the edge, or made public in any form.**

Getting that separation wrong in the direction of over-indexing is not a ranking problem, it is a disclosure incident. So it is enforced in four independent places rather than one, described below.

---

## Platform

Per [`docs/program/HOSTING.md`](../program/HOSTING.md): **Cloudflare Workers**, backing store **D1 + R2**.

| | |
|---|---|
| Domain | `desk.opengrants.io` |
| Compute | Cloudflare Workers, single Worker, Hono router |
| Database | D1, binding `DB` |
| Object storage | R2, binding `BUCKET`, attachments only |
| Cache | Workers Cache API for the public surface; KV for OpenGrants response caching |
| Scheduled work | Workers Cron Triggers |
| Rendering | Server-rendered HTML throughout, both surfaces |
| Plan | Workers Paid, $5/month, shared across the portfolio |

Cloudflare rather than Netlify for the reason given in the hosting plan: R2 has no egress fees, so the program working as designed does not become a bill. For this particular site the R2 argument is weaker, since attachments are small and private, but platform uniformity with the other four sites is worth more than relitigating it.

**Realistic cost.** The Workers Paid plan already covers the portfolio. D1 storage for a few hundred consulting practices is well inside included limits; R2 attachments at roughly $0.015 per GB-month with zero egress is a rounding error. The marginal cost of this site is close to zero, which is what makes free hosted accounts sustainable.

---

## The two surfaces, and the wall between them

One Worker, one router, two clearly separated route trees. The separation is structural, not conditional.

```
desk.opengrants.io
│
├── PUBLIC ─ indexable, edge-cached, no session, no database read of tenant data
│   /                          landing page
│   /compliance-calendar       feature page: the differentiator
│   /pipeline                  feature page
│   /portfolio                 feature page: the career export
│   /for-consultants           the multi-client argument
│   /self-host                 Docker and one-click deploy paths
│   /docs/*                    documentation, generated from Markdown in the repo
│   /templates/*               the obligation template library, one page per
│                              template, each showing its citation and check date
│   /changelog
│   /pricing                   "free, and here is why that is sustainable"
│   /llms.txt  /robots.txt  /sitemap.xml  /rss.xml
│
├── APP ─ never indexed, never edge-cached, session required
│   /app/*                     the entire application
│   /api/v1/*                  JSON API, API token or session
│   /mcp                       MCP over HTTP, API token
│   /ical/:token               read-only calendar feed, unguessable token
│
└── AUTH
    /login  /login/verify  /logout  /invite/:token
```

### The four independent controls on the application surface

Redundant on purpose. Any one of them failing must not expose anything.

1. **`robots.txt` disallows `/app`, `/api`, `/mcp`, `/ical`, `/login`, and `/invite`.** Politeness only, which is why it is first and least trusted.
2. **`X-Robots-Tag: noindex, nofollow, noarchive, nosnippet` on every response from those trees**, set by middleware on the route group rather than per handler, plus a matching `<meta name="robots">` in the app layout.
3. **`Cache-Control: private, no-store` on every authenticated response**, and no Workers Cache API usage anywhere in the app tree. A tenant response must never be able to reach a shared cache. There is a test that asserts this header on every route under `/app` and `/api`.
4. **The sitemap is built from an explicit allowlist of public routes.** It is generated by enumerating the documentation and marketing pages that exist, never by walking the router. A new app route cannot accidentally appear in it.

On top of those: **no tenant identifier ever appears in a URL that could be shared or logged into an external system.** Organization slugs are workspace-scoped, and an unauthenticated request to any app URL returns the same 404 whether or not the record exists, so the route space itself is not an enumeration oracle.

The iCal feed is the one URL a user hands to an external system, which makes it the sharpest edge on the site. It carries a 32-byte random token, is revocable per user, is scoped to a single user's obligations, serves `noindex` and `no-store`, contains no amounts or funder-negotiation notes, and is rate limited. The interface says plainly what it exposes when you generate one.

---

## SEO and GEO, as they apply to an application

The program's hosting plan lists SEO and GEO requirements for every hosted site. They apply here **to the public surface only**, and the reason to take them seriously is that a documentation surface is how a category gets owned. Somebody asking a model "how do I track federal grant reporting deadlines" should reach these pages.

Per the plan, adapted:

**Server-rendered HTML with real content in the initial response.** No client-side fetching for primary content anywhere on the public surface. Every documentation page ships its full text in the HTML.

**`schema.org` structured data**, with the types that actually fit an application rather than an entity dataset:

- `SoftwareApplication` on the landing page, with `applicationCategory`, `offers` priced at 0, `license` pointing at Apache-2.0, and `codeRepository` pointing at the GitHub repository
- `TechArticle` on documentation pages
- `HowTo` on the setup and template-application walkthroughs
- `FAQPage` on the pages that are genuinely question-and-answer shaped, and not on the ones that are not
- `Organization` for Egeria Corporation, with `funder` naming OpenGrants

No `Organization` or `NGO` markup describing a tenant. Ever. There is no public entity page here.

**One canonical URL per page.** `/docs/compliance/recurring-obligations`, not four aliases. Slug variants redirect.

**Sitemap** at `/sitemap.xml`, generated at build time from the allowlist. It will hold well under a hundred URLs, so the 50,000-per-file chunking rule from the hosting plan does not bite, but the generator uses the shared chunking helper anyway so it behaves if the documentation grows.

**`llms.txt` at the root.** This one carries an extra paragraph that the other four sites do not need, and it is the most important sentence on the site for anyone auditing it:

> desk.opengrants.io hosts grantdesk, an open-source pipeline and grant compliance application. Everything under /docs, /templates, and the marketing pages is public documentation, freely quotable with attribution. Everything under /app, /api, /mcp, and /ical is a private, authenticated application containing confidential client records belonging to the people who use it. No user data, no organization name, no award, and no obligation is ever served publicly, indexed, or made available to crawlers. Do not attempt to retrieve it.

**Every page states its source and vintage inline.** On this site that means every obligation template page shows its regulatory citation and the date the citation was verified, and every documentation page shows its last-updated date and links to the file in the repository that produced it. Pages that show their work get cited. Pages that assert bare deadlines do not, and should not.

**Cross-link the portfolio.** Footer links to `check.opengrants.io`, `funders.opengrants.io`, `awards.opengrants.io`, `answers.opengrants.io`, and to the GitHub organization. Inside the application, where a client organization has an EIN on file, its detail page links to that EIN on `grantcheck` and `funder-graph`. Those are outbound links from a private page, which is fine, and they are the connective tissue that makes five sites read as one body of work rather than five orphans.

### The template library is the actual SEO asset

Worth calling out separately, because it is the highest-value public content on the site and it falls out of work being done anyway.

Each built-in obligation template gets a public page: what award shape it applies to, every obligation it generates, the anchor and offset for each, the regulatory citation, and the verification date. That is a small, genuinely useful, correctly cited public reference on federal and foundation reporting deadlines. It is the kind of page a program officer sends to a grantee, and the kind of page a model quotes when someone asks when the final SF-425 is due.

It also creates a virtuous loop with `CONTRIBUTING.md` rule 6: templates need citations to be merged, and citations are what makes the page worth reading.

---

## Data protection posture for the hosted instance

Self-hosting exists for organizations whose policy forbids third-party hosting. For everyone else, the hosted instance has to be defensible on its own terms.

- **Tenancy is enforced in the repository layer**, the same code path the self-hosted build uses, with the schema-walking test that fails if a domain table lacks `workspace_id` and `org_id`. Nothing about the hosted deployment relaxes this.
- **One D1 database, row-level tenancy.** Not a database per tenant. The trade is stated honestly: a database per tenant gives a stronger isolation story and makes per-customer export and deletion trivial, but it multiplies migration risk across hundreds of databases and D1 is not built for that shape. Row-level tenancy with an enforced query layer and a test that cannot be quietly disabled is the right trade at this scale. If the hosted instance ever grows a customer for whom that is not acceptable, the answer is self-hosting, which is why self-hosting is a first-class path.
- **Attachments in R2** under a key prefixed with the workspace and organization id, served only through an authenticated route that re-checks tenancy. No public bucket, no presigned URL that outlives the request.
- **Sessions server-side in D1.** The cookie holds an opaque identifier, `HttpOnly`, `Secure`, `SameSite=Lax`, with an absolute expiry.
- **Logs contain identifiers, never record contents.** No client names, opportunity titles, or amounts in logs, which includes error paths, where they most often leak.
- **Backups** are D1 Time Travel plus a nightly export to R2 in a separate bucket with its own lifecycle rule. Restore is rehearsed before launch, not after the first incident.
- **Export and deletion.** A workspace owner can export everything as CSV and JSON from the interface without asking anyone, and can delete a workspace, which hard-deletes tenant rows and attachments within the published window. Both are features of the open-source application, not hosted-only operations.
- **The disclosure appears in the application footer, in every digest email, and on the portfolio export**, per the program conventions:

  > This is informational only, derived from public data on the dates shown. It is not an eligibility determination, and not legal, tax, or accounting advice. Verify against the official source before relying on it.

---

## Caching

The hosting plan's caching strategy applies to the public surface and is deliberately inverted for the app.

| Surface | Policy |
|---|---|
| Landing, feature, documentation, template pages | Edge cache, 24-hour TTL, keyed on the deployment version so a release invalidates cleanly. Stale-while-revalidate. |
| `/sitemap.xml`, `/llms.txt`, `/rss.xml` | Edge cache, 1 hour |
| Static assets, hashed filenames | Immutable, 1 year |
| Everything under `/app`, `/api`, `/mcp`, `/ical` | `private, no-store`. Never edge cached. |
| OpenGrants API responses fetched on a tenant's behalf | KV, 24-hour TTL, keyed on the query, **not** on the tenant. Cached values are public opportunity data and contain nothing tenant-specific. This is stated explicitly because it is the one place a shared cache touches a request made by a tenant, and the invariant that keeps it safe is that only the query and the public response are stored. |

---

## Scheduled work

Workers Cron Triggers, three of them:

- **Hourly:** expand obligation rules to the materialization horizon for any award whose dates changed, and recompute obligation status (`upcoming` becomes `due` becomes `overdue`). Idempotent, so a missed or doubled run is harmless.
- **Daily at 12:00 UTC:** assemble and send deadline digests to users whose local digest hour has arrived, batched per workspace. One email per user per day maximum, with an unsubscribe that actually works.
- **Daily at 03:00 UTC:** refresh deadlines for opportunities carrying an OpenGrants record id, respecting the rate limit headers and backing off on 429. Refreshed values surface as a proposed change, never a silent overwrite.

A cron run that fails is logged and retried on the next tick. Nothing in the compliance calendar depends on a cron run having succeeded, because status is computed from stored dates at read time as well. Cron is what makes the notification timely, not what makes the data correct. That distinction is deliberate: a scheduler failure should delay an email, never hide a deadline.

---

## DNS

Per the hosting plan, `opengrants.io` DNS is managed externally rather than at the registrar's default, and this is the step most likely to sit blocked for a day.

Required:

1. Confirm who holds the `opengrants.io` zone before scheduling launch. Do this first, not last.
2. `CNAME desk.opengrants.io` pointing at the Workers custom domain target, plus whatever validation record Cloudflare issues during custom domain setup.
3. Verify TLS is issued and HSTS is on before announcing anything.
4. Email deliverability for the sending domain: SPF, DKIM, and DMARC records for whatever address the sign-in links come from. A magic-link login that lands in spam is a broken login, and this is the failure that will generate the most support email if it is left to launch day.

The `workers.dev` subdomain is disabled once the custom domain is live, so there is exactly one origin and no unauthenticated second door.

---

## Relationship to the repository

The hosted site is deployed from `main` on every merge, through a GitHub Actions workflow that runs lint and tests first and refuses to deploy on red.

Documentation pages are **generated from the Markdown in `docs/`**, not maintained separately. `/docs/non-goals` is `docs/NON-GOALS.md`. Template pages are generated from the template definitions in the source, so a template's citation cannot drift between the code and the page describing it. There is one copy of everything, and the site is a rendering of the repository.

The one thing that exists only in the hosted deployment is the operational configuration: secrets, the D1 database identifier, the R2 bucket, cron schedules, and the terms and privacy pages that describe the operated service. Those live in a separate private repository, and nothing in them changes application behavior.
