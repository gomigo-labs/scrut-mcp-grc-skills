---
name: scrut-find-tests
description: >-
  Locate the right Scrut continuous automated tests (CAT) in a large test
  library — hundreds of tests spanning access reviews, evidence, policies,
  vendors and cloud/CSPM checks — without missing whole categories to
  pagination. Use whenever a question depends on which tests exist or what
  they say, before answering from a partial list. Triggers include "is there a
  test for X", "which tests cover encryption", "what's failing in our cloud
  tests", "do we have a test for MFA", "how many tests are failing", and any
  request to search, count, or summarize Scrut tests.
version: 1.0.0
---

# Scrut: find the right tests

Scrut test libraries are large — a typical org has **several hundred to a few
thousand** tests. `scrut_list_tests` returns them in a fixed order with a
default page of 50 and a hard cap of 200, so a naive call reads a small,
**systematically biased** slice of the library and silently misses entire
categories.

This skill is the discovery front-end. Use it before answering any question
whose answer depends on which tests exist.

## The failure this prevents

`scrut_list_tests` returns **CAT module tests first, then legacy cloud/CSPM
tests appended last**. Cloud tests therefore always sit at the *end* of the
sequence.

Measured on a real 714-test org. CAT modules come first in **alphabetical
order of `moduleType`**, then legacy cloud/CSPM tests are appended:

| Offsets | `moduleType` | Count |
|---|---|---|
| 0–99 | `accessReview` | ~100 |
| 100–~469 | `evidence` | ~370 |
| ~470 | `integrations` (Slack, JumpCloud, …) | few |
| ~471–491 | `people` | ~20 |
| 492–~59x | `policy` | ~100 |
| ~59x–595 | `vendor` | few |
| **596–713** | **`test`** — cloud/CSPM, `isExistingCSPM: true` | **118** |

So on that org:

- Default `scrut_list_tests()` (`limit: 50`) returns **50 vendor access-review
  rows and zero cloud tests**.
- Even `limit: 200`, the maximum, stops at offset 200 — still entirely inside
  `evidence`, still **zero cloud tests**.
- Filtering `test_status: "needs_attention"` does not save you: of 124 failing
  tests, the cloud ones are items **#122, #123 and #124**.

An agent that reads page one and answers "here are your failing tests" is
reporting vendor account-review noise and omitting every failing cloud control.
Never answer from a first page.

## Choose the strategy by question shape

### A. "Is there a test for X?" / "Which tests cover X?" — keyword search

**Do not page the list.** Use `scrut_search_documents`, which also searches
tests even though its description emphasizes documents.

1. `scrut_search_documents(query: "<keyword>", response_format: "json")`.
2. Read the **`products`** array and keep entries with `type: "test"`.
3. **Strip the path prefix** from the entry's `id`. There are two forms:
   - `cloud/tests/<uuid>` → use the bare `<uuid>` (cloud/CSPM tests)
   - `tests/<slug>` → use the bare `<slug>`, e.g.
     `tests/integrations_slack-mfa-enabled` → `integrations_slack-mfa-enabled`
4. `scrut_get_test(test_id: "<stripped id>", response_format: "json")`.

> **ID trap — silent empty result.** Passing an unstripped id to
> `scrut_get_test` returns `{"data": null}` with **no error**. Verified for
> both forms. That is a malformed id, *not* a missing test — strip and retry
> before reporting the test does not exist.

**Always pass `limit: 50`** (the maximum). This is the single biggest factor
in whether you find a test. Tests share the `products` bucket with policy and
evidence content chunks, and `limit` applies **per source** — so at the default
of 10, long policy excerpts crowd the tests out entirely:

| `scrut_search_documents(query: "MFA", …)` | Tests returned |
|---|---|
| `limit: 4` | **0** — four policy/evidence chunks only |
| `limit: 50` | **4** — `MFA on root account`, `Slack MFA Enforcement Check`, `MFA on JumpCloud Users`, `IAM assume role lacks external ID and MFA` |

Search ranks best when the query **reads like a test title** — a resource plus
a property (`"IAM user key rotation"`, `"root account MFA enabled"`) rather
than a bare acronym or concept (`"MFA"`, `"encryption"`). If a query returns no
tests, retry with a more title-shaped phrasing before concluding none exist.

Keyword search is a fast *first* attempt, not a completeness guarantee — when
the user asked for a complete list, sweep (strategy C).

### B. "How many tests are failing?" — counts only

`scrut_list_tests(limit: 0)` returns the summary with **no items**: a
`total_count` plus `by_status` and `by_assignee` breakdowns computed over the
**entire** library, pre-pagination. One cheap call, exact numbers.

Use this before any sweep, so you know how much you are about to read and can
verify you read it all.

### C. "Show me everything failing" / a complete list — full sweep

1. `scrut_list_tests(limit: 0)` first to learn the true `total_count`.
2. Page with `limit: 200` and `response_format: "json"`, looping
   `offset = next_offset` **until `has_more` is `false`**.
3. Confirm the number of items you collected equals `total_count`. If it does
   not, keep paging — do not summarize.
4. **Group by `moduleType` before presenting.** Because the ordering is
   grouped, an incomplete read is biased toward whichever module you stopped
   inside. Grouping makes a gap obvious.

Scope the sweep as tightly as the question allows — `test_status:
"needs_attention"` is usually a small fraction of the library.

### D. Cloud/CSPM tests specifically

Cloud tests are the ones people mean by "our S3 test" or "our IAM test":
`moduleType: "test"`, `isExistingCSPM: true`, a bare-UUID `testId`, and a
`targetEntity` slug like `s3-bucket-no-default-encryption`.

There is **no `module_type` filter**, so you cannot request them directly.
Either:

- **search by keyword** (strategy A) when you know the service or control; or
- **sweep and filter client-side** on `moduleType == "test"` (strategy C).

Because cloud tests are appended last, seeking the tail is a legitimate
shortcut: request `offset = total_count - 200` with `limit: 200`, then filter
client-side on `isExistingCSPM: true`. On the measured org that window
(514–713) captured all 118 cloud tests in **one call**.

This only works while the cloud-test count is under 200 — the page you get
also contains the tail of the CAT modules, and if the *first* item already has
`isExistingCSPM: true` you have truncated the cloud tests and must page back
further. The boundary is org-specific and moves as CAT tests are added, so
derive it from `total_count` every time; never hard-code an offset.

## Reading a test's detail

`scrut_get_test` gives status, mapped controls and frameworks, flagged
resources, run history and remediation. Prefer `response_format: "json"` when
you need `flaggedResources` or control mappings.

Two fields to read honestly:

- `flaggedResourceCount` vs `totalFlaggedResourceCount` and
  `flaggedResourcesTruncated: true` — the resource list may be cut off. Say so
  rather than implying you saw every affected resource.
- `detailSource: "list_fallback"` — detail was reconstructed from the list API
  because direct lookup was empty. `remediation` may be `null` and
  `testHistory` empty for these. Fall back to
  `scrut_answer_question("How do we remediate <test title>?")` for guidance
  instead of reporting "no remediation available".

### A single `scrut_get_test` can exceed the response limit

`scrut_get_test` has no field-selection or pagination parameters, so a test
with a long run history can blow the whole call. Measured on
`integrations_slack-mfa-enabled` — **128,387 characters**, rejected outright:

| Field | Size |
|---|---|
| `testHistory` (485 entries) | 94,593 chars |
| `flaggedResources` (100 of 245) | 40,443 chars |
| everything else | ~1,600 chars |

The run history — almost never what you were asked about — is 74% of it.
There is no server-side way to suppress it today. When a call fails this way:

- The output is saved to a file and the error names the path. Pull just what
  you need with `jq` rather than re-requesting, e.g.
  `jq '{title,status,flaggedResourceCount,totalFlaggedResourceCount,remediation}'`.
- Do not retry with `response_format: "markdown"` expecting it to fit — the
  payload is the problem, not the formatting.
- Never treat the failure as "the test doesn't exist."

Prefer reading such tests via the lean list item plus
`scrut_answer_question` for remediation.

## Budget: never bulk-load details

`include_details: true` returns full upstream objects with HTML-laden
`description` and `remediation` bodies — roughly **2,000 tokens per test**. On
a 714-test org a full detailed listing is over **1.4 million tokens**.

- Broad listing → leave `include_details` off (lean items, ~100 tokens each).
- Need depth on specific tests → `scrut_get_test` per test, after narrowing.

A full lean sweep of 714 tests is still ~75K tokens across 4 calls. Prefer
`limit: 0` counts and keyword search; sweep only when completeness is actually
required.

## Scope: workspace and entity

- Multiple workspaces → `scrut_list_workspaces`, then pass `tenant_id` on
  every call. `scrut_get_active_workspace` shows what is in effect.
- CAT tests default to **Organization Wide**, *not* all entities. If a test
  the user expects is missing, list entities with `scrut_list_entities` and
  retry with the relevant `entity_id` before concluding it does not exist.

## Worked example

User: *"Do we have a test for S3 encryption, and is it passing?"*

1. `scrut_search_documents(query: "S3 bucket encryption", response_format: "json")`
   → `products` contains
   `{"id": "cloud/tests/f72b9863-…", "name": "S3 bucket encryption", "type": "test"}`.
2. Strip the prefix → `test_id: "f72b9863-…"`.
3. `scrut_get_test(test_id: "f72b9863-…", response_format: "json")` → status
   `Ok`, `targetEntity: "s3-bucket-no-default-encryption"`, mapped to ISO
   27001:2022 / SOC 2 / GDPR / CMMC.
4. Answer: the test exists, currently passing, and note the frameworks it
   covers.

Paging the list for this would have cost four calls and ~75K tokens — and an
agent stopping at page one would have answered "no such test."

### A failing cloud test

User: *"Our ELBv2 deletion-protection test is failing — what else is wrong
with our load balancers?"*

`scrut_search_documents(query: "ELBv2 deletion protection", limit: 50,
response_format: "json")` returns the target test ranked first **plus its
siblings**, in one ~1,200-token call:

```
cloud/tests/2be4ba9b-…  ELBv2 deletion protection      ← the failing one
cloud/tests/8c79cc8d-…  ELBv2 no access logs
cloud/tests/b2933c86-…  ELBv2 listener allowing cleartext
cloud/tests/598b2156-…  ELBv2 older SSL policy
cloud/tests/b2c80bf7-…  ELB no access logs
cloud/tests/99fb7ba3-…  ELB older SSL policy
cloud/tests/c7b86849-…  ELB listener allowing cleartext
```

Neighbouring-resource recall is a real advantage of search over paging: it
answers the "what else" half of the question for free. Strip the prefixes and
`scrut_get_test` each id you need.

## Edge cases and honesty

- **Never answer "there is no test for X" from one page or one search phrasing.**
  Say which strategy you used and what it covered.
- **Report truncation.** If you stopped paging, say how far you got and that
  the answer is partial.
- **Distinguish `Ignored` from `passing`.** `Ignored` tests are excluded from
  scoring, not passing — and on the measured org they were the single largest
  bucket (314 of 714). Reporting them as healthy overstates posture.
- **`isRelevant: 0`** marks a test the org has deemed not applicable. Don't
  present it as an active gap.
- **Report tool errors plainly** rather than retrying blindly. Partial
  `source_errors` in a search response mean some sources failed — say which.

## Changelog

- **1.0.0** — initial release. Discovery strategies for large test libraries;
  documents the module-ordering bias, the `cloud/tests/` id trap, the
  `limit: 0` counts shortcut, and the `include_details` cost.
