---
name: jc-scrut-device-compliance-evidence
description: >-
  Streams JumpCloud MDM device posture as evidence for a Scrut device-security
  / endpoint control. Use for "device compliance evidence", "endpoint posture",
  "encryption evidence", "are devices encrypted / locked / patched", "MDM
  evidence for Scrut". Read-only against JumpCloud; files the finished evidence
  into Scrut via the Scrut MCP.
version: 1.0.0
---

# JumpCloud × Scrut: device compliance evidence

Report cross-OS endpoint posture from JumpCloud that maps to a Scrut
device-security control, then capture that evidence in Scrut. Read-only against
JumpCloud; the only write this skill performs is filing the finished evidence
into Scrut.

## Inputs

- Optional baseline to check (default: full-disk encryption ON, screen-lock
  policy applied, OS patch current, MDM-managed).
- Optional scope for large fleets (e.g. "stale devices first", a named device
  group, or an OS family). Full-fleet detail costs one `device_get` per device —
  for big fleets, confirm scope with the user before scanning.

## Tools this skill uses

JumpCloud MCP (all read-only):

- `devices_list` — fleet inventory: `hostname`, `os`, `osFamily`, `active`, `agentVersion`, `serialNumber` (max 20 per page)
- `device_get` — per-device posture: `fde.active` / `fde.keyPresent` (disk encryption), `mdm` enrollment (`enrollmentType`, `userApproved`), `osVersionDetail`, `lastContact` (staleness), `secureLogin`
- `policies_list` — device policies (screen lock and other configuration requirements are policy-based, not device attributes)
- `patch_summary_get` — org-wide patch posture summary
- `patch_kbs_list` / `patch_devices_list` — per-device patch detail, only when a deep dive is explicitly needed
- `search_api_execute` — one-call aggregate counts (e.g. devices by OS); not a substitute for per-device posture

Scrut MCP (required for the capture step; skip capture if not connected):

- `scrut_list_evidence` — find the target evidence item (never guess; ask if ambiguous)
- `scrut_get_evidence` — read the item's requirements and mapped controls before filing
- `scrut_upload_file` — upload the evidence file, returns a document object
- `scrut_attach_evidence_document` — attach the document object to the evidence item, with a note

## Workflow (JumpCloud steps read-only; only step 6 writes, and only to Scrut)

1. **Inventory:** `devices_list` (paged via `skip`) → the fleet. Use
   `search_api_execute` for aggregate counts (devices by OS, active vs
   inactive) if a headline breakdown is useful.
2. **Per-device posture:** `device_get` on each in-scope device → encryption
   (`fde.active`), MDM enrollment, OS version (`osVersionDetail`), last
   check-in (`lastContact`). For large fleets, confirm scope first (see
   Inputs) rather than scanning the whole fleet by default.
3. **Screen-lock dimension:** `policies_list` → which devices/groups have a
   screen-lock policy applied. Do not look for a screen-lock field on the
   device object — there isn't one.
4. **Patch dimension:** `patch_summary_get` → org-wide patch posture. Go
   per-device (`patch_kbs_list` → `patch_devices_list`) only when the user
   asks for device-level patch detail.
5. **Evaluate:** classify each device against the baseline — Compliant /
   Non-compliant / Stale (no recent `lastContact`). Summarize by OS and by
   policy dimension (encryption, lock, patch).
6. **Capture the evidence in Scrut** (when the Scrut MCP is connected — this is
   the point of the exercise, not an optional extra):
   a. Save the posture summary + non-compliance table as **CSV** and upload
      with `content_type: text/csv`. `scrut_upload_file` rejects
      `text/markdown` with an opaque server error ("no document object
      returned") — CSV is the known-good format.
   b. `scrut_list_evidence` → find the evidence item for the device control.
      Exactly one clear match → use its `evidenceId`. Zero or several plausible
      matches → show candidates and **ask the user**; never attach to the
      wrong item.
   c. `scrut_get_evidence` → read the item's required elements **before
      filing** and disclose in the attachment note which elements the pack
      satisfies and which it cannot.
   d. `scrut_upload_file` → upload the CSV; keep the returned document object.
   e. `scrut_attach_evidence_document` → attach with a note (e.g. "Device
      compliance evidence generated from JumpCloud on \<date\> — NN% of NN
      devices compliant") plus the element gaps from (c).
   f. Confirm what was attached, to which evidence item (name + id), and the
      resulting status. Expect **Draft / pending approval** — do not claim it
      is "uploaded" or approved.
   If the Scrut MCP is not connected, export the CSV and tell the user to
   attach it to the evidence item manually in Scrut.

## Output

- Headline: % compliant devices; counts of non-compliant and stale.
- Non-compliance table: device, owner, failing dimension(s), last check-in.
- Evidence note mapping the result to the Scrut device control, with the
  scope actually scanned (full fleet vs subset).
- Capture confirmation: the Scrut evidence item (name + id) the evidence was
  attached to, the filename, the note, and the resulting status (expect Draft /
  pending approval) — or, when the Scrut MCP is not connected, the exported
  CSV plus instructions to attach it manually.
- Offer handoff to a gated device-remediation skill for any device action.

## Edge cases and honesty

- **Read-only in JumpCloud.** Never call a device write tool (`device_lock`,
  `device_erase`, `device_restart`, `device_shutdown`, `command_run`, …). The
  only permitted write is the Scrut evidence capture in step 6. Any device
  action is handed to a gated remediation skill — and `device_erase` is never
  offered.
- If `scrut_upload_file` or `scrut_attach_evidence_document` errors, report it
  plainly and stop — do not silently retry or fabricate a successful filing.
  Known upstream quirk: non-CSV MIME types can fail with "no document object
  returned" rather than a validation message naming the accepted types.
- After attach, the evidence item typically sits in Draft pending approval —
  the attach tool's description overstates this as "uploaded". Report the real
  status.
- Screen lock is evidenced via policy application (`policies_list`), not a
  per-device setting — say so in the pack.
- Patch posture is summary-level unless the per-device deep dive was run —
  label which one is being reported.
- State the scan scope (full fleet vs subset) and the staleness threshold used
  for `lastContact`.
- Report posture only; never device contents.

## Changelog

- **1.0.0** — initial release.
