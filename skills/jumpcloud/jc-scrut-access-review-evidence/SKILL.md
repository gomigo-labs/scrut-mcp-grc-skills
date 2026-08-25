---
name: jc-scrut-access-review-evidence
description: >-
  Assembles auditor-ready access-review evidence from JumpCloud to satisfy a
  Scrut access control. Use for "access review evidence", "SOC 2 / ISO 27001 /
  HIPAA evidence", "evidence for Scrut", "who has admin", "who's missing MFA",
  "least privilege proof", "user access certification". Read-only against
  JumpCloud; files the finished evidence pack into Scrut via the Scrut MCP.
version: 1.2.0
---

# JumpCloud × Scrut: access-review evidence

Produce evidence-grade access data from JumpCloud that maps directly to a Scrut
access control, then capture that evidence in Scrut. Never modify anything in
JumpCloud: Scrut owns the control and the framework mapping; this skill
supplies the JumpCloud truth behind it. The only write this skill performs is
filing the finished evidence pack into Scrut.

## Tools this skill uses

JumpCloud MCP (all read-only):

- `admin_list` / `admin_get` — administrators, roles, MFA, suspension state
- `users_list` — users with `state`, `suspended`, and `mfaEnrollment` (max 20 per page)
- `user_get` — full user detail, including the user's groups and the MFA enforcement flag (`enable_user_portal_multifactor`)
- `di_events_get` — Directory Insights events (last-auth evidence)
- `applications_list` / `application_associations_list` — SSO apps and who has access
- `search_api_execute` — one-call org-wide aggregation for counts (e.g. devices by OS). Not reliable for MFA enrollment posture — page `users_list` for that

Scrut MCP (required for the capture step; skip capture if not connected):

- `scrut_list_evidence` — find the target evidence item (never guess; ask if ambiguous)
- `scrut_get_evidence` — read the item's requirements and mapped controls before filing
- `scrut_upload_file` — upload the evidence pack, returns a document object
- `scrut_attach_evidence_document` — attach the document object to the evidence item, with a note

## Inputs

- Optional framework to format for (SOC 2, ISO 27001, HIPAA). Default: generic.
- Optional staleness threshold for dormant access (default 30 days).

## Workflow (JumpCloud steps read-only; only step 6 writes, and only to Scrut)

1. **Privileged access:** `admin_list` → table every admin, role, MFA status,
   suspended state; flag over-privileged or shared admins. Use `admin_get` for
   detail on flagged admins.
2. **MFA posture:** `users_list` (paged via `skip`) → users whose
   `mfaEnrollment.overallStatus` is not `ENROLLED`. Do not substitute a
   `search_api_execute` aggregation here — it has returned misleading MFA
   enrollment rollups. Highest-signal finding; surface first.
3. **Stale access:** first probe the available Directory Insights window —
   request a long `startTime` (e.g. `365d`) and inspect the oldest event
   returned. If retention is shorter than the staleness threshold, report
   dormancy as an **evidence gap** ("not determinable — N days of retention")
   rather than a clean pass. Otherwise `di_events_get` (login event types
   across the `directory` and `sso` services) → users with no auth in the
   window who still hold access (groups via `user_get`; apps via
   `application_associations_list`).
4. **Access map:** `applications_list` → for sensitive apps, who has access and
   via which group (`application_associations_list` — call it **once per
   target type**; the API accepts exactly one target per call despite the
   array schema).
5. **Compile:** severity-rate each finding and map it to the Scrut control it
   supports.
6. **Capture the evidence in Scrut** (when the Scrut MCP is connected — this is
   the point of the exercise, not an optional extra):
   a. Save the findings pack as **CSV** and upload with `content_type:
      text/csv`. `scrut_upload_file` rejects `text/markdown` with an opaque
      server error ("no document object returned") — CSV is the known-good
      format. Keep a markdown copy for humans if useful, but upload CSV.
   b. `scrut_list_evidence` → find the evidence item for the control. Exactly
      one clear match → use its `evidenceId`. Zero or several plausible
      matches → show candidates and **ask the user**; never attach to the
      wrong item.
   c. `scrut_get_evidence` → read the item's required elements and mapped
      controls **before filing**. Note which elements the pack satisfies and
      which it cannot (e.g. review sign-off, corrective-action records, and
      physical-access controls are outside JumpCloud's scope) — disclose this
      in the attachment note instead of letting the reviewer discover it.
   d. `scrut_upload_file` → upload the CSV; keep the returned document object.
   e. `scrut_attach_evidence_document` → attach with a note covering scope,
      date, and the element gaps from (c).
   f. Confirm what was attached, to which evidence item (name + id), and the
      resulting status. Expect the item to land in **Draft / pending
      approval** — do not claim it is "uploaded" or approved.
   If the Scrut MCP is not connected, export the CSV and tell the user to
   attach it to the evidence item manually in Scrut.

## Output

- Executive summary: counts of admins, no-MFA users, stale-access users.
- Findings table: finding, affected identities, severity, recommended
  remediation, mapped Scrut control.
- Evidence note: "Generated from JumpCloud directory state on \<date\>. All
  queries logged in Directory Insights and attachable to the Scrut control as
  evidence."
- Capture confirmation: the Scrut evidence item (name + id) the pack was
  attached to, the filename, the note, and the resulting status (expect Draft /
  pending approval) — or, when the Scrut MCP is not connected, the exported
  CSV plus instructions to attach it manually.

## Edge cases and honesty

- **Read-only in JumpCloud.** Never call a JumpCloud write tool (`user_*`
  writes, `device_*` actions, `admin_update`, group/policy mutations). The only
  permitted write is the Scrut evidence capture in step 6. Hand remediation to
  the gated Gap-to-Fix skill.
- If `scrut_upload_file` or `scrut_attach_evidence_document` errors, report it
  plainly and stop — do not silently retry or fabricate a successful filing.
  Known upstream quirk: non-CSV MIME types can fail with "no document object
  returned" rather than a validation message naming the accepted types.
- After attach, the evidence item typically sits in Draft pending approval —
  the attach tool's description overstates this as "uploaded". Report the real
  status.
- `user_group_membership` lists the members of a group (group → users). For a
  user's groups, use `user_get`.
- `applications_list` responses are large (embedded certificates) — page small
  and extract only the fields needed.
- Directory Insights retention may be shorter than the staleness threshold —
  probe the window first (step 3) and state the window actually queried.
- Report posture only; never secrets or credentials.

## Changelog

- **1.2.0** — correct the MFA field reference (`enable_user_portal_multifactor`,
  not `mfa.exclusion`); drop the `search_api_execute` preference for MFA
  posture after it returned misleading enrollment rollups in a live run.
- **1.1.0** — field-tested against a live org: upload evidence as CSV
  (`text/markdown` is rejected upstream), read the evidence item's requirements
  before filing and disclose element gaps in the note, expect Draft status
  after attach, probe DI retention before staleness claims, call
  `application_associations_list` once per target type.
- **1.0.0** — initial release.
