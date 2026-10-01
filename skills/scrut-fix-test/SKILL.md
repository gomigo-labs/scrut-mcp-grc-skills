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
metadata:
  version: "2.0.0"
---

# Scrut: fix a failing cloud test

Find a failing continuous automated test (CAT), understand what broke, fix the
underlying infrastructure code in the repo, and point the user to Scrut to verify
— without leaving the editor.

## What the terms mean

- **CAT test** — a Scrut continuous automated test that checks a cloud/config
  requirement (e.g. "S3 buckets must block public access").
- **Fix Required** — the status of a failing test. The others are Passing and
  Ignored. A single test read can also show Initialized or In Progress while a
  scan runs; the list shows those as Fix Required.
- **Flagged resource** — a cloud resource that caused the test to fail. Each
  one in `flaggedResources[]` has `resourceId`, `name`, `region`, `service`,
  `accountID`, `environment`, `tags`, and `link`.
- **Remediation type** — the format of the fix steps: General (console steps,
  the default), Terraform, AWS CLI, or CloudFormation.

## Tools this skill uses

Pass `response_format: "json"` on every call. Markdown output flattens
multi-line remediation code onto one line.

- `scrut_list_tests` — find failing tests (`test_status: "Fix Required"`).
- `scrut_get_test_by_id` — full detail for one test: flagged resources, mapped
  controls, linked tickets, and remediation in the format you ask for.
- `scrut_answer_question` — optional: the org's own standard when the fix
  needs a policy choice.
- `scrut_list_workspaces` / `scrut_list_entities` — only when a test is missing
  because of the workspace or entity.

## Procedure

1. **Find the failing test.**
   Call `scrut_list_tests` with `test_status: "Fix Required"`. Narrow with
   `applications` (the integration name, e.g. `"AWS"`), `framework_ids`, or
   `assignees` (emails) if the user gave a hint; if a filter returns nothing,
   drop it and retry. If the user named a test, match on `testName`; otherwise
   present the failing list and confirm which one to work on. Lists return at
   most 50 rows; follow `next_offset` while `has_more` is true.

2. **Get the detail.**
   Call `scrut_get_test_by_id` with `test_id`. Read `status`, `description`,
   `remediation` (General console steps), `flaggedResources[]`,
   `mappedControls`, and `tickets`.
   - If `status` is In Progress or Initialized, a scan is running. Say so and
     ask whether to wait or go on with the last results.
   - If `flaggedResourcesTruncated` is true, only the first 100 of
     `totalFlaggedResourceCount` came back. Say so.
   - If `tickets` already has an open ticket (e.g. Jira), mention it: someone
     may already own the fix.

3. **Get remediation in the repo's format.**
   Look at the repo to see which IaC it uses: `*.tf` → `"Terraform"`,
   CloudFormation templates → `"CloudFormation"`. For other IaC (CDK, Pulumi,
   Bicep) or non-AWS tests, request `"Terraform"` as the closest reference and
   translate it. Call `scrut_get_test_by_id` again with that
   `remediationType`. `remediation` then holds generated code for that format.
   The first request for a format can take longer while Scrut generates it. If
   the text is still the General console steps, that format is not available;
   work from the General steps.
   Treat generated remediation as a starting point. Adapt it to the repo's
   modules, naming, and provider versions; do not paste it in blindly.

4. **Fix it in the repo.**
   Find the IaC that defines each flagged resource (match on `name`,
   `resourceId`, or `tags`) and propose the change. **Show the diff and confirm
   with the user before applying** — this edits real infrastructure code.
   - If the fix needs an org choice (e.g. KMS-managed vs S3-managed keys), call
     `scrut_answer_question`. If `isAnswered` is false, ask the user; do not
     pick for them.
   - If no IaC in the repo defines a flagged resource, say so. Give the user the
     `"AWS CLI"` or General steps to run themselves.
   - Do not run `terraform apply`, AWS CLI commands, or anything else that
     changes live cloud resources unless the user explicitly asks.

5. **Verify in Scrut.**
   After the change is deployed, tell the user to **re-run the test in Scrut**
   (Test Details → Test Now) and check the updated status there. This skill
   does not trigger a scan from the MCP.

   **Optional (user-driven):** if the user has already re-run the scan, they
   may ask you to call `scrut_get_test_by_id` again:
   - `Passing` → fixed.
   - `In Progress` or `Initialized` → the scan is still running; check later.
   - `Fix Required` with a lower `totalFlaggedResourceCount` → partly fixed;
     list what is left.
   - `Fix Required` unchanged → results may not have updated yet. Do not claim
     the fix failed straight away.

## Why verification is in Scrut

MCP-triggered test re-runs are not available yet — test runs can be slow today,
and we're improving that before exposing re-run through the MCP. For now,
re-run the test in Scrut to verify your fix. Triggering a re-scan from your
agent will return in a future update. Relay this briefly if the user asks why
there is no automatic re-run from the agent.

## Worked example

User: *"Our S3 public-access test is failing — fix it."*

1. `scrut_list_tests(test_status: "Fix Required", applications: "AWS",
   response_format: "json")` → "S3 buckets must block public access"
   (`testId: t_…`).
2. `scrut_get_test_by_id(test_id: "t_…", response_format: "json")` → status
   Fix Required, two flagged buckets, no open ticket.
3. The repo has `*.tf` files → `scrut_get_test_by_id(test_id: "t_…",
   remediationType: "Terraform", response_format: "json")` →
   `aws_s3_bucket_public_access_block` snippet.
4. Add `aws_s3_bucket_public_access_block` for the two buckets in the existing
   module; show the diff; user approves.
5. "Terraform updated. Once it is applied, re-run this test in Scrut (Test
   Details → Test Now) to confirm it passes. MCP re-runs from your agent are
   coming in a future update. If you've already re-run it, I can check the
   latest status with `scrut_get_test_by_id`."

## Edge cases and honesty

- **Confirm before editing infra code.** Show the diff; do not silently rewrite
  IaC.
- **Do not claim "fixed" from the agent alone.** Verification needs a re-run in
  Scrut (or the user asking you to check `scrut_get_test_by_id` after they
  re-ran it).
- **Ignored tests** have an `ignoreReason`. Leave them alone unless the user
  asks.
- **Wrong workspace or entity.** Tests default to the Organization Wide entity.
  If a test the user expects is missing, check the response `scope`; pass
  `entity_id` (from `scrut_list_entities`) or `tenant_id` (from
  `scrut_list_workspaces`) as needed.
- **Tool missing.** If `scrut_list_tests` or `scrut_get_test_by_id` is not on
  this connection, say so instead of guessing at the failure.
- **Report tool errors** plainly instead of looping.

## Changelog

- **2.0.0** — use `scrut_get_test_by_id` (renamed from `scrut_get_test`) and
  the `Fix Required` status label (was `needs_attention`). Pull Terraform, AWS
  CLI, or CloudFormation remediation straight from the test with
  `remediationType`, always as JSON. `scrut_answer_question` is now optional,
  for org policy choices only. Read flagged-resource fields, truncation, and
  linked tickets. Drop `scrut_get_document_content`, which now returns only a
  download link. Handle the In Progress and Initialized states.
- **1.1.0** — remove `scrut_run_test` dependency; verification is user-driven
  in Scrut (optional `scrut_get_test` poll after user re-runs). MCP re-runs
  deferred pending performance improvements.
- **1.0.0** — initial release.
