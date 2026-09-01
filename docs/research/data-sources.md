# Research: the domain model

Most repositories in this program derive their value from an upstream dataset, and their research file is a list of endpoints and their gotchas. grantdesk is different. It has one optional upstream API and otherwise stores records you create. Its research problem is not "where does the data come from" but **"what are the actual shapes of the things being tracked."**

Get the domain model wrong and no amount of good engineering rescues it. A compliance calendar that does not know the difference between a budget period and a period of performance will generate the wrong final report date for every federal award it touches, confidently.

Everything below was verified on **2026-08-30** unless marked otherwise. Anything marked **VERIFY** must be re-checked before it goes into public copy or into a shipped obligation template. Federal reporting rules change, and the 2024 revision of the Uniform Guidance changed several of them.

**Standing rule for this repository: an obligation template must cite a source and record the date it was checked. A remembered deadline is not a source.**

---

## Part 1: Upstream data (short, because there is not much)

### OpenGrants REST API — optional

- Base: `https://qnoicxojartltrownmal.supabase.co/functions/v1/`
- Auth: `Authorization: Bearer <key>`, keys from the OpenGrants Developer Dashboard
- Endpoints used by grantdesk: `GET /grants-api`, `GET /grants-api/{id}`, `GET /funders-api/{id}`
- Search modes: semantic, keyword, hybrid. Pagination 1 to 100 per page. Status filter: open, closed, upcoming.
- Rate limit headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`
- Docs: https://ops.opengrants.io/api-docs · Hosted MCP: https://mcp.opengrants.io/mcp
- Scale at verification: 139,000+ indexed opportunities, 43,000+ currently open.

Three uses in grantdesk, all optional:

1. **Saved search to pipeline.** A saved search or alert produces matches; the user accepts one and it becomes an `opportunity` with `source = 'opengrants_search'` and the OpenGrants record id retained.
2. **Deadline refresh.** An opportunity carrying an OpenGrants id has its due dates re-checked on a schedule. Federal deadlines move, and a deadline that moved is the single most damaging stale value in the system. **The refreshed value never silently overwrites a user-entered date.** It surfaces as "OpenGrants now reports 2026-11-14, your record says 2026-10-30," with an accept action.
3. **Outcome feedback.** Optional and off by default: award outcomes can be reported back as match-quality signal. This sends outcome and funder identifiers, never client names or amounts, and it requires an explicit opt-in per workspace.

**Webhooks.** OpenGrants publishes 8 event types with HMAC-SHA256 signature verification. This is the mechanism that makes the alert-to-pipeline path timely instead of poll-based, which matters when a NOFO closes in nine days. Implementation requirements are in [`../../prompts/01-build-core.md`](../../prompts/01-build-core.md): verify the signature over the **raw** body before parsing, reject unverified deliveries with 401, store `(provider, external_event_id)` with a unique constraint so redelivery is idempotent, and return 2xx quickly with processing done afterward. **VERIFY** the exact event names, header name, and signature encoding against the current OpenGrants webhook documentation before implementing; do not guess them from this file.

### Grants.gov — reference only

- API guide: https://grants.gov/api/api-guide, base `https://api.grants.gov`
- `search2` (POST, JSON body) and `fetchOpportunity` require **no authentication**.

grantdesk does not index Grants.gov. It uses `fetchOpportunity` for exactly one thing: when a user pastes a funding opportunity number, resolve the title, agency, assistance listing number, and close date so they do not retype them. If the call fails, the user types them. Discovery is `funder-graph` and OpenGrants' job, per [`../NON-GOALS.md`](../NON-GOALS.md).

### Sibling repositories in this program

Cross-links, not dependencies:

- `grantcheck` (`check.opengrants.io`) — an organization's exemption and registration posture by EIN
- `funder-graph` (`funders.opengrants.io`) — the 990-derived funder-to-recipient graph
- `precedent` (`awards.opengrants.io`) — who has actually received federal awards, from USAspending and the Federal Audit Clearinghouse
- `answerbank` (`answers.opengrants.io`) — reusable proposal answer templates

Where a client organization has an EIN on file, grantdesk links out to the same EIN on those sites. It does not embed their data.

### One upstream fact that belongs in a template

The **single audit threshold** rose from $750,000 to $1,000,000 in annual federal expenditures in the 2024 Uniform Guidance revision, effective for fiscal years beginning on or after **2024-10-01**. Organizations below the threshold do not file. This matters to grantdesk because "single audit due" is a legitimate obligation type, and applying it to a client with $600,000 in federal expenditures generates a deadline that does not exist.

---

## Part 2: Federal reporting obligations, the real ones

This is the section that the compliance templates are built from. Citations are to Title 2 of the Code of Federal Regulations, Part 200 (the Uniform Guidance), current at https://www.ecfr.gov/current/title-2/subtitle-A/chapter-II/part-200.

### The vocabulary that has to be right in the schema

Confusing these three is the most common way a grants calendar produces a wrong date.

**Period of performance.** The full span during which the recipient may incur allowable costs, start to end, across all years of a multi-year award.

**Budget period.** A subdivision of the period of performance, usually twelve months, for which a specific funding amount was approved. A five-year federal award typically has five budget periods. Continuation funding is released budget period by budget period, and the progress report that triggers the next release is due **before** the current budget period ends, not after.

**Project period.** Used by some agencies as a synonym for period of performance and by others to mean something slightly different. Because it is ambiguous, grantdesk does not use the term in the schema at all. `award.period_start` / `award.period_end` are the period of performance; `budget_period_end` is the current budget period's end.

**Why this matters concretely.** A recipient with a five-year NIH award who anchors "final report" to the end of the first budget period will file a closeout report four years early. A recipient who anchors the annual progress report to the period of performance end will miss four continuation deadlines. Both are anchoring errors, which is why the recurrence rule has an explicit `anchor` field rather than a single implicit "grant end date."

### SF-425, the Federal Financial Report

The standard form on which recipients report federal cash transactions and expenditures. It replaced the SF-269 and SF-272 family. Published by GSA; instructions and the current form are linked from https://www.grants.gov/forms/post-award-reporting-forms.

Reporting frequency is set by the agency in the Notice of Award, and the permitted options under 2 CFR 200.328 are **quarterly, semiannual, annual, or final**. The regulation caps frequency: the federal agency may not require financial reporting more often than quarterly except in unusual circumstances.

Due dates, per 2 CFR 200.328:

| Frequency | Due |
|---|---|
| Quarterly or semiannual | No later than **30 calendar days** after the end of the reporting period |
| Annual | No later than **90 calendar days** after the end of the reporting period |
| Final | See closeout below |

**Final report timing changed and this is the most commonly wrong date in the domain.** Under 2 CFR 200.344 (closeout), the recipient must submit all financial, performance, and other reports required by the terms of the award no later than **120 calendar days after the end of the period of performance**. This was 90 days before the 2020 revision, and a great deal of guidance still in circulation says 90. A template that says 90 will make people file early, which is harmless, but a template that says 120 where an agency's own terms say 90 will make them file late, which is not. **The award terms govern.** The template's job is to give the right default and show its citation.

**Quarter alignment is not a detail.** Federal financial reporting is typically aligned to the **federal fiscal quarter** (quarters ending December 31, March 31, June 30, September 30; federal fiscal year runs October 1 through September 30), not to the award's anniversary and not to the recipient's own fiscal year. The recurrence engine must be able to express both "quarterly on the federal fiscal quarter" and "quarterly from the award start anniversary," because both occur in the wild and they produce completely different date sets for an award starting in, say, February.

**Cash transaction reporting.** For recipients drawing funds through the HHS Payment Management System, the cash transaction portion of the SF-425 (Lines 10a through 10c) is reported quarterly to PMS on its own schedule, generally within 30 days of the quarter end, separately from the expenditure reporting that goes to the awarding agency. Treating these as one obligation is a common source of a missed filing. grantdesk models them as two obligations of kinds `financial_report` and `payment_report`. **VERIFY** current PMS submission windows against https://pms.psc.gov before shipping a PMS-specific template.

### Performance and progress reports

Governed generally by 2 CFR 200.329. The agency sets the frequency, which may be quarterly, semiannual, or annual, and the regulation similarly caps it at no more often than quarterly except in unusual circumstances. Annual performance reports are generally due 90 calendar days after the reporting period ends; quarterly and semiannual, 30 days. Final performance report timing follows the closeout rule at 2 CFR 200.344, meaning 120 days after the period of performance ends.

Agency-specific forms and timing that a template library needs to know about:

- **SF-PPR, Performance Progress Report.** The government-wide standard form, used with agency-specific attachment pages.
- **RPPR, Research Performance Progress Report.** Used by NIH, NSF, and other research agencies. **NIH:** for non-SNAP awards the RPPR is due the **first of the month preceding the month in which the budget period ends**; for SNAP (Streamlined Non-competing Award Process) awards it is due **45 days before** the next budget period start date. This is the clearest example of a report due *before* the period it closes, and a naive "due N days after period end" engine cannot express it. The recurrence rule therefore allows a **negative offset**. **VERIFY** against the current NIH Grants Policy Statement before shipping the NIH template.
- **NSF.** Annual project reports are due at least **90 days prior to the end of the current budget period**. The final project report and the separate Project Outcomes Report for the general public are due **within 120 days following the end of the award**. **VERIFY** against the current NSF Proposal and Award Policies and Procedures Guide.

### Payment, drawdown, and reimbursement

How money actually reaches the recipient, and each mechanism carries its own deadline shape:

- **Advance payment / drawdown.** Recipient requests funds through a payment system: the HHS Payment Management System, Treasury's ASAP.gov, or an agency portal. 2 CFR 200.305 requires that advances be limited to the minimum amounts needed and be timed to the recipient's actual immediate cash requirements. Practical consequence for the calendar: **drawdown windows**. Many awards permit drawdowns only within a set window after an expense is incurred, or close drawdown entirely a fixed number of days after the period of performance ends. A grant that is fully spent but not fully drawn down before the window closes leaves money on the table, and this is a real and expensive failure that no spreadsheet catches.
- **Reimbursement.** Recipient spends first and requests reimbursement, typically on **SF-270, Request for Advance or Reimbursement**, or through an agency portal. Frequency is set by the award; monthly and quarterly are both common. This is a recurring obligation like a report, and it is the one where being late costs the client cash flow immediately.
- **SF-271, Outlay Report and Request for Reimbursement for Construction Programs.** Narrower, but real for capital awards.
- **Foundation tranche payments.** The non-federal analogue: a foundation releases the second half of a grant on acceptance of the interim report. That makes the interim report a payment trigger, not just paperwork, and the calendar should say so.

### Prior approval and modification deadlines, where the quiet damage is

2 CFR 200.308 governs revision of budget and program plans. The recipient must obtain **prior written approval** from the awarding agency for, among other things:

- A change in the scope or objective of the project
- A change in a key person specified in the application or award
- The disengagement of the project director or principal investigator for more than three months, or a 25 percent reduction in time devoted to the project
- The need for additional federal funds
- The transfer of funds budgeted for participant support costs to other categories of expense
- The subawarding, transferring, or contracting out of any work under the award, unless described in the approved application

Cumulative budget transfers among direct cost categories require prior approval when they exceed **10 percent of the total approved budget**, and this threshold applies only where the federal share of the award exceeds the **Simplified Acquisition Threshold** (currently **$250,000**). **VERIFY** the Simplified Acquisition Threshold at the time of shipping; it is adjusted for inflation periodically under 41 U.S.C. 1908.

**One-time no-cost extension.** Under 2 CFR 200.308, a recipient may make a one-time extension of the period of performance of up to **12 months** without prior approval, provided the agency has not restricted this and the extension is not merely to use unobligated funds. The recipient must **notify the agency in writing, with supporting reasons and the revised end date, at least 10 calendar days before the end of the period of performance.**

That ten-day requirement is the highest-value single deadline in this entire domain, and it is the one most often missed. It is a small, obscure window at the exact moment everyone's attention is on finishing the project. Miss it and the extension requires the agency's discretion instead of being available as a right. grantdesk's federal templates therefore generate a `no_cost_extension_deadline` obligation at **period end minus 30 days** (not minus 10) with the citation attached, because a deadline that surfaces on the day it expires is not useful. The obligation is dismissible in one click when no extension is needed.

Similarly, agencies generally expect prior-approval requests to arrive well before the period ends, and many terms require them **at least 30 days** before the change or before the period of performance ends. Template default: **period end minus 60 days** for a `prior_approval_deadline` reminder, so there is room for the conversation.

### Other recurring federal obligations worth modeling

- **SF-428, Tangible Personal Property Report**, and **SF-429, Real Property Status Report.** Annual and final variants, applicable when federally funded equipment or real property is involved.
- **Subaward reporting under FFATA.** Prime recipients report first-tier subawards over the reporting threshold in the FFATA Subaward Reporting System (FSRS), generally **by the end of the month following the month in which the subaward was made**. **VERIFY** the current reporting threshold and whether FSRS reporting has migrated to SAM.gov, which was in progress at last check.
- **SAM.gov registration renewal.** Registration must be renewed **annually** and an expired registration blocks both new applications and, in some agencies, payment. This is the single most common avoidable disqualification in federal grant work, it is fully preventable, and it belongs on the calendar as a recurring annual obligation anchored to the registration expiration date, with the reminder at expiry minus 60 days because renewal is not instant.
- **Single audit submission.** Required of organizations expending $1,000,000 or more in federal awards in a fiscal year (2024 revision; previously $750,000), effective for fiscal years beginning on or after 2024-10-01. Due to the Federal Audit Clearinghouse the earlier of **30 calendar days after receipt of the auditor's report** or **nine months after the end of the audit period**. Anchored to the client organization's fiscal year end, which is why `organization.fiscal_year_end_month` is on the schema.
- **Closeout package.** Beyond the final financial and performance reports, closeout under 2 CFR 200.344 can include a final invention statement, property reports, and liquidation of obligations. All within the same 120-day window.

### The pattern the schema has to support, stated as requirements

Reading the above back, the recurrence engine must be able to express all of these, or it fails on real awards:

1. Due a fixed number of days **after** a period ends (quarterly SF-425 at plus 30)
2. Due a fixed number of days **before** a period ends (NSF annual report at minus 90, NIH SNAP RPPR at minus 45)
3. Anchored to a **calendar or federal fiscal quarter**, independent of the award start date
4. Anchored to the **award anniversary**
5. Anchored to the **client organization's fiscal year end** (single audit)
6. Anchored to an **external date** unrelated to the award (SAM registration expiry)
7. A **one-time** obligation at an offset from the period of performance end (final reports, the no-cost extension window)
8. A period end that **moves** when an extension is granted, causing dependent dates to move with it, without destroying obligations already filed

Requirement 8 is the one that makes this an engine rather than a loop. A no-cost extension on a three-year award moves the period end, which should move the final report, the closeout obligations, and the drawdown window, while leaving the eight quarterly reports already filed exactly where they are. That behavior is specified in the build prompt.

---

## Part 3: Foundation reporting patterns

Foundation requirements are not published in one place and vary by funder, so this section describes observed patterns rather than citable rules. Templates built from it are labeled **pattern, not requirement**, and every generated obligation says so.

### The common shapes

**The single final report.** Most common for grants under roughly $25,000 and for most family foundations. One narrative and financial report, due somewhere between 30 and 90 days after the grant period ends. Some smaller family foundations require nothing at all, and a template that generates three obligations for a $5,000 grant is noise that trains people to ignore the calendar.

**Interim plus final.** The standard shape for a twelve-month grant of meaningful size. Interim narrative and financial report at the six-month mark, final report 30 to 60 days after the grant period ends. Frequently the interim report release the second payment tranche.

**Annual reporting on a multi-year commitment.** A three-year grant paid in three annual installments, with each year's report due before the next payment is released. Structurally identical to federal continuation funding: the report is a precondition for money, so it is due before the period ends, not after.

**Report cycle keyed to the funder's fiscal year rather than the grant.** Less common but real, particularly with community foundations that batch grantee reporting. Requires the calendar-anchored rule type rather than the anniversary-anchored one.

### The dates that are not reports and get missed anyway

- **Renewal application or LOI deadline.** Frequently falls **before** the current grant's final report is due. A funder with a February LOI deadline and a grant period ending March 31 means the renewal ask is written before the current grant's results are written up. Missing it costs a year. This is a pipeline item and a compliance obligation at the same time, and grantdesk links them: an obligation of kind `renewal_application` can carry the id of the opportunity it created.
- **Board docket cutoff.** Foundation boards typically meet quarterly, and the materials deadline commonly precedes the meeting by six to ten weeks. The published "application deadline" and the internal docket cutoff are not always the same date, and the program officer knows the real one. This is a pipeline deadline (`opportunity_deadline` of kind `docket_cutoff`), not an award obligation.
- **Budget modification approval.** Many foundations require written approval to move more than 10 to 20 percent between budget lines, or to change the project scope. The window is usually described as "in advance," which in practice means before the money is spent, and there is no hard date. Template default: a reminder at the grant midpoint that a modification request must precede the spend, which is a nudge rather than a deadline.
- **No-cost extension request.** Typically 30 to 60 days before the grant end date, by email to the program officer. Far more informal than the federal version and far more commonly granted, and still routinely asked for after the grant has already ended, which is a much worse conversation.
- **Site visit.** Scheduled, not deadline-driven, but it belongs on the calendar because preparing for one is a week of work.
- **Acknowledgment and recognition obligations.** Logo placement, press release timing, naming in an annual report. Small, dated, and awkward to miss because they are visible to the funder.
- **Expenditure responsibility reports.** When a private foundation makes a grant to an entity that is not a public charity, it must exercise expenditure responsibility, which obliges the grantee to report on the use of funds until the grant is fully expended. Relevant to fiscally sponsored projects and to grants to for-profit or foreign entities. The reporting cadence is set by the foundation and is usually annual until full expenditure. **VERIFY** the governing citation (Internal Revenue Code section 4945(h) and the related Treasury regulations) before putting it in template help text.

### Portals, and why grantdesk does not touch them

Foundation reports are usually filed through a grants management portal: **Foundant GLM**, **Fluxx**, **Submittable**, **SurveyMonkey Apply**, **Blackbaud Grantmaking**, or the funder's own form. A consultant with seven clients may hold logins to twenty of these.

grantdesk stores the portal URL on the award and links to it from the obligation, so the deadline and the place it gets filed are one click apart. It does not log in, does not scrape, and does not submit. See [`../NON-GOALS.md`](../NON-GOALS.md) section 3.

---

## Part 4: The stage sequence

The stage list in the schema is a research output, not a design preference. It is the sequence that grant work actually follows, and the two branches in it are the reason single-funnel pipeline tools model this domain badly.

### Private and community foundation track

```
identified          prospect research turned it up, nobody has looked hard yet
qualifying          fit, eligibility, geography, average grant size, whether they
                    accept unsolicited requests at all
inquiry             the call or email to the program officer that precedes an LOI
                    at most foundations that matter
loi_drafting        writing the letter of inquiry or concept paper, typically
                    one to three pages
loi_submitted       sent, waiting
invited             the branch: invited to submit a full proposal
                    (declining here is `declined`, and it is a different
                    outcome than declining after a full proposal)
proposal_drafting
proposal_submitted
site_visit          a site visit or interview, common above roughly $50,000
under_review        with the board, awaiting a docket decision
awarded | declined | withdrawn
```

**The branch is the whole point.** LOI-to-invitation and invitation-to-award are two different conversion rates measuring two different skills. A writer with a 70 percent LOI-to-invitation rate and a 30 percent invitation-to-award rate is excellent at fit and positioning and is losing proposals on substance or budget. The reverse profile is the opposite problem. A tool that reports one blended "win rate" tells that person nothing, and it is the number every single-funnel tool reports.

**Not every opportunity enters at the top.** Invited-only funders enter at `invited`. A renewal of an existing grant may enter at `proposal_drafting`. The stage model must permit entry at any stage and must not require passing through earlier ones.

**Stage duration is worth capturing** and is nearly free: recording `stage_entered_at` on each transition gives time-in-stage, which is what tells a consultant that their LOIs sit for eleven weeks at one funder and three at another. That is genuinely useful for planning and costs one column.

### Federal track

```
forecast            a forecasted opportunity, no NOFO published yet
identified          NOFO published
qualifying          eligibility check, cost share, SAM registration status,
                    whether the client can realistically staff it
loi_submitted       optional and agency-specific. NIH letters of intent are
                    generally due 30 days before the application deadline and
                    are not binding, but they inform review panel assembly
proposal_drafting
proposal_submitted   submitted through Grants.gov, NIH ASSIST, NSF Research.gov,
                    or an agency portal
under_review         peer or panel review; months, sometimes many
awarded | declined
```

Federal specifics that change how the pipeline behaves:

- **Deadlines are hard.** A foundation will usually take a proposal a day late. Grants.gov will not, and the difference should be visible in the interface. `opportunity.deadline_is_hard` drives more aggressive lead-time alerting.
- **The submission validation window.** A Grants.gov submission is not complete when it is uploaded. It is validated, and errors must be corrected by the Authorized Organization Representative. Submitting at 4:55pm on the deadline day with a validation error is a failed submission. The federal template therefore generates an internal `submission_target` deadline **five business days before** the published deadline, which is what experienced federal writers do by hand anyway.
- **SAM registration is a gate, not a task.** An expired SAM.gov registration blocks submission entirely. grantdesk surfaces the client organization's SAM expiration date on every federal opportunity in the pipeline. This one field prevents the most common avoidable disqualification in the domain.
- **JIT, Just-in-Time.** NIH requests additional information after review and before award for applications likely to be funded. It arrives with a short fuse and it is a real deadline that lands between `under_review` and `awarded`.
- **The gap between award notice and Notice of Award.** A verbal or emailed "you're funded" is not the document that carries the reporting terms. Obligations should be generated from the actual Notice of Award, and grantdesk's award form asks for the NoA date specifically to make that distinction visible.

### Outcomes worth distinguishing

A single `lost` flag throws away most of the value of the ledger. The outcomes modeled:

`awarded_full`, `awarded_partial` (funded below ask, which is very common and belongs in the amount fields rather than hidden in a status), `declined`, `not_invited` (declined at the LOI stage, a different signal), `withdrawn` (the client pulled out, usually a capacity decision), `no_response` (real, especially with family foundations, and pretending it is a decline distorts the denominator), `ineligible_discovered_late` (a qualification failure, not a writing failure, and separating it is how a practice fixes its qualification process).

Decline reason, where known, is a free-text note plus an optional coded reason: `not_a_fit`, `funding_exhausted`, `too_competitive`, `budget_concerns`, `capacity_concerns`, `incomplete_application`, `missed_deadline`, `unknown`. `unknown` will be the most common value and that is fine. A field that is honestly empty is better than a field that is guessed.

---

## Part 5: What the ledger has to record for the portfolio export to be honest

The portfolio export is a headline feature, and it is only defensible if the underlying record supports the segmentation. Minimum per submission:

- Date submitted, and date of outcome
- Client organization, and its type or sector
- Funder, and **funder type**: `federal`, `state`, `local_government`, `private_foundation`, `family_foundation`, `community_foundation`, `corporate_foundation`, `corporate_giving`, `united_way_or_federated`, `individual_or_other`
- Submission type: `loi`, `full_proposal`, `renewal`, `continuation`, `report_only`
- Amount requested, amount awarded
- Outcome, from the list above
- Lead writer, where more than one person works the pipeline
- Whether it was a competitive process or an invited or renewal ask

**Funder type is the axis the entire export depends on** and it must be a controlled vocabulary rather than a free-text field. It is also the field users are most likely to fill in carelessly, so the interface should default it from the funder record and from OpenGrants data where available rather than asking every time.

**The segmentation rule, restated because it is the difference between a useful export and a vanity metric.** Hit rate is reported by funder type and by award size band, never as one number. Denominators are always visible. A segment with fewer than five submissions is labeled as insufficient rather than shown as a percentage. LOI-to-invitation and invitation-to-award are reported separately and never blended. Multi-year trend is shown per segment, because a consultant deliberately moving a client from local foundations to federal work will show a falling blended rate while doing better work, and an export that makes that look like decline is actively harmful to the person it was built to help.

---

## Verification log

| Fact | Source | Checked |
|---|---|---|
| Uniform Guidance financial reporting frequency and 30/90 day due dates | 2 CFR 200.328 | 2026-08-30 |
| Closeout reports due 120 days after period of performance ends | 2 CFR 200.344 | 2026-08-30 |
| Performance reporting frequency and due dates | 2 CFR 200.329 | 2026-08-30 |
| Prior approval requirements; 10 percent cumulative transfer threshold | 2 CFR 200.308 | 2026-08-30 |
| One-time 12-month no-cost extension; 10-day advance written notice | 2 CFR 200.308 | 2026-08-30 |
| Advance payment timing limited to immediate cash requirements | 2 CFR 200.305 | 2026-08-30 |
| Single audit threshold $1,000,000, FY beginning on or after 2024-10-01 | Egeria `docs/program/RESEARCH.md`, 2024 Uniform Guidance revision | 2026-08-30 |
| Simplified Acquisition Threshold $250,000 | 2 CFR 200.1 definitions | 2026-08-30, **VERIFY** at ship |
| NIH RPPR due dates, non-SNAP and SNAP | NIH Grants Policy Statement | **VERIFY** at ship |
| NSF annual report 90 days prior, final and outcomes report within 120 days | NSF PAPPG | **VERIFY** at ship |
| FFATA subaward reporting threshold and FSRS status | FSRS / SAM.gov | **VERIFY** at ship |
| HHS Payment Management System cash transaction windows | pms.psc.gov | **VERIFY** at ship |
| Expenditure responsibility reporting citation | IRC 4945(h) and Treasury regulations | **VERIFY** at ship |
| OpenGrants API base, endpoints, webhook count and signature scheme | Egeria `docs/program/RESEARCH.md`; https://ops.opengrants.io/api-docs | 2026-08-30 |
| Grants.gov `search2` and `fetchOpportunity` require no authentication | https://grants.gov/api/api-guide | 2026-08-30 |

Every shipped obligation template carries its own `citation` and `citation_checked_on` fields, rendered next to the generated obligation in the interface. When a template's citation is more than eighteen months old, the interface says so rather than pretending the date is current.
