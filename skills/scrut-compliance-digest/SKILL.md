---
name: scrut-compliance-digest
description: >-
  Produce a leadership-ready snapshot of the organization's compliance program
  from Scrut. Use when the user wants a current-state summary for a board update,
  standup, exec review, or "are we on track" check across frameworks, controls,
  policies, evidence, and continuous tests. Triggers include "compliance
  digest", "where does our SOC 2 program stand", "give me a board update on
  compliance", and "how many policies are still in draft".
version: 1.1.0
---

# Scrut: compliance digest

Assemble a concise, paste-ready status summary of the compliance program:
readiness and next audits, program counts, and what is stale or unfinished.

**The counts must reflect the program the org actually runs.** A Scrut workspace
accumulates items from frameworks that were trialed, scoped out, or imported as
templates. Those items are marked not-relevant and are excluded from the
product's own dashboards. If you count them, your digest will contradict the
dashboard the reader is looking at, and it will invent problems that do not
exist. Scoping is not an optional refinement here — it is the difference between
a correct digest and a misleading one.

## What the terms mean

- **Framework** — a compliance standard the org is pursuing (SOC 2, ISO 27001,
  GDPR, …), each with a `nextAuditDate`.
- **Control** — a requirement within a framework, with an owner and a
  per-framework status.
- **Relevance** (`isRelevant`) — on policies and evidence, `1` means the item is
  in scope for the org's live program and `0` means it is not. The Scrut UI
  filters on this; **the MCP list tools do not**.
- **Policy status** — a policy's lifecycle state: `Not Uploaded`, `Draft`,
  `Published`, `Approved`, `Needs Review`.
- **Evidence status** — `Not Uploaded`, `Draft`, `Uploaded`, `Needs Attention`.
- **Evidence review date** (`reviewDate`) — when the evidence is next due for
  review. A date **in the past** means the evidence is **stale**. `null` means
  **no review scheduled** — that is not stale, and must not be counted as such.
- **Test status** — a continuous automated (CAT) test is `Ok`,
  `Needs Attention` (failing), or `Ignored`.
- **Test scoping** — tests carry no `isRelevant` field except legacy CSPM
  findings (`isExistingCSPM: true`). For tests, **`Ignored` is the scoping
  mechanism**: the active test set is `Ok` + `Needs Attention`.

## Tools this skill uses

All are read-only. Always pass `response_format: "json"` so you can filter and
count reliably.

- `scrut_list_frameworks` — frameworks + `nextAuditDate`.
- `scrut_list_controls` — controls (optionally per `framework_ids`).
- `scrut_list_policies` — policies + status + `reviewDate` + `isRelevant`.
- `scrut_list_evidence` — evidence + status + `reviewDate` + `isRelevant`.
- `scrut_list_tests` — tests + status; `test_status` filters (`ok`,
  `needs_attention`, `ignored`).

**Two traps in these tools:**

1. **The `summary` block ignores relevance.** Its `by_status` counts are
   computed over every item in the workspace, in-scope or not. Use the summary
   only as a reconciliation total — never as the reported numbers.
2. **There is no `is_relevant` request parameter.** Relevance can only be
   applied client-side, per item, after paging. Consequently **do not use
   `limit: 0`** (counts-only): it returns a summary with no items, which cannot
   be scoped.

## Procedure

1. **Set scope.**
   If the user named a framework, resolve it with `scrut_list_frameworks` (match
   on name or `shortKey`) and note its `nextAuditDate`. Otherwise cover all
   enabled frameworks. Capture each framework's `frameworkId` for optional
   filtering.

2. **Pull the program data — page through every item.**
   Call each list tool with `response_format: "json"` and `limit: 200`. The JSON
   carries a pagination envelope (`total_count`, `has_more`, `next_offset`).
   **Loop with `offset = next_offset` until `has_more` is false** — do not
   summarize from only the first page, or the counts will be wrong.
   - `scrut_list_controls` (pass `framework_ids` when scoped to one framework)
   - `scrut_list_policies`
   - `scrut_list_evidence`
   - `scrut_list_tests`

3. **Scope the items before counting.**
   - **Policies and evidence:** keep only items where `isRelevant` is `1`.
     Retain the excluded count — you will need it in step 4.
   - **Tests:** drop `Ignored`. The active set is `Ok` + `Needs Attention`.
     Retain the ignored count.
   - Report the scoped set. Never report a raw workspace total as if it were the
     program.

4. **Reconcile — do this before writing anything.**
   For policies and evidence, `relevant + not-relevant` must equal the
   `summary.by_status` totals for the same statuses. For tests,
   `Ok + Needs Attention + Ignored` must equal `total_count`. If a check fails,
   you have mis-paged or mis-filtered: fix it before reporting. If it still
   fails, say so rather than publishing numbers you cannot reconcile.

5. **Compute the numbers (scoped set only).**
   - **Next audit date** per in-scope framework.
   - **Counts:** controls, in-scope policies, in-scope evidence, active tests.
   - **Policies not finalized:** count `Draft`, `Needs Review`, and
     `Not Uploaded` separately — they mean different things and different work.
   - **Evidence gaps:** count `Not Uploaded`, `Needs Attention`, and `Draft`.
   - **Stale evidence:** `reviewDate` before today. Exclude `null` review dates;
     report those separately as *no review scheduled* if the count is material.
   - **Failing tests:** `Needs Attention` within the active set, with the pass
     rate against the active set (not the workspace total).

6. **Write the digest.**
   Lead with readiness (frameworks + next audit dates), then the counts, then a
   short **"What to watch"** (2-3 lines naming the biggest gaps). State the
   scope in one clause — e.g. "in-scope items only; N template/out-of-scope
   items excluded" — so the reader knows the numbers match their dashboard.
   Keep it tight and leadership-toned; it should paste straight into a doc,
   email, or Slack.

## Worked example

User: *"Give me a compliance digest for the board."*

1. `scrut_list_frameworks` (json) → SOC 2 (next audit 12 Aug 2026), ISO 27001
   (next audit 3 Nov 2026).
2. Page through controls, policies, evidence and tests (json, `limit: 200`)
   until each `has_more` is false.
3. Scope: policies 89 → **60 in-scope** (29 excluded); evidence 359 → **125
   in-scope** (234 excluded); tests 715 → **402 active** (313 ignored).
4. Reconcile: 60 + 29 = 89 ✓ · 125 + 234 = 359 ✓ · 270 + 132 + 313 = 715 ✓.
5. Compute: 383 controls · 60 policies (all Published) · 125 evidence (70
   uploaded, 38 not uploaded, 13 need attention, 4 draft) · 402 active tests
   (270 Ok, 132 need attention — 67% pass).
6. Output:
   > **Compliance snapshot — 21 Aug 2026** *(in-scope items only; 234 template
   > /out-of-scope evidence and 313 ignored tests excluded)*
   > - SOC 2 — next audit **12 Aug**; ISO 27001 — next audit **3 Nov**.
   > - 383 controls · 60 policies · 125 evidence · 402 active tests.
   > - **All 60 policies published.** **51 evidence items open** (38 not
   >   uploaded, 13 need attention). **132 tests need attention (67% pass).**
   >
   > **What to watch:** the failing tests cluster in access reviews and employee
   > policy acknowledgment — assign owners; close the 13 evidence items already
   > flagged before the SOC 2 window.

Note how different this is from the unscoped read of the same workspace, which
would have claimed 21 policies needing review when the correct answer is zero.

## Edge cases and honesty

- **Never invent numbers.** If a list call fails, say which part is unavailable
  rather than guessing a count.
- **Pagination is mandatory** for accurate totals — always follow `has_more`.
- **Relevance is mandatory** for accurate scope — always filter `isRelevant`
  before counting policies and evidence.
- **Watch for relevance drift.** If in-scope items reference frameworks the org
  does not run (e.g. cardholder-data or wireless artifacts with no PCI framework
  enabled), the flag is likely stale from a template import. Report the number,
  and flag that the scope itself needs a human pass — do not silently correct it.
- **Empty results** may mean the connection is pointed at the wrong region/tenant
  or the user's role scopes the data — say so instead of reporting "0 program".
- **Cross-check one dashboard tile** before the digest goes anywhere external.
  If your number and the product's tile disagree, the tile is right and your
  scoping is wrong.
- **Summarize, don't dump.** The output is a digest; do not paste raw lists.

## Changelog

- **1.1.0** — scope correctness: filter `isRelevant` on policies and evidence,
  treat `Ignored` as test scoping, add a mandatory reconciliation step, stop
  counting `null` review dates as stale, split policy/evidence gap statuses,
  drop the `limit: 0` shortcut, and require the digest to state its scope.
- **1.0.0** — initial release: framework/control/policy/evidence/test snapshot
  with next-audit dates, draft/stale/failing counts, and a "what to watch".
