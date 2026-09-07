---
name: scrut-fix-test
description: >-
  Remediate a failing Scrut continuous cloud test (CAT) from the user's IDE:
  find the failing test, gather remediation guidance and the flagged resources,
  help edit the infrastructure-as-code, and direct the user to re-run the test in
  Scrut to verify. Use when a cloud/config compliance test is failing and the
  user wants to fix it where their code lives. Triggers include "fix this failing
  test", "remediate our public S3 buckets", "what's failing in our cloud tests",
  and "our encryption test is failing, help me fix it".
version: 1.2.0
---

# Scrut: fix a failing cloud test

Find a failing continuous automated test (CAT), understand what broke, fix the
underlying infrastructure code in the repo, and point the user to Scrut to verify
— without leaving the editor.

## What the terms mean

- **CAT test** — a Scrut continuous automated test that checks a cloud/config
  requirement (e.g. "S3 buckets must block public access").
- **`needs_attention`** — the status of a failing test (vs. `passing` or
  `ignored`).
- **Flagged resource** — a specific cloud resource that caused the test to fail;
  its type/config appears in the test detail payload.

## Tools this skill uses

- `scrut_search_documents` — keyword-find a test by name (returns tests in its
  `products` array, not just documents).
- `scrut_list_tests` — enumerate failing tests (`test_status:
  "needs_attention"`).
- `scrut_get_test` — full detail for one test (`response_format: "json"` to see
  mapped controls and any flagged resources/config).
- `scrut_answer_question` — remediation guidance from the knowledge base.
- `scrut_get_document_content` — a related policy/evidence, if useful.

## Procedure

1. **Find the failing test.** Test libraries run to hundreds of tests, and
   `scrut_list_tests` returns **cloud/CSPM tests last** — so a default page of
   50 typically contains *no cloud tests at all*. Pick by situation:

   - **User named a test or a service** ("our S3 public-access test") →
     `scrut_search_documents(query: "<name or service>", response_format:
     "json")`, take `products` entries with `type: "test"`, and **strip the
     `cloud/tests/` prefix** from the `id` to get the `test_id`. Passing the
     unstripped id to `scrut_get_test` returns `{"data": null}` with no error.
   - **User asked what's failing** → `scrut_list_tests(limit: 0)` for exact
     counts, then page `test_status: "needs_attention"` with `limit: 200`,
     looping `offset = next_offset` until `has_more` is `false`. Group by
     `moduleType` and confirm your item count matches `total_count` before
     presenting. Cloud tests are `moduleType: "test"` /
     `isExistingCSPM: true`.

   Never present a failing-test list from a single default-limit call — on a
   measured 714-test org the cloud tests were items #122–124 of 124 failing
   tests. For the full strategy set, see the **`scrut-find-tests`** skill.

2. **Get the detail.**
   Call `scrut_get_test` with the `test_id` and `response_format: "json"`. Note
   the mapped controls and any **flagged resources** and their config in the
   payload — this is what you need to write a correct fix.

   Check two fields before you rely on the payload:
   - `flaggedResourcesTruncated: true` (or `flaggedResourceCount` <
     `totalFlaggedResourceCount`) means you are **not** seeing every affected
     resource. Fix what you can see and tell the user the list was truncated.
   - `detailSource: "list_fallback"` means detail was reconstructed from the
     list API; `remediation` may be `null` and `testHistory` empty. That is not
     "no guidance exists" — go to step 3.

3. **Gather remediation guidance.**
   Call `scrut_answer_question` (e.g. *"How do we remediate <test name>?"*) for
   the fix steps. Optionally call `scrut_get_document_content` for a related
   policy or evidence if it clarifies the required end state.

4. **Fix it in the repo.**
   Locate the relevant infrastructure-as-code (Terraform, CloudFormation, etc.)
   for the flagged resource and propose the change. **Show the diff and confirm
   with the user before applying** — this edits real infrastructure code.

5. **Verify in Scrut.**
   After the change is applied, tell the user to **re-run the test in Scrut**
   (Test Details → Test Now) and check the updated status there. This skill does
   not trigger a scan from the MCP.

   **Optional (user-driven):** If the user has already re-run the scan in Scrut,
   they may ask you to call `scrut_get_test` again to read the current status.
   If status is still `needs_attention`, note that results may not have updated
   yet — check again later in Scrut; do not claim the fix failed immediately.

## Why verification is in Scrut

MCP-triggered test re-runs are not available yet — test runs can be slow today,
and we're improving that before exposing re-run through the MCP. For now,
re-run the test in Scrut to verify your fix. Triggering a re-scan from your
agent will return in a future update. Relay this briefly if the user asks why
there is no automatic re-run from the agent.

## Worked example

User: *"Our S3 public-access test is failing — fix it."*

1. `scrut_search_documents(query: "S3 bucket public access", response_format:
   "json")` → `products` contains
   `{"id": "cloud/tests/<uuid>", "name": "S3 bucket world star policy",
   "type": "test"}`. Strip the prefix → `test_id: "<uuid>"`.
2. `scrut_get_test(test_id: "<uuid>", response_format: "json")` → flagged
   buckets, mapped controls, `targetEntity: "s3-bucket-world-policy-star"`.
3. `scrut_answer_question("How do we remediate public S3 buckets?")` →
   block-public-access guidance.
4. Edit Terraform to add `aws_s3_bucket_public_access_block` for the flagged
   buckets; show the diff; user approves.
5. "Terraform updated. Re-run this test in Scrut (Test Details → Test Now) to
   confirm it passes. MCP re-runs from your agent are coming in a future update.
   If you've already re-run it, I can check the latest status with
   `scrut_get_test`."

## Edge cases and honesty

- **Never pick the test from a first page.** `scrut_list_tests` orders
  cloud/CSPM tests last, so a default-limit call is biased toward access-review
  and evidence rows. Search by keyword, or page to completion.
- **A `{"data": null}` from `scrut_get_test` means a malformed id**, not a
  missing test — strip any `cloud/tests/` prefix and retry before telling the
  user the test does not exist.
- **Confirm before editing infra code.** Show the diff; do not silently rewrite
  IaC.
- **Do not claim "fixed" from the agent alone.** Verification requires a
  re-run in Scrut (or the user asking you to poll `scrut_get_test` after they
  re-ran it).
- **Report tool errors** plainly instead of looping.

## Changelog

- **1.2.0** — fix test discovery: search-first by keyword (`scrut_search_documents`
  returns tests), document the `cloud/tests/` id trap and the module-ordering
  bias that hid cloud tests behind hundreds of CAT tests, require paging to
  completion for failing-test lists, and handle `flaggedResourcesTruncated` /
  `detailSource: "list_fallback"`. Defers to the new `scrut-find-tests` skill.
- **1.1.0** — remove `scrut_run_test` dependency; verification is user-driven
  in Scrut (optional `scrut_get_test` poll after user re-runs). MCP re-runs
  deferred pending performance improvements.
- **1.0.0** — initial release.
