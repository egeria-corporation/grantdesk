# What this replaces

> **Note on scope.** This file describes the capability gap `grantdesk` fills. It deliberately names
> no vendor and quotes no price. Comparative analysis of commercial products is maintained outside
> this repository for now. Nothing in the application, its help text, its output, the public
> marketing surface, or any hosted page may name or price a commercial product — see
> `docs/program/CONVENTIONS.md`.

## The asymmetry this is built on

Commercial tools in this category compete on the **pre-award** half: prospect research, funder
matching, opportunity discovery, pipeline. That half is well served, genuinely hard to build well,
and `grantdesk` does not attempt it.

The **post-award** half is where the expensive mistakes live, and almost nothing does it well:

- An interim report missed by two weeks.
- A budget modification request filed after the window closed.
- A final report that costs a client their renewal.

Each of those is a calendar problem with a regulation behind it. None of them is hard to prevent if
something is watching the dates. **That is the whole product**, and it is why the recurrence engine
is the milestone not to compress.

## The honest scope claim

`grantdesk` does not replace prospect research, and the README should never imply it does. What it
replaces is the spreadsheet — which is survivable for pipeline and is not survivable for compliance,
because a spreadsheet does not know that a quarterly SF-425 due 2027-01-30 lands on a Saturday and
shifts to the next business day.

Three things, and it refuses to do a fourth. The list of fourths is `docs/NON-GOALS.md`, and it is
long on purpose.

## Where consultants get failed specifically

Most nonprofit software assumes **one organization per install**. A consultant has seven clients.
That assumption is cheap to make on day one and effectively impossible to remove later, which is why
multi-tenancy here starts in the first migration rather than arriving in v2.

Every domain table carries both `workspace_id` and `org_id`. A cross-tenant leak is a release
blocker.

## The part nobody else has a reason to build

The win/loss ledger exports a **grant professional portfolio**: total submitted, total awarded, hit
rate segmented by funder type and award size, multi-year trend, with denominators visible and small
segments labeled as such.

Development professionals have no standard way to prove their value — "I raised money" is not a
number — and certification requires documented experience that most people reconstruct painfully
from memory. A tool that accumulates that record automatically is genuinely useful for rate-setting
and negotiation.

A commercial vendor has no incentive to build a portable career asset, because portability is the
opposite of retention. That is precisely why it belongs in an Apache-2.0 repository.

## What `grantdesk` does not claim

- It is not a CRM, and the moment it grows contact management it becomes a product with a support
  burden instead of a repository.
- It does not file anything on your behalf, log into any portal, or scrape one.
- It reports compliance posture. It does not determine compliance, and it is not legal, tax, or
  accounting advice.
