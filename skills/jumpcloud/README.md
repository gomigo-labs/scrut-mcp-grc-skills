# JumpCloud × Scrut skills

Co-branded skills that assume **both** the JumpCloud MCP server and the Scrut
MCP server are connected. JumpCloud supplies the identity, access, and device
truth (read-only, or gated writes behind a typed CONFIRM); Scrut owns the
control and receives the filed evidence.

| Skill | What it does | Say something like |
| --- | --- | --- |
| [jc-scrut-access-review-evidence](jc-scrut-access-review-evidence) | Assembles auditor-ready access-review evidence (admins, MFA gaps, stale access) and files it in Scrut. | "Assemble SOC 2 access-review evidence." |
| [jc-scrut-mfa-coverage](jc-scrut-mfa-coverage) | Proves MFA enforcement/enrollment across the JumpCloud admin plane; files the coverage evidence. | "What's our admin MFA coverage? File it for the MFA control." |
| [jc-scrut-device-compliance-evidence](jc-scrut-device-compliance-evidence) | Reports endpoint posture (encryption, MDM, patch, staleness) and files it in Scrut. | "Prove our fleet is encrypted and managed." |
| [jc-scrut-policy-check](jc-scrut-policy-check) | Reads a Scrut policy and checks the live JumpCloud configuration against each requirement — policy drift before it fails a test. Read-only. | "Does JumpCloud match our Device Management policy?" |
| [jc-scrut-remediate](jc-scrut-remediate) | Detect → remediate → verify → document: closes a failing Scrut control in JumpCloud behind a typed-CONFIRM gate, then files the remediation record. | "Scrut flagged users without MFA — fix it after I confirm." |
| [jc-scrut-offboard](jc-scrut-offboard) | Interval-based offboarding evidence (sampling, human review) for Scrut; secondary gated per-person execution mode. | "Generate offboarding evidence for this month." |

## Prerequisites

1. **JumpCloud MCP server** connected (Admin Portal → Settings → Features →
   JumpCloud AI → MCP Server; endpoint `https://mcp.jumpcloud.com/v1`, API key
   or OAuth). The agent inherits the connecting admin's permissions.
2. **Scrut MCP server** connected for the evidence-filing steps (see the repo
   root README). Without it, each skill degrades to exporting a CSV you attach
   manually in Scrut.

## Safety contract

- Discovery and evidence skills are read-only against JumpCloud.
- The two write-capable skills (`jc-scrut-remediate`, `jc-scrut-offboard`
  Mode B) never act without a typed `CONFIRM`, operate only on the named
  items, prefer reversible actions, and never call `user_delete`,
  `device_erase`, or other destructive tools.
- Evidence is uploaded as CSV (`text/csv`); after attach, the Scrut evidence
  item sits in Draft pending approval.
