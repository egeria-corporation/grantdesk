# Non-goals

grantdesk does three things: pipeline, compliance calendar, win/loss ledger. This document is the list of things it does not do, will not do, and will close feature requests about.

It is written before the first commit on purpose. This is the largest-scope repository in the Egeria program and the only one with meaningful ongoing maintenance cost, which means it is the one most likely to fail by growing rather than by breaking. A data-processing script that nobody extends still works in three years. An application that accepted eleven reasonable feature requests is a product with a support burden, a migration path, a security surface, and a maintainer who has stopped answering issues.

The test applied throughout: **does this make the compliance calendar more reliable, the pipeline more honest, or the ledger more complete?** If the answer is no, it is out, no matter how obviously useful it is in general.

A non-goal is a decision, not a law. If you think one is wrong, open an issue and argue it. What will not work is building it and submitting a pull request.

---

## 1. This is not a CRM

**No contact management.** No people records, no relationship history, no "last touched" on a program officer, no org chart of a foundation's staff. An opportunity has an owner, who is someone on your team. A funder has a name, a type, and a website. That is the extent of it.

This is the boundary most likely to erode, because the request is so reasonable. You do need to know who the program officer is. You will want to note that she is leaving in June. Then you want her phone number in the record, then a log of your calls, then a reminder to follow up, and at that point grantdesk has a contact database with all of the deduplication, merge, privacy, and export obligations that implies, plus a competitor in every CRM ever built. Put the program officer in the opportunity's notes field. It is a text field. It is enough.

**No email integration of any kind.** No inbox sync, no send-from-the-app, no thread capture, no BCC-to-log address, no Gmail or Outlook add-in. grantdesk sends exactly two kinds of email: your sign-in link, and your deadline digest. It reads none.

Email sync is the single largest source of complexity, breakage, and security exposure available to a small tool. It means OAuth applications with two vendors, token refresh, permission scopes that make an IT department nervous, provider API changes on someone else's schedule, and a copy of your correspondence in a database that now needs a very different security posture. There is no version of this that is a weekend of work, and there is no version that stays working without someone maintaining it every quarter.

**No calendar sync as a write target.** grantdesk publishes a read-only iCalendar feed per client and per user, which is a URL you subscribe to in Google Calendar, Outlook, or Apple Calendar. It does not create events in your calendar via an API, does not read your calendar, and does not attempt two-way sync. One-way iCal is thirty lines of code and never breaks. Two-way sync is a permanent job.

**No task management.** An obligation has an assignee, a due date, and a status. It is not a task with subtasks, checklists, dependencies, time estimates, or a Kanban board. If your practice needs task management, you already have a tool for it, and grantdesk's iCal feed and JSON API are how it gets the deadlines.

---

## 2. No money movement, ever

**No donation processing.** No payment forms, no Stripe integration, no donor records, no receipting, no acknowledgment letters, no year-end tax statements. grantdesk records that an award was made, in what amount, and what reporting it obliges. It never touches a dollar.

**No accounting, no general ledger, no fund accounting.** grantdesk does not know your chart of accounts, does not track expenditures against budget line items, does not calculate indirect cost recovery, and does not produce the numbers that go on the SF-425. It tracks that the SF-425 is due on 2027-01-30 and whether it was filed. The figures on it come from your accounting system, where they belong and where an auditor expects to find them.

This distinction is worth stating plainly because it defines the tool. grantdesk is a system of record for **obligations and outcomes**, not for **money**. The moment it holds a budget figure that someone might reconcile against, it is financial software, and financial software carries an accuracy duty that an open-source project maintained on evenings cannot honestly accept.

**No invoicing or time tracking for your consulting practice.** Your award-dollars-per-engagement number is available from the ledger export. Your hours are not grantdesk's business.

---

## 3. No form builder, no application authoring, no submission

**No form builder.** Not for intake, not for client questionnaires, not for internal review. Every form builder becomes a schema editor, a conditional-logic engine, a validation language, and a rendering engine, and then it becomes the thing the project actually is.

**No proposal writing.** No document editor, no narrative templates, no boilerplate library, no version control for drafts, no collaborative editing, no AI drafting. Your proposals live in Google Docs or Word, which are better at documents than anything this project could build. grantdesk stores a link and, optionally, a copy of the final PDF as an attachment.

**No submitting on your behalf.** grantdesk does not file to Grants.gov, does not log into a foundation's Foundant or Submittable portal, does not fill in an application, and does not upload your report. Every one of those integrations is a screen-scraping relationship with a vendor who did not agree to it, breaks without notice, and creates an obligation to hold your credentials to those systems. grantdesk tells you a thing is due, links you to where it gets filed, and records that you filed it.

**No grantmaking side.** grantdesk is for people applying for and managing awards. It is not grant management software for funders: no application intake, no reviewer scoring, no docket assembly, no board packets, no payment scheduling to grantees. That is a different product with a different buyer, and Foundant, Fluxx, and Submittable already sell it.

---

## 4. No reporting content, no compliance judgments

**grantdesk does not generate report content.** It does not draft your performance narrative, does not populate the SF-425, does not compute your federal cash-on-hand, and does not pre-fill anything with numbers it inferred.

**grantdesk does not determine compliance.** It will not tell you that you are compliant, that you are out of compliance, or that a particular expense is allowable. It tracks dates and statuses that you and the built-in templates supplied.

This is a hard line for a specific reason: a tool that appears to make compliance determinations will be relied on as if it does. The award terms in a specific Notice of Award routinely override the general rule in the Uniform Guidance, agencies impose additional conditions, and a high-risk designation changes reporting frequency entirely. grantdesk shows the citation next to every generated obligation precisely so you check it against the actual award document. Every surface carries the disclosure:

> This is informational only, derived from public data on the dates shown. It is not an eligibility determination, and not legal, tax, or accounting advice. Verify against the official source before relying on it.

**No audit preparation module, no single-audit workflow.** grantdesk can hold "single audit submission due" as an obligation with the correct due date. It does not assemble the Schedule of Expenditures of Federal Awards, does not track subrecipient monitoring, and does not manage an audit.

---

## 5. No grant discovery database

grantdesk does not maintain a searchable database of funding opportunities. It does not scrape Grants.gov, does not index foundation 990-PF filings, and does not offer matching or recommendations of its own.

Opportunity discovery is [OpenGrants](https://opengrants.io)' job, and it is the job of the sibling repositories in this program: `funder-graph` for the 990-derived funding graph, `precedent` for who has actually been funded, `grantcheck` for eligibility posture. grantdesk consumes a saved search and turns a match into a pipeline item. It is the downstream system.

If you use grantdesk without an OpenGrants key, you enter opportunities yourself or import them from CSV. That is a fully supported way to use it and always will be.

---

## 6. No public or funder-facing surface

**No client portal.** Your client's executive director does not get a login. There is no read-only dashboard you share with a board, no funder-facing status page, no public URL for an award.

A client portal means an entirely different permission model, a second interface to design and maintain, an audit question about what the client can see, and a support relationship with people who are not your team. What grantdesk offers instead: a PDF or CSV export you send, and the read-only iCal feed you can hand to a program director who wants the deadlines in their calendar.

**Nothing about a tenant is ever indexed or public.** The hosted companion at desk.opengrants.io serves marketing and documentation pages to search engines and to model crawlers. Application routes are excluded at the robots, header, and sitemap level. No client name, opportunity, award, or obligation is ever a public URL. See [`docs/hosted/architecture.md`](hosted/architecture.md).

---

## 7. No plugin system, no scripting, no workflow engine

**No plugins.** A plugin API is an API contract you cannot change, forever, plus a security model for third-party code, plus a registry. Extend grantdesk by forking it or by using the JSON API and MCP server from outside.

**No user-defined automation rules.** No "when an opportunity moves to Awarded, do X." No trigger builder, no condition editor. The one place grantdesk generates work automatically is the obligation recurrence engine, which is deliberately a narrow, well-specified rule type with a preview mode, not a general automation language.

**No webhooks out in version one.** grantdesk receives OpenGrants webhooks. It does not yet emit its own, and adding outbound webhooks means delivery retries, a dead-letter queue, signature key management, and an endpoint health story. If it happens it will be after the calendar and the ledger have been stable for a year.

---

## 8. No custom fields, no configurable schema

Opportunities, submissions, awards, and obligations have the fields they have, plus a notes field and a tag list.

Custom fields sound cheap and are not. They mean a metadata table, a type system, a form renderer, per-field validation, migration behavior when a field is deleted, export column ordering, and reporting that has to handle fields it has never seen. They also destroy the thing that makes the win/loss ledger valuable: if every workspace has different fields, there is no shared definition of "funder type" and therefore no meaningful segmentation in the portfolio export.

The opinionated schema **is** the product. A tool that models grant work specifically is worth more to a grant consultant than a tool that can be configured to model anything, which is what the spreadsheet already is.

---

## 9. No mobile app, no offline mode, no real-time collaboration

The web interface is responsive and works on a phone. There is no native application in an app store, because that is two more platforms, two review processes, and a release cadence.

No offline-first sync, no conflict resolution, no local database. No live cursors, no presence indicators, no websockets. Two people editing the same obligation is handled by an optimistic-concurrency check that tells the second person the record changed, which is the correct amount of engineering for a team of four.

---

## 10. No enterprise identity in version one

No SAML, no SCIM provisioning, no Active Directory, no OAuth sign-in with Google or Microsoft. Authentication is an emailed magic link plus server-side sessions, with API tokens for programmatic access.

This is not a permanent refusal, it is a sequencing decision. Enterprise single sign-on is the correct request from a hospital system or a university, and those organizations are not the initial user. A solo consultant and a six-person firm are, and for them an identity provider integration is a barrier rather than a feature. If SAML arrives it will be because a specific organization needs it and is willing to help test it, not on speculation.

**No per-record permissions.** Access is scoped at the client organization level: a team member can see a client or cannot. There is no field-level security, no record sharing, no approval chain.

---

## 11. No internationalization in version one

United States English, U.S. date handling, U.S. dollars, U.S. federal award structures.

The compliance model is built specifically around the Uniform Guidance, U.S. standard forms, and U.S. foundation practice. That specificity is the value. A generic "reporting deadline" tracker that works everywhere is much less useful to the person this is built for than one that already knows what an SF-425 is and when the final one is due.

Canadian, U.K., and E.U. grant compliance are real and different problems. They are a fork or a sibling project, not a configuration option.

---

## 12. Scale ceiling, stated honestly

grantdesk is designed for a consulting practice with up to roughly fifty client organizations and a few thousand opportunities, on a single SQLite or D1 database.

It is not designed for a fifty-person grant department with a hundred thousand records, and it will not be optimized in that direction. If you outgrow it, that is a good outcome and you should buy something. The JSON API and CSV export exist so that leaving is easy, because a tool you cannot leave is a trap regardless of its license.

---

## What is deliberately in scope, so the boundary is clear

To keep this from reading as a list of refusals, the things grantdesk does commit to:

- Multi-client tenancy enforced at the schema and query layer, from the first commit
- Pipeline stages that match how grant work actually sequences, including the LOI branch
- Obligation recurrence rules with a preview mode, idempotent regeneration, and citations
- Built-in templates for common federal and foundation award shapes, each with a citation and a verification date
- A compliance calendar that spans every client at once
- The win/loss ledger, and the segmented portfolio export built on it
- CSV import and export, a JSON API, an MCP server, and a read-only iCal feed, so your data is never captive
- Both deployment paths, Cloudflare Workers and Docker, at parity
- Optional OpenGrants enrichment that degrades silently to nothing

That is the whole tool. It should be possible to read the source in an afternoon, and it should still be true in five years.
