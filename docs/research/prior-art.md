# Prior art

What already exists, what it is good at, and specifically where it fails a consultant with seven
clients and a post-award problem. The point is to be honest about what is already solved, so
grantdesk does not rebuild it, and precise about what is not, so grantdesk's scope stays defensible.

> **Note on scope.** Sections comparing named commercial products have been removed from this
> repository for now; that analysis is maintained internally. What remains is what the program's
> conventions actually require prior art to cover: the open source and community work this
> repository builds on, the incumbent practice it has to beat, and the upstream contribution
> posture. Nothing here names or prices a commercial product.

---

## Commercial grant management and prospect research software

**Removed from this repository for now.** The short version that matters for scope: the category
competes on the pre-award half — prospect research, funder matching, opportunity discovery,
pipeline — and that half is well served. grantdesk does not attempt it. The post-award half, where
recurring reporting obligations live, is the gap. See `competitive.md`.

One structural observation is worth keeping, because it is a design constraint rather than a
comparison: **most software in this sector assumes one organization per install.** A consultant has
several clients. That assumption is cheap on day one and effectively impossible to remove later,
which is why multi-tenancy here starts in the first migration.

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
