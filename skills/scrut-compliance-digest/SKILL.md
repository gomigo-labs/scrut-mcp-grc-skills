---
name: scrut-compliance-digest
description: >-
  Produce a leadership-ready snapshot of the organization's compliance program
  from Scrut. Use when the user wants a current-state summary for a board update,
  standup, exec review, or "are we on track" check across frameworks, controls,
  policies, evidence, and continuous tests. Triggers include "compliance
  digest", "where does our SOC 2 program stand", "give me a board update on
  compliance", and "how many policies are still in draft".
metadata:
  version: "2.0.0"
---

# Scrut: compliance digest

Assemble a concise, paste-ready status summary of the compliance program:
readiness and next audits, program counts, and what is stale or unfinished.

## What the terms mean

- **Framework** — a compliance standard the org is pursuing (SOC 2, ISO 27001,
  GDPR, …). Each has a `frameworkName`, a `nextAuditDate` (epoch ms) with a
  `nextAuditDateIso` twin, and progress blocks matching the framework cards in
  Scrut: `compliancePercentage` (`percentage` = compliant controls ÷ total
  controls, with `compliantControls` / `totalControls`), plus
  `policiesPublished`, `evidenceUploaded`, and `automatedTests`.
- **Control** — a requirement within a framework. Status labels: Compliant,
  Non Compliant, Not Applicable.
- **Policy status** — Not Uploaded, Draft, Pending Approval, Approved, Needs
  Review, Published. **Draft** means not yet finalized or approved.
- **Evidence status** — Not Uploaded, Draft, Pending Approval, Needs Revision,
  Uploaded, Needs Attention.
- **Review due** — policies and evidence carry a `nextReviewDate`. The
  `next_review_date` filter (`overdue`, `this_month`, `next_month`) narrows a
  list to one window. **Overdue evidence that was uploaded is stale.**
- **Test status** — an automated or cloud test is Passing, Fix Required
  (failing), or Ignored.
- **Scope** — every response carries a `scope` block (`organizationName`,
  `tenantId`, `environment`, `region`, `entity`) naming the workspace the data
  came from.

## Tools this skill uses

All are read-only. Use `response_format: "json"` so you can count reliably.

- `scrut_list_frameworks` — frameworks, next audit dates, and readiness.
- `scrut_list_controls` — control counts by status.
- `scrut_list_policies` — policy counts by status and review due.
- `scrut_list_evidence` — evidence counts by status and review due.
- `scrut_list_tests` — test counts by status.
- `scrut_list_workspaces` / `scrut_list_entities` — only when the user has
  several workspaces or asks about one entity (product or business unit).

## Procedure

1. **Set scope.**
   - **Workspace:** if a call returns `tenant_required`, or the user names a
     company, run `scrut_list_workspaces`, confirm which one, and pass that
     `tenant_id` on every call.
   - **Entity:** if the user names a product or business unit, run
     `scrut_list_entities` and pass its `entity_id` on the list calls.
   - **Framework:** call `scrut_list_frameworks` (it pages with `limit`, max
     50; follow `next_offset` while `has_more` is true). Match the user's
     framework on `frameworkName` and keep its `frameworkId`. If several match
     (e.g. SOC 2 Type I and Type II), ask. Otherwise cover every framework
     returned.

2. **Pull counts with `detail_level: "counts"`.**
   With `detail_level: "counts"`, each list tool returns a `summary` computed
   over **every** matching item and no item rows, so there is nothing to page
   through. Pass `framework_ids: [<frameworkId>]` when scoped to one framework.
   - `scrut_list_controls` → `summary.total_count`, `summary.by_status`.
   - `scrut_list_policies` → `summary.total_count`, `summary.by_status`.
   - `scrut_list_evidence` → `summary.total_count`, `summary.by_status`. Make
     a second counts call with `next_review_date: "overdue"` for stale
     evidence (see step 3).
   - `scrut_list_tests` → `summary.total_count`, `summary.by_status`.

   A status missing from a `by_*` bucket means 0.

3. **Compute the numbers.**
   - **Next audit date** per framework from `nextAuditDateIso`. `null` means no
     audit is scheduled. Say so; do not invent a date.
   - **Readiness** per framework from `compliancePercentage.percentage` (and the
     policy, evidence, and test percentages if useful). A `null` percentage
     means there is nothing to measure yet: report "no data", not 0%.
   - **Counts:** controls, policies, evidence items, tests (`total_count`).
   - **Non-compliant controls:** `controls.summary.by_status["Non Compliant"]`.
   - **Policies in Draft:** `policies.summary.by_status["Draft"]`.
   - **Stale evidence:** from the `next_review_date: "overdue"` call,
     `summary.total_count` minus `summary.by_status["Not Uploaded"]`. The
     overdue window also catches items never uploaded; count those as missing
     evidence, not stale.
   - **Failing tests:** `tests.summary.by_status["Fix Required"]`. This also
     counts tests whose scan is still running.
   - To **name** the items behind a number for "What to watch", list them with
     `detail_level: "lean"` and a filter: `policy_status: "Draft"`,
     `next_review_date: "overdue"` plus
     `evidence_status: ["Draft", "Pending Approval", "Needs Revision", "Uploaded", "Needs Attention"]`
     (stale evidence, without the never-uploaded items), or
     `test_status: "Fix Required"`. Lists return at most 50 rows per call.

4. **Write the digest.**
   Head it with the workspace name (`scope.organizationName`), the entity if
   one was selected, and today's date. Lead with readiness (frameworks, next
   audit dates, percentages), then the counts, then a short **"What to watch"**
   (2–3 lines naming the biggest gaps: non-compliant controls, Draft policies,
   stale evidence, failing tests). Keep it tight and leadership-toned; it
   should paste straight into a doc, email, or Slack.

## Worked example

User: *"Give me a compliance digest for the board."*

1. `scrut_list_frameworks` (json) → SOC 2 (`nextAuditDateIso`
   2026-11-12…, `compliancePercentage.percentage` 87) and ISO 27001
   (2027-02-03…, 74).
   `scope.organizationName` is "Acme".
2. `scrut_list_controls`, `scrut_list_policies`, `scrut_list_evidence`,
   `scrut_list_tests`, each with `detail_level: "counts"` (json).
   Plus `scrut_list_evidence(next_review_date: "overdue", detail_level:
   "counts")`.
3. Compute: 142 controls (`by_status["Non Compliant"]` 12), 31 policies
   (`by_status.Draft` 4), 210 evidence items, 9 overdue of which 3 Not
   Uploaded → 6 stale, 58 tests (`by_status["Fix Required"]` 3).
4. Output:
   > **Acme compliance snapshot — 1 Oct 2026**
   > - SOC 2 — 87% of controls compliant, next audit **12 Nov 2026**;
   >   ISO 27001 — 74%, next audit **3 Feb 2027**.
   > - 142 in-scope controls · 31 policies · 210 evidence · 58 automated/cloud
   >   tests.
   > - **12 controls non-compliant**, **4 policies still in Draft**, **6
   >   evidence items past review**, **3 tests need fixing**.
   >
   > **What to watch:** close the 12 non-compliant controls and finalize the 4
   > Draft policies before the SOC 2 audit; refresh the 6 stale evidence items;
   > triage the 3 failing tests.

## Edge cases and honesty

- **Never invent numbers.** If a list call fails, say which part is unavailable
  rather than guessing a count.
- **Say what the defaults cover.** Controls default to In Scope; policies and
  evidence default to Relevant; tests default to the Organization Wide entity
  and cover automated and cloud tests only (policy and evidence tests are left
  out). Most of these show in `applied_filters`. Name them in the digest when
  it matters (e.g. "in-scope controls").
- **Empty results** may mean the wrong workspace, region, or entity, or that the
  user's role limits what they can see. Check `scope` and say so instead of
  reporting "0 program". `tenant_region_mismatch` means the workspace lives on
  another region's Scrut MCP URL: tell the user to connect that region's URL
  (see the README). The agent cannot switch regions itself.
- **Tool missing.** If a list tool is not on this connection, say which part of
  the digest it would cover and leave it out; do not estimate.
- **Summarize, don't dump.** The output is a digest; do not paste raw lists.

## Changelog

- **2.0.0** — use `detail_level: "counts"` summaries (computed over the full
  set) instead of paging every list. Read the new fields: `status` (was
  `policyStatus` / `evidenceStatus`), `nextReviewDate` and the
  `next_review_date: "overdue"` filter (was `reviewDate`; never-uploaded items
  are no longer counted as stale), test status `Fix Required` (was
  `needs_attention`), `nextAuditDateIso`, and framework readiness percentages.
  Match frameworks on `frameworkName` (`shortKey` was removed). Add
  non-compliant control counts and workspace, entity, and `scope` handling.
- **1.0.0** — initial release: framework/control/policy/evidence/test snapshot
  with next-audit dates, draft/stale/failing counts, and a "what to watch".
