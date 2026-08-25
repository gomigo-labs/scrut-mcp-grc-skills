---
name: jc-scrut-mfa-coverage
description: >-
  Produces MFA coverage evidence for a Scrut MFA control from JumpCloud's admin
  plane — the users added to manage the workspace. Use for "MFA coverage",
  "who's missing MFA", "MFA evidence for Scrut", "prove MFA is enforced",
  "authentication control evidence". Read-only against JumpCloud; files the
  finished evidence into Scrut via the Scrut MCP.
version: 1.3.0
---

# JumpCloud × Scrut: MFA coverage proof

Prove the state of MFA enrollment and enforcement across the JumpCloud **admin
plane** — the users added to manage the workspace — as evidence for a Scrut
MFA control, then capture that evidence in Scrut. Read-only against JumpCloud;
the only write this skill performs is filing the finished evidence into Scrut.

## Scope

The control population is **JumpCloud administrators only** (`admin_list`) —
not the general directory. Do not enumerate all directory users for this
control. Report directory-wide numbers only as clearly-labeled supplemental
context, and only if the user explicitly asks.

## Tools this skill uses

JumpCloud MCP (all read-only):

- `admin_list` — the control population: every admin's `enableMultiFactor` (requirement), `totpEnrolled` (enrollment), `suspended`, `apiKeySet`, `sessionCount`
- `admin_get` — detail on a flagged admin
- `users_list` / `user_get` — only if the user explicitly widens scope to directory users (see edge cases before doing so)

Scrut MCP (required for the capture step; skip capture if not connected):

- `scrut_list_evidence` — find the target evidence item (never guess; ask if ambiguous)
- `scrut_get_evidence` — read the item's requirements and mapped controls before filing
- `scrut_upload_file` — upload the evidence file, returns a document object
- `scrut_attach_evidence_document` — attach the document object to the evidence item, with a note

## Workflow (JumpCloud steps read-only; only step 5 writes, and only to Scrut)

1. **Enumerate:** `admin_list` → every admin's `enableMultiFactor` (MFA
   required), `totpEnrolled` (enrolled), and `suspended` state. This is the
   whole control population — no directory-wide paging.
2. **Classify:** Enforced-and-enrolled / Enforced-not-enrolled / Not-enforced.
   Use `admin_get` for detail on flagged admins.
3. **Prioritize:** not-enforced or not-enrolled admins are critical findings.
   Flag admins with `apiKeySet: true` — API keys bypass interactive MFA, so
   enrollment is not the whole story for key-holders. Note stale or shared
   admin accounts.
4. **Compute coverage %:** enforced admins / active admins, with enrollment
   reported alongside (requirement and enrollment diverge during grace
   periods).
5. **Capture the evidence in Scrut** (when the Scrut MCP is connected — this is
   the point of the exercise, not an optional extra):
   a. Save the coverage summary + gap table as **CSV** and upload with
      `content_type: text/csv`. `scrut_upload_file` rejects `text/markdown`
      with an opaque server error ("no document object returned") — CSV is the
      known-good format.
   b. `scrut_list_evidence` → find the evidence item for the MFA control.
      Exactly one clear match → use its `evidenceId`. Zero or several plausible
      matches → show candidates and **ask the user**; never attach to the
      wrong item.
   c. `scrut_get_evidence` → read the item's required elements **before
      filing** and disclose in the attachment note which elements the pack
      satisfies and which it cannot.
   d. `scrut_upload_file` → upload the CSV; keep the returned document object.
   e. `scrut_attach_evidence_document` → attach with a note (e.g. "MFA coverage
      evidence generated from JumpCloud on \<date\> — NN.N% enforced") plus the
      element gaps from (c).
   f. Confirm what was attached, to which evidence item (name + id), and the
      resulting status. Expect **Draft / pending approval** — do not claim it
      is "uploaded" or approved.
   If the Scrut MCP is not connected, export the CSV and tell the user to
   attach it to the evidence item manually in Scrut.

## Output

- Headline: admin-plane MFA coverage % and the count of gaps.
- Gap table: admin, role, MFA required/enrolled, live API key, severity.
- Evidence note mapping the result to the Scrut MFA control, with the
  Directory Insights trail.
- Capture confirmation: the Scrut evidence item (name + id) the evidence was
  attached to, the filename, the note, and the resulting status (expect Draft /
  pending approval) — or, when the Scrut MCP is not connected, the exported
  CSV plus instructions to attach it manually.
- Offer handoff to the gated `jc-scrut-remediate` skill to enforce MFA on the
  gap list.

## Edge cases and honesty

- **Read-only in JumpCloud.** Never call a JumpCloud write tool. The only
  permitted write is the Scrut evidence capture in step 5. Enforcement is
  handled only by the gated `jc-scrut-remediate` skill.
- If `scrut_upload_file` or `scrut_attach_evidence_document` errors, report it
  plainly and stop — do not silently retry or fabricate a successful filing.
  Known upstream quirk: non-CSV MIME types can fail with "no document object
  returned" rather than a validation message naming the accepted types.
- After attach, the evidence item typically sits in Draft pending approval —
  the attach tool's description overstates this as "uploaded". Report the real
  status.
- Admin-plane MFA fields are `enableMultiFactor` (requirement) and
  `totpEnrolled` (enrollment) from `admin_list`. If scope is ever widened to
  directory users, enforcement there is `enable_user_portal_multifactor ===
  true` on `user_get` — **never** `mfa.exclusion`, which is `false` by default
  for every user (enrolled or not, required or not) and would report ~100%
  coverage when the truth may be near zero.
- "Enforced" means the MFA **requirement** is set; report requirement and
  enrollment separately — they diverge during grace periods.
- `apiKeySet: true` admins can authenticate via API key without interactive
  MFA — call this out alongside the MFA table.
- Never expose authentication secrets.

## Changelog

- **1.3.0** — scope the control population to the JumpCloud admin plane
  (`admin_list`) instead of the whole directory; add the API-key MFA-bypass
  flag.
- **1.2.0** — fix a dangerous misclassification: enforcement is
  `enable_user_portal_multifactor === true`, not `mfa.exclusion` (which is
  `false` by default for all users and would have reported ~100% coverage).
  Scope `search_api_execute` to the aggregate enforcement count only.
- **1.1.0** — field-tested against a live org: upload evidence as CSV
  (`text/markdown` is rejected upstream), read the evidence item's requirements
  before filing and disclose element gaps in the note, expect Draft status
  after attach.
- **1.0.0** — initial release.
