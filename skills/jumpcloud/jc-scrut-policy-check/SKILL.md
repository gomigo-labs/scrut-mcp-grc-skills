---
name: jc-scrut-policy-check
description: >-
  Compares a Scrut policy or control requirement against live JumpCloud
  configuration and identifies gaps or policy drift. Use for "policy check",
  "policy vs configuration", "does JumpCloud match our policy", "policy drift",
  "Device Management policy compliance", "password policy gap". Read-only in
  both systems; files drift findings into Scrut via the Scrut MCP. Never
  remediates — hand off to jc-scrut-remediate to close gaps.
version: 1.0.0
---

# JumpCloud × Scrut: policy → configuration → gap

Catch **policy drift** before it becomes a failed audit test. Scrut holds the
documented requirement; JumpCloud holds the actual configuration. This skill
reads the policy, checks live JumpCloud state against each enforceable
requirement, and reports Met / Gap / Cannot verify — without waiting for a
monitoring cycle to fail.

Read-only in both systems. The only write is filing the gap report in Scrut.
Remediation is handled by `jc-scrut-remediate`, not here.

## What the terms mean

- **Policy requirement** — an enforceable statement extracted from a Scrut
  policy document (e.g. "MFA required for all users", "full-disk encryption
  on all endpoints").
- **Policy drift** — the documented requirement and the live JumpCloud
  configuration disagree.
- **Cannot verify via MCP** — a valid finding when JumpCloud MCP has no tool
  to read the relevant setting (e.g. org-wide password minimum length).

## Inputs

- Optional policy name or topic (e.g. "Device Management", "Access Control",
  "Password Policy"). If omitted, search Scrut for device-management and
  access-related policies and confirm with the user which to check.
- Optional scope for device checks on large fleets — full-fleet `device_get`
  is expensive; confirm before scanning.

## Tools this skill uses

Scrut MCP (policy source + document):

- `scrut_search_documents` — find policies by name or keyword
- `scrut_get_document_content` — read the policy body (exact name match opens
  automatically; partial matches return a candidate list)
- `scrut_upload_file` — upload the gap-report CSV
- `scrut_attach_evidence_document` — attach with a note

JumpCloud MCP (all read-only):

- `admin_list` / `admin_get` — admin-plane MFA and access (`enableMultiFactor`, `totpEnrolled`, `apiKeySet`, `roleName`)
- `users_list` / `user_get` — directory users; enforcement is
  `enable_user_portal_multifactor === true`, never `mfa.exclusion`
- `devices_list` / `device_get` — encryption (`fde.active`), MDM enrollment, `lastContact`
- `policies_list` / `policy_get` — device and configuration policies (screen lock, etc.)
- `patch_summary_get` — org-wide patch posture
- `di_events_get` — optional context on recent configuration changes

## Requirement → JumpCloud verification map

Use this map after extracting requirements from the policy text. Only check
what the policy actually states — do not invent requirements.

| Policy theme | Example requirement | JumpCloud verification |
| --- | --- | --- |
| MFA / authentication | MFA required for admins or users | `admin_list` (`enableMultiFactor`, `totpEnrolled`); directory users via `user_get` (`enable_user_portal_multifactor`) |
| Privileged access | Admin access restricted, least privilege | `admin_list` — roles, live API keys, suspended state |
| Endpoint encryption | Full-disk encryption on all devices | `devices_list` + `device_get` (`fde.active`) |
| Screen lock | Auto-lock / timeout enforced | `policies_list` — screen-lock policies applied to device groups; no per-device screen-lock field |
| Patch / OS currency | Devices kept patched | `patch_summary_get` (summary-level; per-device only on explicit request) |
| MDM enrollment | Devices must be managed | `device_get` (`mdm.enrollmentType`, `mdm.userApproved`) |
| Password complexity | Minimum length, complexity rules | **Cannot verify via MCP** — no password-policy read tool; report as limitation and point to JumpCloud console |
| Offboarding / access removal | Accounts revoked on termination | `users_list` + `di_events_get` for suspension events — partial; HR roster not in JumpCloud |

## Workflow (read-only until Phase 4; Phase 4 writes only to Scrut)

### Phase 1 — Find and read the policy (Scrut, read-only)

1. `scrut_search_documents` for the policy name or topic the user named.
   Common starting points: Device Management, Access Control, Password,
   Acceptable Use, Information Security.
2. If zero or several matches → show candidates and **ask the user** which
   policy to check.
3. `scrut_get_document_content` → read the full policy body.
4. Extract **enforceable requirements** — concrete, testable statements.
   Ignore aspirational language ("we strive to…"). List each requirement with
   a short ID (R1, R2, …) for the gap table.

### Phase 2 — Check JumpCloud (read-only)

For each extracted requirement, use the verification map above:

1. Call the matching JumpCloud tool(s).
2. Record live evidence: counts, sample objects, policy names, or the specific
   gap.
3. If no MCP tool can verify the requirement, mark **Cannot verify via MCP**
   — do not guess or infer from unrelated fields.
4. For MFA on directory users: never use `mfa.exclusion` as the enforcement
   signal — use `enable_user_portal_multifactor`. For admins: use
   `enableMultiFactor` from `admin_list`.
5. For large device fleets, confirm scope before running per-device checks.

### Phase 3 — Report gaps

Produce a gap table with one row per requirement:

| Req | Policy statement | Status | Live evidence / gap |
| --- | --- | --- | --- |
| R1 | … | Met / Gap / Cannot verify | … |

Headline: N requirements checked · X met · Y gaps · Z cannot verify via MCP.

Offer handoff to `jc-scrut-remediate` for any gap the user wants to close —
this skill does not remediate.

### Phase 4 — File in Scrut (when the Scrut MCP is connected)

1. Save the gap table as **CSV** with columns: requirement_id, policy_text,
   status, live_evidence, jumpcloud_tools_used.
2. Upload with `content_type: text/csv`. `scrut_upload_file` rejects
   `text/markdown` with an opaque server error — CSV is the known-good format.
3. `scrut_list_evidence` → find the evidence item. Exactly one clear match →
   use its `evidenceId`. Zero or several plausible matches → show candidates
   and **ask the user**; never attach to the wrong item.
4. `scrut_get_evidence` → read required elements **before filing**; disclose
   in the attachment note which elements this report satisfies (configuration
   reconciliation) and which it cannot (formal policy approval, manual
   console-only settings).
5. `scrut_upload_file` → upload the CSV; keep the returned document object.
6. `scrut_attach_evidence_document` → attach with a note (e.g. "Policy drift
   check: [policy name] vs JumpCloud on \<date\> — Y gaps found") plus element
   gaps from step 4.
7. Confirm what was attached, to which evidence item (name + id), and the
   resulting status. Expect **Draft / pending approval**.

If the Scrut MCP is not connected, export the CSV and tell the user to attach
it manually in Scrut.

## Output

- Policy checked: name, id, date read.
- Requirements extracted: numbered list with enforceable text.
- Gap table: Met / Gap / Cannot verify per requirement with live evidence.
- Headline counts and severity on gaps (prioritize MFA, encryption, admin access).
- Capture confirmation (if filed): evidence item, filename, note, Draft status.
- Handoff offer to `jc-scrut-remediate` for closable gaps.

## Edge cases and honesty

- **Read-only everywhere.** No JumpCloud writes, no Scrut writes except evidence
  filing in Phase 4. Never call remediation tools from this skill.
- Only assess requirements **present in the Scrut policy text** — do not
  import requirements from frameworks or controls unless the user asks to
  cross-check a specific control via `scrut_get_control`.
- "Cannot verify via MCP" is a legitimate outcome — especially password
  complexity, some IdP routing rules, and HR-linked offboarding. Say so plainly;
  do not fabricate compliance.
- Screen lock is evidenced via `policies_list`, not a device attribute.
- Patch checks are summary-level via `patch_summary_get` unless the user
  requests per-device detail.
- If `scrut_upload_file` or `scrut_attach_evidence_document` errors, report it
  plainly and stop. Known quirk: non-CSV MIME types fail with "no document
  object returned".
- After attach, expect Draft / pending approval — do not claim "uploaded" or
  approved.
- This skill finds drift; `jc-scrut-remediate` closes it. Keep the boundary clear.

## Changelog

- **1.0.0** — initial release: policy-first gap analysis, requirement-to-tool
  map, read-only in both systems, CSV evidence capture.
