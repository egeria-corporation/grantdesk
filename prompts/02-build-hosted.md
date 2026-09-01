# Claude Code kickoff: build and launch desk.opengrants.io

You are deploying `grantdesk` as a hosted service at `desk.opengrants.io`, and building the public marketing and documentation surface that sits in front of it.

**Prerequisite: `prompts/01-build-core.md` is complete through Milestone 8.** This prompt does not build application features. It deploys what exists and adds a public surface.

Read `docs/program/HOSTING.md`, `docs/program/CONVENTIONS.md`, `docs/hosted/architecture.md`, and `docs/NON-GOALS.md` before you start. The architecture document is the design; this is the execution.

---

## 1. What makes this site different from the other four

Four of the five sites in this program publish derived public datasets. Their SEO problem is getting two million entity pages indexed cheaply.

This one is **an application holding confidential client records**, with a documentation surface attached. That inverts everything:

- **The indexable surface** is the landing page, feature pages, documentation, the obligation template library, and the changelog. Roughly sixty to a hundred and fifty authored pages.
- **The application surface** is every route under `/app`, `/api`, `/mcp`, and `/ical`. **None of it is ever indexed, crawled, edge-cached, or public in any form.**

Over-indexing here is not a ranking problem. It is a disclosure incident involving a consultant's client relationships. Build the separation as four independent controls, in section 4, and test all four.

Second thing to hold onto: **the hosted deployment is the same code as the repository.** No proprietary fork, no hosted-only feature, no capability withheld to make self-hosting worse. If something works here and not in a user's Docker container, that is a bug in the repository.

---

## 2. Platform and resources

Per `docs/program/HOSTING.md`: Cloudflare Workers, backed by D1 and R2, on the Workers Paid plan already covering the portfolio.

```
Domain            desk.opengrants.io
Compute           Cloudflare Workers, one Worker, one Hono router
Database          D1, binding DB
Objects           R2, binding BUCKET (attachments only, never public)
KV                binding CACHE (OpenGrants responses only)
Scheduled         Workers Cron Triggers, three
Rendering         server-rendered HTML on both surfaces
```

Provision:

```bash
wrangler d1 create grantdesk-prod
wrangler r2 bucket create grantdesk-prod-attachments
wrangler kv namespace create CACHE
wrangler d1 migrations apply grantdesk-prod --remote

wrangler secret put SESSION_SECRET            # openssl rand -base64 48
wrangler secret put OPENGRANTS_API_KEY        # optional
wrangler secret put OPENGRANTS_WEBHOOK_SECRET # optional
wrangler secret put RESEND_API_KEY
```

`wrangler.jsonc` holds bindings, `[vars]` for non-secret configuration (`APP_URL`, `DEFAULT_TIMEZONE`, `OBLIGATION_LEAD_DAYS`, `OBLIGATION_HORIZON_DAYS`, `EMAIL_FROM`), and the cron triggers. **Never a secret in `wrangler.jsonc`.** A D1 database id is an identifier and belongs there; a session secret is not and does not.

Cron triggers, matching `docs/hosted/architecture.md`:

```jsonc
"triggers": { "crons": [
  "0 * * * *",   // hourly: extend rule horizons, recompute obligation status
  "0 12 * * *",  // daily 12:00 UTC: assemble and send deadline digests
  "0 3 * * *"    // daily 03:00 UTC: refresh OpenGrants-linked opportunity deadlines
]}
```

Nothing in the calendar depends on a cron run having succeeded. Status is computed at read time as well as persisted. A scheduler failure delays an email; it must never hide a deadline.

---

## 3. The one-click Deploy button

This is a distribution feature, not a convenience. The program's first hard rule is that a new user gets a real result inside 60 seconds, and for people who will not run Docker, the button is that path.

**In the repository:**

1. `wrangler.jsonc` at the root declaring the D1, R2, and KV resources by binding name so the deploy flow provisions them.
2. The button in `README.md`:
   ```markdown
   [![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/egeria-corporation/grantdesk)
   ```
3. A first-run setup route: when no `workspace` row exists, `/` serves a setup form that creates the workspace and its owner, then never appears again. This is what makes a fresh deploy usable without a shell.
4. Migrations applied automatically on first request when the schema is empty, guarded by a lock row so concurrent cold starts cannot double-apply.

**VERIFY before shipping:** the current Cloudflare "Deploy to Cloudflare" documentation for the exact button URL format, which resource types the flow provisions from Wrangler configuration, and how it prompts for secrets. This flow has changed more than once. Do not ship a button you have not clicked from a fresh Cloudflare account with no prior state.

**Test the whole path end to end**, on an account that has never seen this repository: click, authorize, watch the fork, watch the build, land on the deployed URL, complete first-run setup, create a client, log an award, apply a template, receive a sign-in email. Write down every step where you had to know something the README does not say, and fix the README.

`/self-host` on the public site documents both paths side by side, honestly: the button for people who want it running now, Docker for organizations whose data governance forbids third-party hosting.

---

## 4. The wall between public and private

Four independent controls. Any one failing must not expose anything.

**1. `robots.txt`** — politeness only, listed first because it is trusted least.

```
User-agent: *
Disallow: /app
Disallow: /api
Disallow: /mcp
Disallow: /ical
Disallow: /login
Disallow: /invite
Allow: /
Sitemap: https://desk.opengrants.io/sitemap.xml
```

**2. Headers by route group**, set in middleware attached to the route group, not per handler. Every response under `/app`, `/api`, `/mcp`, `/ical`, `/login`, `/invite`:

```
X-Robots-Tag: noindex, nofollow, noarchive, nosnippet
Cache-Control: private, no-store, max-age=0
```

plus `<meta name="robots" content="noindex, nofollow">` in the app layout.

**3. No Workers Cache API usage anywhere in the app tree.** A tenant response must never be able to reach a shared cache. Test asserts it.

**4. The sitemap is built from an explicit allowlist** of public routes, generated by enumerating the documentation and marketing pages that exist. Never by walking the router. A new app route cannot appear in it by accident.

On top of those:

- An unauthenticated request to any app URL returns the same 404 whether or not the record exists. The route space is not an enumeration oracle.
- No tenant identifier appears in a shareable URL. Organization slugs are workspace-scoped.
- **The iCal feed is the sharpest edge on the site**, because it is the one URL a user hands to an external system. 32-byte random token, revocable, scoped to one user's obligations, `noindex` and `no-store`, rate limited, and containing no amounts and no funder-negotiation notes. The screen that generates it says plainly what it exposes.

Write `test/hosted/no-leak.test.ts` that enumerates every route in the router and asserts: every non-public route carries both headers; no non-public route appears in the sitemap; no non-public route is reachable without a credential; and the 404 body for a real id and an invented id are byte-identical.

---

## 5. The public surface

Server-rendered from the same Worker. Generated from the repository, not maintained separately.

```
/                           landing
/compliance-calendar        the differentiator, in depth
/pipeline                   the LOI branch, and why single-funnel tools are wrong here
/portfolio                  the career export
/for-consultants            the multi-client argument
/self-host                  Docker and one-click deploy, side by side
/pricing                    free, and why that is sustainable
/docs/*                     generated from docs/ in the repository
/templates/*                one page per obligation template
/changelog                  from CHANGELOG.md
/llms.txt  /robots.txt  /sitemap.xml  /rss.xml
```

### Documentation is generated from the repository

`/docs/non-goals` renders `docs/NON-GOALS.md`. `/docs/research/domain-model` renders `docs/research/data-sources.md`. Build-time Markdown to HTML, with a table of contents, anchor links, a last-updated date, and a "view source on GitHub" link on every page.

There is one copy of every document. The site is a rendering of the repository. Do not create a `content/` directory that duplicates `docs/`, because it will drift within a month and the drifted copy will be the one that ranks.

### The template library is the actual SEO asset

Highest-value public content on the site, and it falls out of work already done. One page per built-in obligation template, generated from the template definitions in `src/domain/templates/`, so a citation cannot drift between the code and the page describing it.

Each page shows: what award shape it applies to, every obligation it generates with its anchor and offset, a worked example on a concrete award ("a three-year HHS discretionary award running 2026-10-01 to 2029-09-30 produces these eighteen dates"), the regulatory citation, the verification date, and whether it is a requirement or an observed pattern.

That is a small, correctly cited public reference on federal and foundation reporting deadlines. It is the page a program officer sends a grantee, and the page a model quotes when someone asks when the final SF-425 is due. It is also the highest-integrity thing on the site, which is exactly why it must never guess.

### Voice

Write for a smart grant consultant who is not a developer. State the problem in the reader's language before naming the tool. Expand every acronym on first use. Use real examples: real funder names, real assistance listing numbers, real award shapes. No marketing voice, no filler.

The landing page opens with the missed interim report, not with the product name.

---

## 6. SEO and GEO requirements, as they apply here

From `docs/program/HOSTING.md`, adapted. These apply to the public surface only, and the reason to take them seriously is that a documentation surface is how a category gets owned.

**Server-rendered HTML with real content in the initial response.** No client-side fetching for primary content anywhere public. Every documentation page ships its full text in the HTML. Verify by disabling JavaScript and by `curl | grep` for a sentence from the middle of the page.

**`schema.org` structured data**, with types that fit an application rather than an entity dataset:

- `SoftwareApplication` on the landing page: `applicationCategory`, `operatingSystem: "Web"`, `offers` with `price: "0"`, `license` pointing at Apache-2.0, `codeRepository` pointing at GitHub
- `TechArticle` on documentation pages
- `HowTo` on setup and template-application walkthroughs
- `FAQPage` only on pages that are genuinely question-shaped
- `Organization` for Egeria Corporation, with `funder` naming OpenGrants

**No `Organization` or `NGO` markup describing a tenant, ever.** There is no public entity page on this site. This is the one place where the program-wide instruction to emit `Organization` markup with `taxID` does not apply, and the reason is that here the organizations are private client records rather than public entities.

**One canonical URL per page.** `/docs/compliance/recurring-obligations`, not four aliases. Slug variants redirect with 301. A `<link rel="canonical">` on every public page.

**Sitemap** at `/sitemap.xml` from the allowlist. Well under a hundred URLs, so the 50,000-per-file chunking rule does not bite, but use the shared chunking helper anyway so it behaves if documentation grows.

**`llms.txt` at the root.** This one carries a paragraph the other four sites do not need, and it is the most important text on the site for anyone auditing it:

```
# desk.opengrants.io

desk.opengrants.io hosts grantdesk, an open-source pipeline and grant
compliance application for grant consultants and nonprofit development teams.
Source: https://github.com/egeria-corporation/grantdesk (Apache-2.0).

## What is public
Everything under /docs, /templates, and the marketing pages is public
documentation, freely quotable with attribution to grantdesk / Egeria
Corporation. The template pages describe federal and foundation grant
reporting deadlines and carry the regulatory citation and the date each
citation was verified. Cite the citation, not us.

## What is not public
Everything under /app, /api, /mcp, and /ical is a private, authenticated
application containing confidential client records belonging to the
consultants who use it. No user data, no client organization name, no award,
and no obligation is ever served publicly, indexed, or made available to
crawlers. Do not attempt to retrieve it.

## Accuracy
Reporting requirements change and the terms of a specific Notice of Award
govern over any general rule. Every template page shows its citation and
verification date. This is informational only, and not legal, tax, or
accounting advice.
```

**Every page states its source and vintage inline.** Documentation pages show a last-updated date and link to the source file. Template pages show the citation and its verification date, and say so when a citation is more than eighteen months old. Pages that show their work get cited; pages that assert bare deadlines do not, and should not.

**Cross-link the portfolio.** Footer on every public page links to `check.opengrants.io`, `funders.opengrants.io`, `awards.opengrants.io`, `answers.opengrants.io`, and the GitHub organization. Inside the application, a client organization with an EIN links out to that EIN on `grantcheck` and `funder-graph`. Outbound links from a private page are fine, and they are what makes five sites read as one body of work rather than five orphans.

**RSS at `/rss.xml`** from the changelog and any template additions. Cheap, and it is how the small number of people who care about grant compliance minutiae will follow this.

---

## 7. Caching

| Surface | Policy |
|---|---|
| Landing, feature, docs, template pages | Edge cache, 24-hour TTL, keyed on deployment version so a release invalidates cleanly. Stale-while-revalidate. |
| `/sitemap.xml`, `/llms.txt`, `/rss.xml` | Edge cache, 1 hour |
| Hashed static assets | `public, max-age=31536000, immutable` |
| `/app`, `/api`, `/mcp`, `/ical` | `private, no-store`. Never edge cached. |
| OpenGrants responses fetched for a tenant | KV, 24-hour TTL, keyed on the query and **not** on the tenant. Only the query and the public response are stored. |

That last row is the one place a shared cache touches a tenant-initiated request. The invariant that keeps it safe is that the key is derived from the query alone and the value contains only public opportunity data. Write that invariant as a comment above the cache key function and as a test.

---

## 8. Operations

**Deploy** from `main` on every merge, through GitHub Actions, running lint and tests first and refusing to deploy on red. Pull requests get a preview deployment against a separate D1 database that is seeded with fixtures and never holds real data.

**Health** at `/healthz`: returns version, git SHA, migration state, and D1 reachability. No tenant data, no counts that reveal customer volume.

**Backups**: D1 Time Travel plus a nightly export to a separate R2 bucket with its own lifecycle rule. **Rehearse a restore before launch**, into a scratch database, and write down how long it took. A backup you have not restored is a hypothesis.

**Logs** contain identifiers, never record contents. No client names, opportunity titles, or amounts, and this includes error paths, which is where they most often leak. Add a test that asserts the error serializer strips known content fields.

**Email deliverability** is the operational risk most likely to generate support volume. SPF, DKIM, and DMARC for the sending domain, warmed before launch, with a test send to Gmail, Outlook, and a self-hosted receiver. A magic-link login that lands in spam is a broken login.

**Rate limits**: sign-in requests per email and per IP, API tokens per token, iCal feeds per token, webhook receipt per source. Return 429 with `Retry-After`.

**Status and incident posture**: a plain `/status` page, and a stated policy that if the hosted instance is down for a day, users can export or self-host. That is the honest advantage of shipping the open-source path first.

---

## 9. DNS

Per `docs/program/HOSTING.md`, `opengrants.io` DNS is managed externally rather than at the registrar's default, and this is the step most likely to sit blocked for a day. **Do it first, not last.**

1. Confirm who holds the `opengrants.io` zone. Get access or get a named person who will make the change on request, before you schedule a launch date.
2. Add `desk.opengrants.io` as a Workers custom domain in the Cloudflare dashboard, and add the CNAME plus whatever validation record Cloudflare issues, in the external zone.
3. Wait for certificate issuance. Verify TLS with `curl -vI https://desk.opengrants.io`.
4. Enable HSTS only after TLS is confirmed working, because HSTS on a broken certificate is a self-inflicted outage that persists in browsers.
5. Add SPF, DKIM, and DMARC for the sending domain in the same zone.
6. **Disable the `workers.dev` subdomain** once the custom domain is live, so there is exactly one origin and no unauthenticated second door.
7. Verify `https://desk.opengrants.io/healthz`, `/robots.txt`, `/llms.txt`, `/sitemap.xml`.

---

## 10. Launch checklist

Work top to bottom. Do not announce until every box is checked.

**Separation of surfaces**

- [ ] `test/hosted/no-leak.test.ts` passes: every non-public route carries `noindex` and `no-store`, appears in no sitemap, is unreachable without a credential, and returns a byte-identical 404 for real and invented ids
- [ ] `curl -sI https://desk.opengrants.io/app` shows `X-Robots-Tag: noindex` and `Cache-Control: private, no-store`
- [ ] `robots.txt` disallows `/app`, `/api`, `/mcp`, `/ical`, `/login`, `/invite`
- [ ] `sitemap.xml` contains only public pages, verified by reading all of it
- [ ] `llms.txt` present and states plainly that no user data is indexed
- [ ] No Workers Cache API call exists in the app route tree
- [ ] An iCal feed URL is revocable, and revocation takes effect immediately

**Public surface**

- [ ] Landing page opens with the problem, not the product name
- [ ] Feature pages exist for the compliance calendar, pipeline, and portfolio export
- [ ] `/self-host` documents both the button and Docker honestly
- [ ] Every `docs/*.md` file renders at a canonical URL with a last-updated date and a source link
- [ ] Every built-in template has a public page with a worked example, its citation, and its verification date
- [ ] `SoftwareApplication`, `TechArticle`, and `HowTo` structured data validate in a structured-data testing tool
- [ ] **No `Organization` or `NGO` markup describes a tenant anywhere**
- [ ] Every public page has a canonical link, and slug variants 301 to it
- [ ] Cross-links to the four sibling sites in the footer
- [ ] Full page text present with JavaScript disabled, verified by `curl`
- [ ] Lighthouse: performance and accessibility both above 95 on the landing page and a documentation page

**Application**

- [ ] Migrations applied to `grantdesk-prod`
- [ ] All secrets set, none in `wrangler.jsonc`, verified by `wrangler secret list` and by reading the committed file
- [ ] First-run setup completes and then disappears
- [ ] Magic-link sign-in delivers to Gmail and Outlook inboxes, not spam
- [ ] Three cron triggers configured and observed firing once each
- [ ] A digest email arrives with correct deadlines, correct grouping by client, a working unsubscribe, and the disclosure
- [ ] The disclosure appears in the app footer, the digest, and the portfolio export
- [ ] With `OPENGRANTS_API_KEY` unset, everything works and nothing mentions the key
- [ ] A webhook with a bad signature returns 401 and is not processed; a redelivery is a no-op
- [ ] Workspace export produces complete CSV and JSON; workspace delete removes rows and R2 objects

**Deploy button and self-host**

- [ ] The Deploy to Cloudflare button completes from an account with no prior state, verified by clicking it
- [ ] The resulting deployment reaches first-run setup with no shell access
- [ ] `docker compose up` works from a clean clone with only `SESSION_SECRET` set
- [ ] Feature parity confirmed between the hosted instance and the Docker instance

**Operations**

- [ ] `/healthz` returns version and migration state and leaks nothing
- [ ] A restore from backup has been rehearsed and timed
- [ ] Logs verified to contain no client names, titles, or amounts, including on an induced error
- [ ] Rate limits verified on sign-in, API tokens, iCal, and webhooks
- [ ] Terms and privacy pages published, describing the operated service, data location, retention, and how to export and delete
- [ ] DNS, TLS, HSTS, SPF, DKIM, DMARC all verified; `workers.dev` disabled

**After launch**

- [ ] Submit the sitemap to Google Search Console and Bing Webmaster Tools
- [ ] Verify in Search Console that no `/app` URL is indexed. Re-check at one week and one month.
- [ ] Ask a model "when is the final SF-425 due on a federal grant" and see whether a template page is reachable and citable
- [ ] Watch the first week of digest emails for delivery failures

---

## 11. Stop and ask the human

1. **Anything that would make a tenant record publicly reachable**, including a "share this calendar" link beyond the existing revocable iCal feed, a public award page, or an embeddable widget. `docs/NON-GOALS.md` section 6 covers this and the answer is no, but come back with the specific request rather than deciding.
2. **Any hosted-only feature.** The hosted deployment is the repository. A feature that exists only here breaks the promise the whole project rests on.
3. **Any analytics or telemetry.** Default is none. If page analytics on the public surface is wanted, it needs an explicit decision, must be cookieless, and must be absent from the app tree entirely.
4. **Any change to the tenancy model, session handling, or the four separation controls.**
5. **Any competitor price or feature claim on a public page.** Re-verify on the vendor's own pricing page and date-stamp it in the text. `docs/program/RESEARCH.md` figures are research, not publishable copy.
6. **Any template page whose citation you cannot verify.** Do not publish a deadline you cannot source. A wrong template on a public page is worse than no page.
7. **Terms of service and privacy policy wording.** Draft it, do not publish it. This is a service holding third-party confidential records and the wording is a legal question, not an engineering one.
8. **Free-tier limits and abuse policy for the hosted instance.** Who can sign up, what the limits are, and what happens when someone runs a hundred workspaces. Decide before launch, not after.
9. **Whether the hosted instance accepts open sign-ups at all at launch**, or starts invite-only. Invite-only is the safer default for a service holding client records, and it is a business decision, not yours.
