# Competitive: the parity target and its price

## The target

**Instrumentl**, at its multi-user and multi-client tiers.

Instrumentl is the product a grant consultant actually evaluates when they decide their spreadsheet has stopped working. It is the honest comparison, and grantdesk's pipeline half has to be good enough that a consultant does not need Instrumentl's tracking features to run their practice.

Note the shape of the claim carefully. **grantdesk does not replace Instrumentl's prospect research.** Instrumentl's matching engine, funder profiles, and opportunity database are its core value, they are expensive to build, and in this program that job belongs to [OpenGrants](https://opengrants.io) and to the sibling repositories `funder-graph` and `precedent`. What grantdesk replaces is the **tracking and management layer** that sits on top of discovery, plus a post-award layer that Instrumentl does not have at all.

---

## The price

| Product | Price at 2026-08-30 | Source | Notes |
|---|---|---|---|
| **Instrumentl** | **$179 / $299 / $499 / $899 per month** | Capterra listing, current at verification | Four tiers. Multi-user and the higher opportunity-tracking limits sit in the upper tiers, which is where a consultant with several clients lands. |
| Candid Foundation Directory | $1,599/year, or $219.99/month, professional level | May 2024 comparison, https://fundingforgood.org/comparing-grant-research-databases/ | A lower "essential" tier exists. **VERIFY** |
| Cause IQ | $199/month or $999/year, limited free tier | Same May 2024 source | **VERIFY** |
| Grant Gopher | $9/month, limited free option | Same May 2024 source | **VERIFY** |
| Foundant GrantHub / GrantHub Pro | Not published, quote-based | Vendor site | **VERIFY** |
| Submittable | Not published, quote-based | Vendor site | **VERIFY** |
| Plinth | Not public at verification | https://www.useplinth.com/ | Newest entrant, positions a 990-derived funding graph |
| OpenGrants | Free unlimited search, no account; paid tier **$9/month** | Egeria `docs/program/RESEARCH.md` | 139,000+ indexed opportunities, 43,000+ open |
| **grantdesk** | **$0.** Apache-2.0. Runs on Cloudflare's free tier or $5/month Workers Paid, or on your own Docker host. | | No per-seat cost, because there is nobody to charge. |

**Rule for any public copy, from the program conventions:** re-verify a competitor price on that vendor's own pricing page before publishing it, and date-stamp it in the text. Stale competitor pricing in a README is an accuracy problem and an easy thing for a competitor to make us look bad over. Capterra and third-party comparison sites are adequate for an internal research file like this one and are not adequate for a claim on a landing page.

---

## The realistic annual cost being displaced

A three-person consulting firm serving eight clients, priced at Instrumentl's tiers, is at $499 or $899 a month depending on client count and seats. Call it **$6,000 to $10,800 a year**.

Add the reality that Instrumentl does not cover post-award, so the compliance calendar is a spreadsheet, and the actual cost includes the incident that spreadsheet eventually produces. One missed final report on a $250,000 federal award, or one foundation renewal lost because the interim report was two weeks late, exceeds a year of subscription by a wide margin and is not recoverable by paying more.

grantdesk's substitute cost is a Cloudflare account and a domain. Realistically **$0 to $5 a month**, and the self-hosted path is the price of a server the firm probably already has.

---

## Where the comparison is honest, and where it is not

Overstating this would be both wrong and easy to disprove, so the boundaries:

**grantdesk is genuinely better at:**

- **Post-award compliance.** Not incrementally. This is a category the paid pipeline tools do not occupy, and a rule-generated obligation calendar with correct anchoring semantics does not exist in any of them.
- **Multi-client architecture.** Tenancy at the schema level with a consultant-wide roll-up view, rather than a pricing tier that permits more organizations.
- **The LOI branch.** Two conversion rates reported separately, because they measure two different things.
- **The career record.** Nobody else does this at all. The professional portfolio export is not a feature any competitor has, partly because a subscription product has no incentive to make your history portable.
- **Data ownership and exit cost.** Your records are in your database. CSV in, CSV out, JSON API, MCP server. Apache-2.0.
- **Price.**

**Instrumentl is genuinely better at:**

- **Prospect research and matching.** Not close, and not the goal.
- **Funder intelligence.** Profiles, giving history, typical grant size, board composition. Real work that grantdesk does not attempt.
- **Being finished.** It is a supported commercial product with onboarding, a help desk, and a roadmap. grantdesk is an open-source repository with a `NON-GOALS` file that says no to most of what a support relationship would produce.
- **Not being your problem.** Somebody else runs it. For a consultant with no technical appetite, "one click deploy" is still one click more than "log in," and that is a real difference.

**Candid Foundation Directory** is a research database with a different job, and it is not a competitor to grantdesk in any direction.

**Foundant GLM, Fluxx, Submittable** are grantmaking-side systems. They are the portals grantdesk links to, not products it competes with.

---

## The positioning sentence

> Instrumentl finds the grant. grantdesk keeps you from losing the client after you win it.

Defensible, checkable, and it does not claim to replace the prospect research. It also names the actual pain: the damage in this business happens after the award, and the tools are all pointed at the part before it.

---

## Why a free tool can win this specific category

Three reasons, and they are structural rather than optimistic.

**The paid tools do not cover the expensive half.** grantdesk is not a cheaper version of an existing product, it is the missing half of one. That is a much better position than undercutting.

**The buyer is fragmented and small.** Independent consultants and two-to-six person firms are the users with the sharpest version of this problem and the least ability to absorb $6,000 a year. They are also badly served by enterprise sales motions, which is why the incumbent in this segment is a spreadsheet rather than a product.

**The compliance model is knowledge, not infrastructure.** What makes the obligation templates valuable is that somebody read 2 CFR 200.344 and knows the final report is due 120 days after the period of performance rather than 90. That knowledge is exactly the kind of thing an open, citable, community-maintained repository holds better than a closed product does, because a template with a public citation can be checked and corrected by the practitioners who hit the edge case.

The risk is the mirror image and should be stated: an open-source tool that people rely on for deadlines carries an accuracy obligation, and a wrong template is worse than no template. That is why every template must cite a source and record a verification date, and why the disclosure appears on every surface. See [`../../CONTRIBUTING.md`](../../CONTRIBUTING.md) rule 6.

---

## What would change this assessment

Written down now so it can be checked later rather than argued about:

- **Instrumentl ships real post-award compliance with rule-generated recurring obligations.** This is the one that matters. It is a plausible product move and it would compress grantdesk's differentiation to multi-client tenancy, the portfolio export, and price. Those are still real, and they are a much weaker position.
- **A well-funded entrant builds this specifically for consultants.** Plinth is the closest thing to a new entrant worth watching.
- **The federal reporting landscape changes materially.** A significant Uniform Guidance revision would invalidate a batch of templates at once. The citation-and-date discipline is what makes that a manageable afternoon instead of a silent correctness failure.

Review this file at each minor release, and re-verify every price before any of it appears in public copy.
