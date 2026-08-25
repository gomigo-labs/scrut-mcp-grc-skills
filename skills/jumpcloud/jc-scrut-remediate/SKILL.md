---
name: jc-scrut-remediate
description: >-
  Closes a failing Scrut control in JumpCloud with human approval — detect the
  finding in Scrut, investigate live JumpCloud state, remediate behind a gate,
  verify, and document the result back in Scrut. Use for "fix the gap",
  "remediate", "enforce MFA on these users", "suspend the terminated accounts",
  "close the finding", "detect remediate verify document". JumpCloud writes are
  gated; Scrut writes are evidence filing only.
version: 1.0.0
---

# JumpCloud × Scrut: detect → remediate → verify → document

Take a failing Scrut control from red to remediated-and-documented in one
workflow: find the failure in Scrut, investigate the live JumpCloud state,
apply an approved fix behind a strict gate, verify what changed, and file the
remediation record in Scrut.

This skill is the write-capable counterpart to the read-only evidence skills.
Every JumpCloud change requires explicit human confirmation. The only Scrut
write is filing the remediation evidence.

## What the terms mean

- **CAT test** — a Scrut continuous automated test (`scrut_list_tests` /
  `scrut_get_test`); status `needs_attention` means failing.
- **Framework control** — a Scrut control mapped to a framework
  (`scrut_list_controls` / `scrut_get_control`); use when the failure is an
  identity or access control rather than a cloud/config test.
- **Remediation record** — a CSV documenting what failed, what was changed in
  JumpCloud, per-step status, and the Directory Insights trail.

## Tools this skill uses

Scrut MCP (detect + document):

- `scrut_list_tests` — find failing CAT tests (`test_status: "needs_attention"`)
- `scrut_get_test` — full detail for one test (flagged resources, mapped controls)
- `scrut_list_controls` — find failing framework controls when not a CAT test
- `scrut_get_control` — control detail and mappings
- `scrut_list_evidence` — find the evidence item to attach the remediation record
- `scrut_get_evidence` — read required elements before filing
- `scrut_upload_file` — upload the remediation CSV
- `scrut_attach_evidence_document` — attach the document with a note

JumpCloud MCP (investigate — read-only until Phase 3):

- `users_list` / `user_get` — resolve affected users; enforcement is
  `enable_user_portal_multifactor === true`, never `mfa.exclusion`
- `admin_list` / `admin_get` — admin-plane MFA (`enableMultiFactor`, `totpEnrolled`)
- `devices_list` / `device_get` — resolve affected devices
- `di_events_get` — audit trail before/after remediation

JumpCloud MCP (remediate — only after typed CONFIRM):

- `user_add_mfa` — set MFA requirement (grace period via `exclusionUntil`, default 7d)
- `user_suspend` — suspend account (reversible via `user_reactivate`)
- `user_force_password_reset` — force password reset on next login
- `device_lock` — remotely lock a device

## Supported remediations (closed list)

| Gap | JumpCloud action | Reversible |
| --- | --- | --- |
| User missing MFA requirement | `user_add_mfa` | Requirement can be removed manually |
| Terminated / dormant account still active | `user_suspend` | `user_reactivate` |
| Compromised or stale credentials | `user_force_password_reset` | User resets on next login |
| At-risk unmanaged device | `device_lock` | User unlocks with device PIN/password |

Anything outside this list — policy configuration (password length, screen-lock
policy, patch enforcement), group membership changes, app access changes, admin
role changes — has **no MCP write tool**. Recommend the fix and document the
finding; do not pretend to execute it.

Never call `device_erase`, `user_delete`, `device_restart`, `device_shutdown`,
`command_run`, or any group/policy/app mutation from this skill.

## Procedure

### Phase 1 — Detect (Scrut, read-only)

1. If the user named a failing test or control, match it directly.
2. Otherwise call `scrut_list_tests` with `test_status: "needs_attention"`.
   If nothing matches (e.g. "MFA enforced" is a framework control, not a CAT
   test), call `scrut_list_controls` and find the failing control.
3. Get full detail: `scrut_get_test` or `scrut_get_control` with
   `response_format: "json"`. Note mapped controls, evidence items, and any
   flagged resources.
4. Restate the finding in plain language before touching JumpCloud.

### Phase 2 — Investigate (JumpCloud, read-only)

1. Resolve each affected identity or device (`user_get`, `admin_get`,
   `device_get`). If the set is ambiguous, list candidates and ask.
2. Confirm the gap in live JumpCloud state — do not rely on stale Scrut sync
   data alone.
3. For MFA gaps: check `enable_user_portal_multifactor` on directory users or
   `enableMultiFactor` / `totpEnrolled` on admins. Never use `mfa.exclusion` as
   the enforcement signal.
4. Optionally pull recent `di_events_get` events for the affected objects to
   establish a before-state audit trail.

### Phase 3 — Remediate (JumpCloud, gated writes)

**Read before write. Preview before execute. STOP until CONFIRM.**

1. **Preview (no writes):** present one clear plan naming every write call and
   every object, e.g.:
   > I will enforce MFA on 6 users [list] / suspend 4 accounts [list] / lock 2
   > devices [list]. Type **CONFIRM** to proceed.
2. **STOP.** Do not call any write tool until the admin types `CONFIRM`.
3. **Execute (post-CONFIRM):** apply the matching fix **one item at a time**,
   reporting each step's success or failure. If any step fails, stop and surface
   the error — do not continue silently.
4. Never batch a destructive action on more than one object without a second,
   counted confirmation naming the exact count.

### Phase 4 — Verify and document

**Verify (two tiers — be honest about both):**

1. **JumpCloud (immediate):** re-read each affected object. For MFA, verify the
   **requirement** is set (`enable_user_portal_multifactor` or
   `enableMultiFactor`) — enrollment completes on the user's next login, often
   after a grace period (`exclusionUntil`).
2. **Scrut (deferred):** tell the user to re-run the test or wait for the next
   monitoring cycle in Scrut to confirm the control flips. Do **not** claim
   "control green" or "finding closed" from the agent alone. If the user has
   already re-run in Scrut, they may ask you to call `scrut_get_test` or
   `scrut_get_control` again to read current status.

**Document (Scrut evidence capture):**

1. Save the remediation record as **CSV** with columns: finding, object,
   action taken, per-step status, timestamp, DI event reference.
2. Upload with `content_type: text/csv`. `scrut_upload_file` rejects
   `text/markdown` with an opaque server error — CSV is the known-good format.
3. `scrut_list_evidence` → find the evidence item. Exactly one clear match →
   use its `evidenceId`. Zero or several plausible matches → show candidates
   and **ask the user**; never attach to the wrong item.
4. `scrut_get_evidence` → read required elements **before filing**; disclose
   in the attachment note which elements this record satisfies (corrective
   action taken) and which it cannot (formal review sign-off, etc.).
5. `scrut_upload_file` → upload the CSV; keep the returned document object.
6. `scrut_attach_evidence_document` → attach with a note covering the finding,
   actions taken, verification status, and element gaps.
7. Confirm what was attached, to which evidence item (name + id), and the
   resulting status. Expect **Draft / pending approval**.

## Output

- Finding restatement: what failed, in Scrut and in JumpCloud.
- Remediation plan (pre-CONFIRM) and execution log (post-CONFIRM).
- Verification: JumpCloud re-read results + honest Scrut re-check guidance.
- Capture confirmation: evidence item, filename, note, Draft status — or export
  instructions if the Scrut MCP is not connected.

## Edge cases and honesty

- **No writes before CONFIRM.** Phase 1 and 2 are read-only. Phase 3 writes
  only after typed `CONFIRM`. Phase 4's Scrut filing is the only other write.
- Operate only on the named items from Phase 2. Do not widen scope.
- `user_add_mfa` sets the requirement; it does not instantly enroll the user.
  Report requirement-set, not enrolled, unless re-read shows enrollment.
- `apiKeySet: true` admins bypass interactive MFA — suspending or resetting
  may not address API-key authentication; call this out.
- If `scrut_upload_file` or `scrut_attach_evidence_document` errors, report it
  plainly and stop. Known quirk: non-CSV MIME types fail with "no document
  object returned".
- After attach, expect Draft / pending approval — do not claim "uploaded" or
  approved.
- Config/policy gaps (password length, screen lock, patch policy) → recommend
  only; no MCP write exists.
- Never expose authentication secrets or credentials.

## Changelog

- **1.0.0** — initial release: detect in Scrut, investigate in JumpCloud,
  gated remediate (closed fix list), verify with honest two-tier status,
  document via CSV evidence capture.
