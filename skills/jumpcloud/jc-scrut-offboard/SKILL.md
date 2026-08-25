---
name: jc-scrut-offboard
description: >-
  Generates human-reviewed offboarding evidence for a time interval from
  JumpCloud and files it in Scrut — who was offboarded between two dates, what
  access was revoked, sampled and confirmed by the reviewer before upload. Use
  for "offboarding evidence", "who was offboarded this month/quarter",
  "offboarding evidence upload", "generate offboarding evidence for August".
  Includes a secondary, gated per-person execution mode for power users
  ("offboard this person", "suspend the employee", "revoke access for").
version: 2.0.0
---

# JumpCloud × Scrut: offboarding evidence (interval) + gated execution

Two modes, in priority order:

- **Mode A — Interval evidence collection (primary, read-only in JumpCloud).**
  For a given period (month/quarter), find who was offboarded, let the reviewer
  pick the sample, and file the evidence in Scrut. This is the intended
  customer workflow: offboarding itself originates in a people system (e.g.
  Deel) and is executed by IT in the JumpCloud UI or scripts; Scrut is the
  audit repository, and this skill produces the periodic, human-reviewed
  evidence for it.
- **Mode B — Per-person execution (secondary, gated, power users).** Suspend a
  specific person and revoke access from the agent, behind a typed-CONFIRM
  gate. Most IT managers will not run sensitive offboarding actions from a chat
  interface — this mode is a template for power users, not the primary trigger.

The only Scrut write in either mode is filing evidence. Mode A never writes to
JumpCloud. Mode B writes to JumpCloud only after explicit confirmation.

## What the terms mean

- **Interval evidence** — one evidence file per period (e.g. "August 2026")
  containing a sample of offboardings, not one file per employee.
- **Sample** — the subset of offboarded users the reviewer chooses to include.
  Auditors expect a sample with a review trail, not an exhaustive dump.
- **Empty-window attestation** — when nobody was offboarded in the period, the
  evidence is a statement to that effect. That is a valid, fileable result.
- **Before-state footprint** (Mode B) — the person's groups, apps, devices, and
  account state captured before any change.

## Inputs

- Optional period: "this month", "last quarter", or explicit dates. If omitted,
  infer from the evidence item's cadence and confirm the exact window before
  querying.
- Optional sample size or selection criteria (e.g. "include one per department").
- Mode B only: the person's email or name, optional termination date, and which
  Tier-2 hygiene items to include.

## Tools this skill uses

JumpCloud MCP (read-only in Mode A; gated writes in Mode B):

- `users_list` / `user_get` — user state (`state`, `suspended`, `account_locked`), groups, bound devices
- `di_events_get` — suspension/deprovisioning events in the window (`startTime`, `eventType`, `initiatorId`; limit ≤ 1000)
- `admin_list` / `admin_get` — whether an offboarded person held admin rights
- `applications_list` / `application_associations_list` — app access the person held (Mode B footprint; once per target type)
- Mode B writes (post-CONFIRM only): `user_suspend` (Tier 1); `user_force_password_reset`, `user_reset_mfa`, `manage_user_group_membership`, `unbind_user_from_device` (Tier 2); `device_lock` (at-risk device only)

Scrut MCP (document):

- `scrut_list_evidence` — find the offboarding / access-revocation evidence item
- `scrut_get_evidence` — read the item's description/requirements (the quality bar) and current status before filing
- `scrut_upload_file` — upload the evidence CSV
- `scrut_attach_evidence_document` — attach with a note

## Mode A — Interval evidence collection (primary)

### A1. Determine the window

Ask or infer the period (e.g. "month of August" = Aug 1 through today). State
the exact start and end dates you will query before proceeding. Directory
Insights retention caps how far back the window can reach — if the requested
window exceeds retention, state the window actually covered.

### A2. Find offboardings in the window (read-only)

1. `di_events_get` with `startTime` covering the window → suspension and
   deprovisioning events. Filter by `eventType` / `query` — note that
   `initiatorId` filters by the **actor** (who performed the action), not the
   offboarded user. Event-type names vary; discover the suspension event types
   from the returned events rather than assuming a fixed name. Capture user,
   timestamp, and initiator for each offboarding found.
2. Cross-reference `users_list` / `user_get` → confirm each candidate is
   currently `suspended: true` (or otherwise deprovisioned).
3. Note anyone flagged by `admin_list` as having held admin rights.
4. **If nobody was offboarded in the window:** the evidence is an
   empty-window attestation ("No employees were offboarded between \<start\>
   and \<end\>"). Say so plainly — never invent rows.

### A3. Human review of the sample (required)

Present the findings table: email, suspension date, initiator, admin rights
held, groups/apps they had. The reviewer confirms which rows go into the
evidence file (all, or a chosen sample). **Do not upload before the reviewer
confirms the sample.** This review is the control — the skill never auto-files.

### A4. Sufficiency check against the evidence item

`scrut_list_evidence` → find the offboarding evidence item (ask if ambiguous,
never guess) → `scrut_get_evidence` → read its description/requirements as the
quality bar. If prior attachments are visible, note how much evidence already
exists for the period and flag if the new sample looks redundant or
insufficient against the stated requirements. Surface this before upload.

### A5. File in Scrut

1. Build the CSV with exactly this schema — do not improvise the format:

```csv
period_start,period_end,user_email,action,action_date,initiated_by,admin_rights,evidence_source,notes
2026-08-01,2026-08-24,ada@example.com,user_suspended,2026-08-12,it-admin@example.com,false,directory_insights,"groups: eng, sso-prod"
```

   For an empty window, one row: the period, `action` = `none_offboarded`,
   empty user fields, and a note stating the attestation.
2. Upload with `content_type: text/csv`. `scrut_upload_file` rejects
   `text/markdown` with an opaque server error ("no document object
   returned") — CSV is the known-good format.
3. `scrut_upload_file` → keep the document object →
   `scrut_attach_evidence_document` with a note naming the window, sample size,
   and any element gaps from A4 (e.g. "Offboarding evidence Aug 1–24: 5
   offboarded, 3 sampled. Formal HR termination letters are out of scope.").
4. Confirm what was attached, to which item (name + id), and the resulting
   status. Expect **Draft / pending approval** — do not claim "uploaded" or
   approved.

If the Scrut MCP is not connected, export the CSV and tell the user to attach
it manually in Scrut.

## Mode B — Per-person execution (secondary, power users, gated)

Most offboarding is initiated in a people system and executed by IT in the
JumpCloud UI or scripts. This mode exists for power users who choose to execute
from the agent. All writes are gated.

### B1. Identify and footprint (read-only)

1. Resolve the person via `users_list` → `user_get`. Zero or multiple matches →
   show candidates and ask.
2. Capture the before-state footprint: account state, groups, SSO apps
   (`applications_list` + `application_associations_list`), bound devices, and
   admin status (`admin_list`, note `apiKeySet`).
3. Already suspended → report idempotent state; skip to verification and let
   Mode A pick the person up in the next interval run.
4. If the person is a **JumpCloud admin**, call it out and require explicit
   acknowledgment before the preview.

### B2. Preview (no writes)

> **Tier 1:** suspend [email] (`user_suspend`).
> **Tier 2 (if requested):** force password reset · reset MFA · remove from
> groups [list] · unbind devices [list].
> **At-risk (if flagged):** lock device [hostname] (`device_lock`).
>
> Type **CONFIRM** to proceed.

Batch Tier-2 removals include the count ("remove from 6 groups — confirm 6").
**STOP** until typed `CONFIRM`.

### B3. Execute (post-CONFIRM)

Suspend first, then Tier-2 items one at a time, reporting each. Stop on first
error and surface it. Suspension is reversible via `user_reactivate` — say so.

### B4. Verify (read-only)

Re-read the user (`suspended: true`); pull `di_events_get` for the suspension
event with timestamp and initiator; spot-check any Tier-2 removals. Report only
what re-read clean.

### B5. Document

Default: the offboarding becomes a row in the **next Mode A interval run**
(sampling model) — not a standalone upload. Only file an individual record if
the user explicitly asks, using the Mode A5 format with a single row.

## Output

- Mode A: window statement, findings table, confirmed sample, sufficiency
  notes, capture confirmation (item, filename, note, Draft status).
- Mode B: before-state footprint, preview plan, execution log, verification
  results, and a pointer that the record will be included in the next interval
  evidence run (or an individual filing if requested).

## Edge cases and honesty

- **Manual trigger by design.** MCP connections require periodic
  re-authentication, so this skill cannot run on a schedule — the cadence is a
  reminder to the IT manager, who runs the skill and reviews. Say this if asked
  about automation.
- **Sampling, not exhaustiveness.** One evidence file per period with a
  reviewer-chosen sample; never one file per employee unless asked.
- **Empty windows are valid evidence.** Attest "no offboardings this period" —
  never fabricate rows.
- **DI retention** limits historical windows; state the window actually covered.
- **The reviewer confirms the sample before upload.** No auto-filing.
- **Fixed CSV schema.** Use the Mode A5 format exactly; do not improvise.
- Mode B: read-only until CONFIRM; never `user_delete`, `device_erase`,
  `device_shutdown`, `device_restart`, `command_run`; one named person only;
  admin offboarding requires explicit pre-preview acknowledgment.
- If `scrut_upload_file` or `scrut_attach_evidence_document` errors, report it
  plainly and stop — do not silently retry or fabricate a successful filing.
- After attach, expect Draft / pending approval.
- Complements `jc-scrut-remediate` (closes a failing control). For finding
  accounts that should have been offboarded, run a dormant/terminated-access
  discovery pass (`users_list` + `di_events_get`) before this skill.

## Changelog

- **2.0.0** — repositioned after partnership review: primary mode is
  interval-based, human-reviewed, sampling-based evidence collection with a
  fixed CSV schema and a sufficiency check against the evidence item's
  requirements; per-person execution retained as a labeled secondary mode whose
  results flow into the next interval run.
- **1.0.0** — initial release (execution-first design).
