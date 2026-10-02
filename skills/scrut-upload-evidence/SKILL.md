---
name: scrut-upload-evidence
description: >-
  File an artifact into Scrut as evidence. Use when the user has produced or
  received a document (access review, config export, vendor report, screenshot,
  signed policy) or a link and wants it attached to the right evidence item in
  Scrut. Triggers include "file this as evidence", "attach this to <evidence or
  control>", "upload this access review to Scrut", and "add this report as
  evidence".
metadata:
  version: "2.0.0"
---

# Scrut: upload and attach evidence

Attach a file or a link to the correct evidence item in Scrut, so an artifact the
user just produced becomes filed evidence without leaving the conversation.

## What "evidence" means here

- **Evidence item** — a record in Scrut that collects proof for one or more
  controls. It has an `evidenceId`, an `evidenceName`, a `status`, assignees,
  and a `nextReviewDate`. You attach documents to it; you do not create it here.
- **Signed upload** — the Scrut MCP never receives file bytes. It returns a
  short-lived `uploadUrl`. Your agent PUTs the file there directly, then calls
  the tool again with the `upload_token` so Scrut attaches it.
- **Evidence link** — an external URL attached in place of a file (for artifacts
  that already live somewhere accessible).
- **Note** — free text stored alongside the attachment (context, not the date).
- **Evidence date** — when the artifact is *as of*, as epoch **milliseconds**
  (13 digits, e.g. 2026-09-24 00:00 UTC = `1790208000000`). It defaults to now.
  Only set it to record a different date (e.g. an access review actually run
  last week), and confirm the date with the user.

## Tools this skill uses

Pass `response_format: "json"` on every call; the field paths below
(`evidence[]`, `attached`, `uploadToken`, `headers`, `scope`) refer to the JSON
response.

- `scrut_search_documents` — find evidence items by name (read `evidence[]`).
- `scrut_list_evidence` — browse evidence items when a name search misses.
- `scrut_list_controls` + `scrut_get_control_by_id` — resolve a control code
  (e.g. "CC6.3") to the evidence mapped to it.
- `scrut_get_evidence_by_id` — confirm the target, and the attachment afterwards.
- `scrut_upload_evidence` — the only write tool. Signed file upload, or attach a
  link.
- `scrut_list_workspaces` — only to pick the workspace when the user has
  several.

## Procedure

1. **Identify the target evidence item — do not guess.**
   - **By name or description:** call `scrut_search_documents` with a short
     `query` that appears word-for-word in the evidence name (case does not
     matter), e.g. `"access review"`, and read `evidence[]` (`id`, `name`). It
     is a plain substring match, so "Q3 access review" will not find "User
     Access Reviews"; try a shorter term if nothing matches. It returns up to 10
     per source (raise `limit`, max 50). If it still misses, page through
     `scrut_list_evidence` (`limit` max 50; repeat with
     `offset = next_offset` while `has_more` is true).
   - **By control code:** page through `scrut_list_controls` the same way until
     you find the row whose `controlCode` matches. There is no code filter;
     pass `framework_ids` if you know the framework, and
     `control_scope: "Out of Scope"` only if the control is out of scope. If
     the code matches in several frameworks, ask which one. Then call
     `scrut_get_control_by_id` and pick from its `mappedArtifacts` where
     `artifactType` is `"Evidence"`; flag any whose `isRelevant` is
     `"Not Relevant"`.
   - The id comes back as `id` (search), `evidenceId` (evidence list), or
     `artifactId` (control). Whichever it is, pass it as `evidence_id`.
   - Zero or several plausible matches → show candidate names + ids and **ask
     the user which one**. Never attach to the wrong item just to finish.
   - `scrut_list_evidence` hides Not Relevant evidence by default. Pass
     `relevance: "Not Relevant"` only if the user is looking for one of those.
     `scrut_search_documents` does not hide them.
   - Optional: `scrut_get_evidence_by_id` shows the item's status, current
     period, and existing `attachments` before you add to it.

2. **Decide file vs. link.**
   - **Link** — the artifact already lives at a URL → confirm (step 3), then
     attach (step 5).
   - **File** — the file is on the machine the agent runs on → confirm (step
     3), then upload (step 4). If the
     content only exists in the chat, write it to a local file first. This
     path needs a way to send an HTTP PUT (a shell with `curl`, or an HTTP
     tool). If the agent cannot run commands or make HTTP requests (for example
     a chat-only client), say so and offer the link path or uploading in the
     Scrut app instead.
   - **Never** put file contents or base64 into a tool call. The tool does not
     accept them.

3. **Confirm before writing.**
   State the workspace (`scope.organizationName` from the step 1 response), the
   evidence item (name + id), the file or link, the note, and the date. Wait for
   the user's go-ahead unless they named the exact evidence item themselves.
   Then run the upload without pausing — the upload link expires.

4. **File: upload (prepare → PUT → complete).** Several files → repeat this
   step once per file.
   - **Prepare:** call `scrut_upload_evidence` with `evidence_id`, `filename`
     (include the extension), and `content_type` (e.g. `application/pdf`; the
     default is `application/octet-stream`). `display_name` is optional. The
     response has `attached: false`, `uploadUrl`, `headers`, `uploadToken`, and
     `expiresInSeconds`.
   - **PUT the bytes** to `uploadUrl`, sending every header in `headers`
     exactly as returned. Finish before `expiresInSeconds` runs out. Use single
     quotes so the shell does not expand anything in the path or URL (escape a
     `'` inside a path as `'\''`):

     ```bash
     curl -sS --fail -X PUT \
       -H 'Content-Type: <headers.Content-Type>' \
       --upload-file '<local file path>' \
       '<uploadUrl>'
     ```

   - **Complete:** call `scrut_upload_evidence` with the same `evidence_id`,
     `upload_token` (the `uploadToken` from prepare), and optional `note` and
     `evidence_date`. The response has `attached: true` and the stored
     `documentObject`.

   `note` and `evidence_date` are **ignored on prepare**. Pass them on the
   complete call. `uploadUrl` and `uploadToken` grant write access until they
   expire: use them only in the PUT and the complete call, and do not repeat
   them in your reply, notes, files, or commits.

5. **Link: attach (one call).**
   Call `scrut_upload_evidence` with `evidence_id`, `evidence_link` (a full
   `https://…` URL), and optional `note` / `evidence_date`. Do not pass
   `filename` or `upload_token` on this call.

6. **Report.**
   Say which evidence item (name + id) in which workspace, what was attached
   (filename or link), and the note and date you sent (the response does not
   echo them; with no `evidence_date` the date is the time of the call).
   Optionally re-read with `scrut_get_evidence_by_id` to show the new entry in
   `attachments`.

## Worked example

User: *"Attach this Q3 access review PDF to our user access reviews evidence."*

1. `scrut_search_documents(query: "access review", response_format: "json")`
   → `evidence[]` has one item, "User Access Reviews" (`id: ev_…`);
   `scope.organizationName` is "Acme".
2. The file is local and the agent has a shell → file path.
3. "I'll attach **q3-2026-access-review.pdf** to **User Access Reviews**
   (`ev_…`) in Acme with the note 'Q3 2026 user access review', dated today.
   Go ahead?" → user confirms.
4. `scrut_upload_evidence(evidence_id: "ev_…", filename:
   "q3-2026-access-review.pdf", content_type: "application/pdf",
   response_format: "json")` →
   `uploadUrl`, `headers`, `uploadToken`.
   `curl -sS --fail -X PUT -H 'Content-Type: application/pdf' --upload-file
   './q3-2026-access-review.pdf' '<uploadUrl>'` → 200.
   `scrut_upload_evidence(evidence_id: "ev_…", upload_token: "<uploadToken>",
   note: "Q3 2026 user access review", response_format: "json")` →
   `attached: true`.
5. Report (step 6): "Attached **q3-2026-access-review.pdf** to **User Access
   Reviews** (`ev_…`) in Acme with the note 'Q3 2026 user access review', dated
   today."

## Edge cases and honesty

- **Ambiguous or missing target** → ask and list candidates. Attaching to the
  wrong evidence is worse than pausing to confirm.
- **Several workspaces.** If a call returns `tenant_required`, or the user has
  more than one workspace, run `scrut_list_workspaces`, confirm which one, and
  pass that `tenant_id` on **every** call, including prepare and complete. The
  upload token only works for the evidence item and workspace it was issued
  for.
- **PUT failed** → report the HTTP status. On 403 or an expired link, check
  that you sent every header in `headers` exactly, then run prepare once more
  and retry the PUT. If it fails again, stop and report. For any other status,
  stop and report.
- **Complete says the object was not found** (`upload_failed`) → the PUT never
  landed. Redo the PUT once (if the link expired, prepare again and use the new
  `upload_token`), then complete. If it is still not found, stop and report.
- **Complete fails with "File uploaded but not attached"** (the error details
  carry `uploadToken`) → the file is stored but not attached. Retry complete
  **once** with the same `upload_token`; do not upload the file again. If that
  fails, report that the file is uploaded but not attached, and stop.
- **Token expired.** The `upload_token` expires with the upload link
  (`expiresInSeconds`, default 1 hour). After that, start again from prepare.
- **Any other `scrut_upload_evidence` error** (e.g. the user's role cannot
  write) → report it plainly and stop. Never claim success without
  `attached: true`.
- **Tool missing.** If `scrut_upload_evidence` is not on this connection, say
  so; do not look for another way to write.
- **One source per call.** A file goes through prepare + complete. A link goes
  through a single call with `evidence_link`. Do not mix them.
- **You are not reading the file into Scrut's knowledge.** This skill files an
  artifact; it does not index its contents.

## Changelog

- **2.0.0** — move to `scrut_upload_evidence` (signed upload: prepare → HTTP
  PUT → complete, or a single link call). `scrut_upload_file` and
  `scrut_attach_evidence_document` were removed from the Scrut MCP, and file
  bytes no longer pass through the model. Find evidence with
  `scrut_search_documents`, or by control code via `scrut_get_control_by_id`.
  Confirm before writing; add workspace (`tenant_id`) handling and bounded
  retry rules.
- **1.0.0** — initial release: find evidence, upload a file or attach a link,
  with note and optional as-of date.
