# Prior art

What already exists, what each thing is good at, and specifically where each one fails a consultant with seven clients and a post-award problem.

This is not a takedown list. Instrumentl is a good product and Foundant runs a large part of the sector's grantmaking. The point of this file is narrower: to be honest about what is already solved, so grantdesk does not rebuild it, and precise about what is not, so grantdesk's scope stays defensible.

Everything here was checked on **2026-08-30**. Pricing and feature claims about commercial products go stale fast and are date-stamped for that reason. **Re-verify on the vendor's own site before repeating any of it in public copy.**

---

## Instrumentl

https://www.instrumentl.com

**What it is.** The strongest prospect research and pipeline tool in the nonprofit space. Matching against a large opportunity database, funder profiles built partly on 990 data, deadline tracking, saved searches, and reporting on your pipeline. It is genuinely good at the thing it is good at, and if the problem is "which funders should we approach," it is the answer.

**Pricing at verification.** Four tiers at $179, $299, $499, and $899 per month (source: Capterra, current at 2026-08-30). Higher tiers are what unlock the multi-client and multi-user capability a consultant needs.

**Where it fails a consultant:**

- **Post-award is essentially absent.** Instrumentl tracks the chase. Once an award is made, the reporting obligations, drawdown windows, and modification deadlines that follow are not a first-class object. The most damaging part of the job is out of scope, and that is the entire gap grantdesk exists in.
- **Multi-client is priced as a tier, not designed as a model.** A consultant with seven clients is doing seven organizations' work, and the pricing treats that as an upgrade path rather than the normal shape of the customer. At $499 to $899 a month it is a real line item for a solo consultant, and the value delivered is mostly on the prospecting side they may already have covered.
- **Single-stage funnel.** The pipeline does not natively distinguish LOI-to-invitation from invitation-to-award, so the two conversion rates that describe a grant writer's actual performance get blended into one number that describes neither. (**VERIFY** current stage customization before repeating this claim publicly; stage configurability may have changed.)
- **Your history is inside their product.** A consultant who leaves takes an export, not a record. The professional portfolio that should accumulate over a career accumulates inside a subscription instead.

**What grantdesk deliberately does not copy.** Prospect research and matching. That is Instrumentl's core competence and, in this program, OpenGrants' and `funder-graph`'s job. grantdesk is the downstream system that receives a match and manages what happens after.

---

## Foundant Technologies

https://www.foundant.com

**What it is.** Two related products. **GLM (Grant Lifecycle Manager)** is grantmaking software sold to foundations: application intake, review, docket assembly, payments, and grantee reporting. **GrantHub** (and its successor GrantHub Pro) is the grantseeker-side product: pipeline, deadlines, document library, task assignment.

**Pricing at verification.** Not published; quote-based. **VERIFY** before stating any figure.

**Where it fails a consultant:**

- **GLM is the wrong side of the transaction.** It is excellent grantmaking software and irrelevant to a grantseeker except as the portal they file reports into. Consultants often encounter Foundant only as "the system this funder uses," and holding twelve GLM logins across twelve funders is one of the daily frictions of the job.
- **GrantHub's model is one organization.** It was designed for a development team at a single nonprofit. A consultant using it either buys an instance per client, which is absurd, or mixes clients in one instance, which is a confidentiality problem waiting for a bad afternoon.
- **Post-award reporting is tracked as tasks, not modeled as obligations.** A generic task with a due date does not know that it is a quarterly SF-425 anchored to the federal fiscal quarter, does not regenerate when the period of performance moves, and cannot cite the rule it came from. It also has to be entered by hand, once per occurrence. For a three-year federal award that is eighteen manual entries, and everyone does the first six.
- **Closed data model.** Limited API access relative to what a consultant integrating their own workflow would want. **VERIFY** current API availability.

---

## Submittable

https://www.submittable.com

**What it is.** A submission and review platform used widely by foundations, arts organizations, and government programs for application intake and grantee reporting. Strong form building, review workflow, and reporting collection.

**Pricing at verification.** Quote-based. **VERIFY**.

**Where it fails a consultant:** it is not a tool for the applicant at all. It is where you file, not where you plan. From the consultant's side Submittable is another portal with another login and another set of deadlines that live inside somebody else's system with no way to pull them into one calendar. That is precisely the situation that makes the compliance calendar valuable: the deadlines are scattered across five vendors' portals, so the only place they can be consolidated is a system the consultant owns.

grantdesk's response is deliberately modest. It stores the portal URL on the award and links to it from the obligation. It does not integrate, scrape, or authenticate. See [`../NON-GOALS.md`](../NON-GOALS.md) section 3.

---

## Fluxx, Blackbaud Grantmaking, SurveyMonkey Apply

The rest of the grants management portal landscape. Same relationship as Submittable: they are systems the consultant files into, on behalf of clients, with credentials held per funder. Each is a place a deadline lives that grantdesk needs to link to and will never integrate with.

Worth naming explicitly because "which portal does this funder use" is real operational knowledge a consultant carries in their head, and `award.portal_url` plus `funder.portal_kind` is a cheap way to write it down.

---

## The spreadsheet

Google Sheets or Excel, one tab per client, sorted by deadline. This is the actual incumbent. It is what nearly every independent consultant and small firm uses, and any honest assessment starts by admitting it works better than most software.

**What it does well, genuinely:**

- Zero cost, zero onboarding, zero lock-in, works offline, works forever
- Infinitely flexible, and shaped to how this particular consultant thinks
- Shareable with a client in one click
- Every consultant already knows how to use it, which is not true of any product on this list

**Where it fails, specifically:**

- **It has no memory for recurrence.** A three-year federal award's eighteen reporting deadlines have to be typed in one at a time, and they are not, because at the moment of award nobody sits down to enumerate the next thirty-six months. They get typed in for the first year at best. This is the single highest-value thing grantdesk does that a spreadsheet cannot.
- **Nothing surfaces.** A spreadsheet is a pull interface. It tells you what is due when you open it, which is Monday morning, and the interim report that was due Thursday was due Thursday. Lead-time alerting is the difference between a calendar and a list.
- **Post-award falls off.** In practice the tracking sheet is a pipeline sheet. Once an award is made the row is marked "Won" and the grant moves to a folder. The obligations that follow live in the program officer's email and in somebody's memory.
- **Dependent dates do not move.** A no-cost extension changes the period of performance end, which should move the final report, the closeout obligations, and the drawdown window. In a spreadsheet, one of those gets updated and the other three do not.
- **No history worth anything.** Rows get overwritten. Last year's tab gets archived or deleted. The three-year segmented record that would support a rate negotiation or a GPC application does not exist, because the spreadsheet was never a ledger.
- **Confidentiality is by convention.** Seven client tabs in one workbook, shared with a subcontractor to give them access to one client, is a mistake that is one careless share away at all times.

**The design consequence.** grantdesk must be better than the spreadsheet at recurrence, surfacing, and history, and must not be worse at the things the spreadsheet is good at. Which means: import from CSV on day one, export to CSV always, no lock-in, and an interface that a consultant can be productive in without training. If setting up a client in grantdesk takes longer than adding a tab, the spreadsheet wins and should.

---

## Adjacent open source

**No direct equivalent exists.** Searching the open-source landscape for grant-seeker pipeline and post-award compliance tooling turns up very little, and what exists is mostly abandoned university projects or the grantmaking side. This is unusual for a category with this much commercial revenue in it, and it is a reasonable part of the case for building it.

Adjacent things that do exist and are worth knowing about:

- **CiviCRM** (AGPL-3.0), https://civicrm.org — has a CiviGrant component, but it is grantmaking-oriented and it arrives attached to a full CRM, which is the exact shape [`../NON-GOALS.md`](../NON-GOALS.md) exists to avoid. Reasonable prior art for the permission model.
- **Ledgerable open-source project management tools** (Vikunja, Focalboard, Plane) — used by some consultants as a deadline tracker. They fail for the same reason a spreadsheet does: a generic task has no idea it is an SF-425, cannot be generated from a rule tied to a period of performance, and cannot cite 2 CFR 200.328.
- **`ical.js` and the iCalendar specification (RFC 5545)** — grantdesk publishes a read-only iCal feed. RRULE in RFC 5545 is a well-designed recurrence grammar and was studied when designing the obligation rule, but it was **not adopted**, and the reason is worth writing down: RRULE expresses recurrence of an *event*, while a reporting obligation has a *period* (the quarter being reported on) and a separate *due date* derived from that period, plus anchoring semantics RRULE has no concept of. Trying to express "quarterly on the federal fiscal quarter, due 30 days after each quarter ends" in RRULE produces something correct and unreadable, and it cannot express the NIH rule at all. grantdesk uses its own narrow rule type and emits RFC 5545 on the way out. Design notes are in [`../../prompts/01-build-core.md`](../../prompts/01-build-core.md).

---

## Upstream contribution posture

Per the program's conventions, where grantdesk needs a fix in something it depends on, the pull request goes upstream first and gets noted here. Nothing to record yet.

Two areas where contribution is likely and should be watched for:

- **Drizzle ORM's D1 driver.** grantdesk exercises the one-schema-two-drivers path hard, particularly around transactions, which behave differently on D1 than on better-sqlite3. Any workaround written here should be reported upstream rather than kept as local folklore.
- **Hono's Node adapter.** Parity between the Workers and Node targets is a hard requirement, and gaps found in that parity belong upstream.

The broader program builds directly on the **[Nonprofit Open Data Collective](https://github.com/Nonprofit-Open-Data-Collective)** and **[GivingTuesday](https://990data.givingtuesday.org/tool-repository/)** Form 990 tooling, credited in [`../../NOTICE`](../../NOTICE). grantdesk does not consume that work directly, but it links to the sibling sites that do, and the same posture applies: contribute upstream, credit loudly, never re-implement something a community project already does well in order to own it.

---

## Summary table

| | Pipeline | LOI branch | Post-award compliance | Recurring obligations from a rule | Multi-client by design | Career portfolio | Price |
|---|---|---|---|---|---|---|---|
| Instrumentl | Strong | No | Minimal | No | Priced as a tier | No | $179 to $899/mo |
| Foundant GrantHub | Good | No | Tasks only | No | No | No | Quote |
| Foundant GLM | Grantmaker side | n/a | Grantmaker side | n/a | n/a | No | Quote |
| Submittable | No | No | Filing portal only | No | n/a | No | Quote |
| Spreadsheet | Adequate | If you build it | Falls off | No | By convention | No | Free |
| **grantdesk** | **Yes** | **Yes** | **The point** | **Yes** | **Schema level** | **Yes** | **Free, Apache-2.0** |

All figures 2026-08-30. Re-verify before publishing.
