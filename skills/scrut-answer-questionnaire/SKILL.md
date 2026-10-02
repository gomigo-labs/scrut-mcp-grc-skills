---
name: scrut-answer-questionnaire
description: >-
  Draft answers to a security or vendor questionnaire from the organization's
  compliance posture using Scrut's knowledge base, with the sources per answer
  and an explicit gap flag when Scrut isn't confident. Use when the user pastes
  or points at a set of security questions (vendor security review, CAIQ, DDQ,
  RFP security section). Triggers include "answer this questionnaire", "fill this
  vendor security review", "respond to these security questions", and "draft an
  answer to this from our compliance posture".
metadata:
  version: "1.1.0"
---

# Scrut: answer a security questionnaire

Turn a list of security questions into drafted answers grounded in the org's
documented posture, each with its sources, and honestly flag anything Scrut
cannot confidently answer. **Never fabricate an answer.**

## How this works

This skill answers with a single tool: **`scrut_answer_question`**. That tool asks
Scrut's own backend to answer the question — Scrut runs the retrieval and answer
synthesis (RAG) over the organization's knowledge base and returns a finished
answer. The MCP does not run a model here; it forwards the question and returns
what Scrut synthesized. Your agent only spends its own tokens invoking the tool
and reading the result.

## What the terms mean

- **`scrut_answer_question`** — asks Scrut to answer one natural-language
  question from the org's knowledge base (policies, Trust Vault, product docs).
  Returns:
  - `answer` — the synthesized answer text (or null).
  - `references[]` — the sources Scrut grounded the answer on.
  - `isAnswered` — a confidence flag; `false` means Scrut was **not** confident
    enough to answer.
- **`type`** (optional) — a scope hint (e.g. `"product-document"`). Omit for the
  default scope unless the user clearly wants to narrow it.
- **`scope`** — every response names the workspace it answered from
  (`scope.organizationName`, `scope.tenantId`). Answers must come from the
  user's own workspace.

## Tools this skill uses

- `scrut_answer_question` — the only tool in scope for questionnaire autofill.
- `scrut_list_workspaces` — only to pick the workspace when the user has
  several. It does not answer anything.

## Procedure

1. **Confirm the workspace.** The workspace must belong to the company
   **answering** the questionnaire (the vendor being assessed), not the
   customer who sent it. If the user has several Scrut workspaces (a call
   returns `tenant_required`, or it is unclear which company is answering), run
   `scrut_list_workspaces`, confirm which one, and pass that `tenant_id` on
   every call. Check `scope.organizationName` on the first answer.

2. **Split the questionnaire into individual questions.** Preserve any numbering
   so answers map back cleanly. For a long questionnaire, tell the user the
   question count first and report progress in batches.

3. **Answer each question with `scrut_answer_question`.** Pass the question text,
   `response_format: "json"` (so each `references` entry arrives whole), and a
   `type` hint only if the user asked to narrow scope. Read back `answer`,
   `references`, and `isAnswered`.

4. **Decide coverage per question — this is the core of the skill.**
   - **Answered:** `isAnswered` is `true`. Include the `answer` **and** its
     `references` as the cited sources. If `references` is empty, treat it as
     needs-human.
   - **Needs human / no coverage:** `isAnswered` is `false` (or `answer` is
     empty). **Do not write an answer.** Mark it for a human. If useful, note
     that the knowledge base didn't have a confident source.

5. **Assemble the output.** For each question return: the question, the drafted
   answer (or a clear "needs human — Scrut not confident" marker), the
   `references`, and the coverage flag. End with a one-line coverage summary
   (e.g. "34 answered, 6 need a human").

## Worked example

User: *"Answer this vendor security questionnaire from our compliance posture."*
— includes *"Do you encrypt customer data at rest?"*

1. One workspace → no `tenant_id` needed; `scope.organizationName` on the first
   answer is "Acme", the company answering.
2. Split into questions.
3. `scrut_answer_question(question: "Do you encrypt customer data at rest?",
   response_format: "json")` → `isAnswered: true`, `answer: "Yes — customer
   data is encrypted at rest using …"`, `references: [ …Encryption Policy… ]`.
4. Coverage: **answered** — include the answer + its references.
5. In the output that row shows the answer, its sources, and the answered flag; a
   question that comes back `isAnswered: false` shows "needs human — Scrut not
   confident" with no drafted answer.

## Edge cases and honesty (non-negotiable)

- **Never fabricate.** An unanswered question is a correct, useful result — a
  made-up answer is a trust failure. Always honor `isAnswered: false`.
- **Cite every answer.** Ship the `references` Scrut returned with each answer;
  don't present an answer with no source.
- **Answer per question.** Call the tool once per question rather than trying to
  batch the whole sheet into one prompt, so each answer's confidence and sources
  stay attributable.
- **Right workspace only.** Never answer one company's questionnaire from
  another workspace's knowledge base. If `scope.organizationName` is not the
  company **answering** the questionnaire (the vendor being assessed, not the
  customer who sent it), stop and ask.
- **Availability:** if `scrut_answer_question` isn't present on this connection,
  say so — this skill depends on it and shouldn't fall back to guessing.

## Not in scope for autofill

Other Scrut tools serve different jobs and are **not** part of questionnaire
autofill: browsing the approved answer library (`scrut_search_answer_library`),
locating a named document (`scrut_search_documents`), or getting a download
link for one (`scrut_get_document_content`, which returns a link, not the
document text) are for targeted lookups, not for drafting a questionnaire. Keep
answering to `scrut_answer_question`.

## Changelog

- **1.1.0** — confirm the workspace (`tenant_id`, `scope`) before answering;
  request `response_format: "json"` so references come back whole. Update the
  out-of-scope notes: `scrut_get_document_content` now returns a download link
  only, and `scrut_search_documents` exists for finding documents.
- **1.0.0** — initial release: per-question drafting via `scrut_answer_question`,
  with confidence-based gap flagging, cited sources, and a strict
  no-fabrication policy.
