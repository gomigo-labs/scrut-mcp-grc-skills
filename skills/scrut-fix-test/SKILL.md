---
name: scrut-fix-test
description: >-
  Remediate a failing Scrut continuous cloud test (CAT) from the user's IDE:
  find the failing test — including tests that are failing but currently ignored
  — establish which resources it matches, work out what the fix would break
  before proposing it, help edit the infrastructure, and verify that the
  exposure is actually gone. Use when a cloud/config compliance test is failing
  and the user wants to fix it where their code lives. Triggers include "fix
  this failing test", "remediate our public S3 buckets", "what's failing in our
  cloud tests", "review our ignored tests", and "our encryption test is failing,
  help me fix it".
version: 1.3.0
---

# Scrut: fix a failing cloud test

Find a failing continuous automated test (CAT), understand what it matches, work
out what a fix would break, change the infrastructure, and verify the exposure is
gone — without leaving the editor.

**The goal is removing the exposure, not turning the test green.** A test can pass
while the underlying risk is untouched. Where those two come apart, say so.

## What the terms mean

- **CAT test** — a Scrut continuous automated test that checks a cloud/config
  requirement (e.g. "S3 buckets must not grant `*` actions to `*` principals").
- **`needs_attention`** — the status of a failing test.
- **`ignored`** — **also a failing test**, with a human justification attached.
  It is hidden from status filters, not passing. Ignored tests are in scope for
  this skill.
- **Flagged resource** — a specific cloud resource that caused the test to fail.
  May be absent or truncated in the payload; see step 2.
- **Blast radius** — everything *other* than the flagged resource that the same
  change would affect.

## Tools this skill uses

- `scrut_search_documents` — keyword-find a test by name (returns tests in its
  `products` array, not just documents).
- `scrut_list_tests` — enumerate failing tests. Run it for **both**
  `test_status: "needs_attention"` and `test_status: "ignored"`.
- `scrut_get_test` — full detail for one test (`response_format: "json"` for
  mapped controls, flagged resources, and `ignoredReason`).
- `scrut_answer_question` — remediation guidance from the knowledge base.
- `scrut_get_document_content` — a related policy/evidence, if useful.

Plus whatever read-only cloud access the user's environment provides (CLI,
shell, or cloud MCP). Much of this skill depends on being able to query the
cloud directly. If you cannot, keep going and mark the affected conclusions
undetermined — do not guess.

## Procedure

### 1. Find the finding

Test libraries run to hundreds of tests, and `scrut_list_tests` returns
**cloud/CSPM tests last** — so a default page of 50 typically contains *no cloud
tests at all*. Pick by situation:

- **User named a test or a service** ("our S3 public-access test") →
  `scrut_search_documents(query: "<name or service>", response_format: "json")`,
  take `products` entries with `type: "test"` (that bucket also holds `policy`,
  `evidence` and `risk` entries — filter on `type`), and **strip the path
  prefix** from the `id` to get the `test_id`. Three prefixes occur:
  `cloud/tests/<uuid>`, `tests/<slug>` and `evidences/<uuid>`. Passing an
  unstripped id to `scrut_get_test` returns `{"data": null}` with no error. The
  default `limit` is normally enough; if no `type: "test"` entry comes back,
  retry once at `limit: 50`.
- **User asked what's failing** → `scrut_list_tests(limit: 0)` for exact counts,
  then page with `limit: 200`, looping `offset = next_offset` until `has_more`
  is `false`. Group by `moduleType` and confirm your item count matches
  `total_count` before presenting. Cloud tests are `moduleType: "test"` /
  `isExistingCSPM: true`.

**Query both statuses, not just `needs_attention`.** An ignored test is a failing
test with a note attached — hidden from the default status filter, not passing.
If the user named a test, match it; otherwise present both lists and confirm
which to work on, making clear that the ignored ones are also failing.

Never present a failing-test list from a single default-limit call — on a
measured 714-test org the cloud tests were items #122–124 of 124 failing tests.
For the full strategy set, see the **`scrut-find-tests`** skill.

### 2. Establish the resource set

Call `scrut_get_test` with the `test_id` and `response_format: "json"`. In the
default markdown format the flagged resources render as a **count, not a list** —
you need the JSON to see them at all.

- If `flaggedResources` is populated, use it.
- **If it is empty, say so plainly**, then reconstruct the set by querying the
  cloud for everything the rule would match. Label every downstream statement as
  *reconstructed* rather than *reported by Scrut*. Never proceed silently as if
  you had a resource list.

Two payload fields change what you are allowed to conclude:

- `flaggedResourcesTruncated: true` (or `flaggedResourceCount` <
  `totalFlaggedResourceCount`) means you are **not** seeing every affected
  resource, and there is no offset parameter to reach the rest. Treat the
  remainder exactly like an empty list — reconstruct it from the cloud — and
  tell the user the reported list was truncated.
- `detailSource: "list_fallback"` means detail was reconstructed from the list
  API; `remediation` may be `null` and `testHistory` empty. That is not "no
  guidance exists" — go to step 3.

If the test is ignored, read `ignoredReason` and check one thing before anything
else: **does the justification answer the question this test actually asks?**
A reason like "we need this bucket to be public" does not address a rule about
granting `*` actions to `*` principals — those can both be true at once. When the
reason and the rule are about different things, the exception may be unnecessary
and the fix may cost the user nothing. Raise that first; it is often the whole
answer.

### 3. Gather remediation guidance

Call `scrut_answer_question` (e.g. *"How do we remediate <test name>?"*).
Optionally `scrut_get_document_content` for a policy that clarifies the required
end state.

Treat the guidance as a starting point, not a change plan. It is written for the
rule in general, not for this estate.

### 4. Classify the fix

| Class | Examples | Handling |
|---|---|---|
| **Additive / reversible** | enable versioning, logging, encryption-at-rest defaults, backup retention | No blast-radius work needed. Propose it. These often *create the telemetry* later steps need — prefer them first. |
| **Removes or narrows an access path** | tighten a bucket policy, narrow a security group, drop an IP allowlist entry, make a database private | **Step 5 is mandatory.** |
| **Account- or organization-wide control** | account-level S3 Block Public Access, SCPs, regional encryption defaults, default network ACLs | **Never bundle with a resource-level fix.** Always its own change, always a full inventory of everything the setting governs. |
| **Needs a code change first** | build tooling that sets public ACLs, hardcoded endpoints, a client that only speaks plaintext | Route to the owning team as a PR. Do not hand it to whoever runs infrastructure as a cloud change. |

A single remediation recommendation that mixes a resource-level fix with an
account-wide control is the most common way a correct-looking fix causes an
outage. Split it.

### 5. Impact check — do not produce a diff until these are answered

For anything that removes or narrows access:

**a. What else does this rule match?**
Enumerate every other resource in the account the same change would hit. An
account- or organization-wide setting affects everything it governs, not just the
flagged resource.

**b. For each consumer: reads or writes, authenticated or anonymous?**
This is usually the deciding fact, and cloud configuration alone will not tell
you. Search the codebase and IaC for the resource identifier — bucket name, ARN,
security group id, queue URL, hostname, endpoint. Removing anonymous *write* and
removing anonymous *read* look almost identical in a policy diff and have
completely different consequences.

**c. Is access granted by more than one mechanism?**
Resource policy **and** object/resource ACLs. Security group **and** network ACL.
Identity policy **and** resource policy. Fixing one layer while another still
grants access produces a passing test and an unchanged exposure. Check every
layer that applies before claiming the access path is closed.

**d. Is there an access log?**
If the resource has no access logging, you cannot tell who is currently using it
— and once the change is applied, you never will be able to answer that
retrospectively. Propose enabling logging as a **separate, earlier** change. It
is read-safe and takes minutes.

**e. What could you not determine?**
Say it explicitly. An honest gap is more useful than a confident guess, and the
person applying the change needs to know where to look themselves.

> **Absence of a reference is weak evidence.** A code search that finds no
> consumer is not proof there is none — it covers the branches and repositories
> it covers. Prefer a real grep over a search index, and when a conclusion rests
> on an absence, say that is what it rests on.

### 6. Propose the change

- **Capture rollback first.** Record the current configuration verbatim before
  proposing a replacement. Much of what these tests flag is not versioned.
- **If it is in IaC:** locate the code and show the diff. **Confirm with the user
  before applying** — this edits real infrastructure.
- **If it is not in IaC:** say so, and give exact CLI or console steps instead.
  Plenty of flagged resources were created outside code; do not invent Terraform
  that does not exist, and note that the resource is unmanaged.
- **Prefer a pattern the estate already uses.** If another resource in the same
  account already satisfies this rule, copy its configuration rather than
  designing a new one. It is proven in their environment and it is an easier
  review.
- **Sequence explicitly** when steps have ordering dependencies, and say what
  each step does and does not accomplish on its own.

### 7. Verify the effect, then the test

Two separate checks, in this order.

1. **Did the exposure close?** Give a direct test that does not involve Scrut —
   an IAM policy simulation, an unauthenticated request that should now fail, a
   reachability analysis, a scratch resource carrying the proposed configuration.
   Include the check that would catch a partial fix from 5(c).
2. **Did the test pass?** Tell the user to re-run the test in Scrut
   (Test Details → Test Now). This skill does not trigger a scan from the MCP.

   **Optional (user-driven):** if they have already re-run it, they may ask you to
   call `scrut_get_test` again. If status is still `needs_attention`, results may
   not have updated yet — check again later in Scrut; do not claim the fix failed
   immediately.

A green test with check 1 unaddressed is not a completed remediation. Say so.

### 8. Report what you could not determine

Close with the 5(e) list, plus the rollback captured in step 6. If a reader
cannot check your reasoning, they will either apply it blind or shelve it.

## Why verification is in Scrut

MCP-triggered test re-runs are not available yet — test runs can be slow today,
and we're improving that before exposing re-run through the MCP. For now, re-run
the test in Scrut to verify. Triggering a re-scan from your agent will return in
a future update. Relay this briefly if the user asks why there is no automatic
re-run from the agent.

## Worked example

User: *"Our S3 public-access test is failing — fix it."*

**1.** `scrut_search_documents(query: "S3 bucket public access",
response_format: "json")` → `products` contains
`{"id": "cloud/tests/<uuid>", "name": "S3 bucket world star policy",
"type": "test"}`. Strip the prefix → `test_id: "<uuid>"`. Checking both statuses
shows the test is **ignored**, not merely failing — so it has been failing
silently with a justification attached.

**2.** `scrut_get_test` returns `flaggedResources: []`, so tell the user the
resource list is empty and reconstruct it from the cloud: three buckets whose
policy grants `s3:*` to `Principal: "*"`.

The `ignoredReason` reads *"we need these buckets public for our business use
case."* That does not answer this rule, which is about granting **all actions** to
**all principals** — not about the bucket being readable. Flag it: the exception
may be unnecessary, and the business need may survive the fix untouched.

**3–4.** Guidance says "reduce to `s3:GetObject` and enable Block Public Access."
That is two fixes in different classes. The policy change is *access-narrowing*;
account-level Block Public Access is an *account-wide control*. Split them.

**5.** The impact check:

- *(a)* Sweep every bucket in the account for a public policy — not just the
  three. Account-level Block Public Access revokes public policies on **all** of
  them, and typically some are public deliberately: software downloads, static
  assets, documents linked from a product.
- *(b)* Search the codebase for the bucket names. Writers are CI pipelines using
  IAM roles, so removing anonymous **write** breaks nothing. Readers include an
  application fetching assets and an update feed polled by installed clients —
  all unauthenticated. So removing anonymous **read** breaks live functionality.
  The safe fix is narrowing the actions, not removing public access.
- *(c)* Check object ACLs, not just the policy. If the publishing tool uploads
  with a public-read ACL, objects stay world-readable after the policy is fixed —
  the test passes and the exposure remains.
- *(d)* No access logging on any of the three. Propose enabling it first.
- *(e)* Unknown: actual read volume, and whether anonymous writes already
  occurred. Both are unanswerable without the logs from *(d)*.

**6.** Another bucket in the same account already grants `s3:GetObject` with an
`aws:SecureTransport` condition and nothing else. Copy that policy. Capture the
existing policies for rollback. Sequence: policy narrowing now; the publishing
tool's ACL setting is a code change for the owning team; the ACL blocks come
after that ships; account-level Block Public Access is deferred and scoped
separately.

**7.** Verify: unauthenticated `GET` still returns 200, unauthenticated `PUT` now
returns 403, object ACLs re-checked. Then re-run the test in Scrut.

**8.** Report the two unknowns from *(e)* and the captured rollback.

Note what happened: the guidance as written would have revoked public access
across every public bucket in the account, and the obvious partial fix would have
turned the test green while leaving the objects readable. Both were caught by
step 5, before a diff existed.

## Edge cases and honesty

- **Never pick the test from a first page.** `scrut_list_tests` orders
  cloud/CSPM tests last, so a default-limit call is biased toward access-review
  and evidence rows. Search by keyword, or page to completion.
- **A `{"data": null}` from `scrut_get_test` means a malformed id**, not a
  missing test — strip any `cloud/tests/`, `tests/` or `evidences/` prefix and
  retry before telling the user the test does not exist.
- **Confirm before editing infra code.** Show the diff; do not silently rewrite
  IaC.
- **Never bundle an account-wide control with a resource-level fix.** If guidance
  arrives bundled, split it and say why.
- **Do not claim "fixed" from the agent alone.** Verification requires the
  effect check plus a re-run in Scrut.
- **Do not claim "no longer exposed" when only one layer was changed.** Name the
  layers still outstanding.
- **Empty or truncated `flaggedResources` is a fact to report, not a gap to
  paper over.**
- **An ignored test is a failing test.** Never describe one as passing or
  resolved.
- **No cloud read access?** Do the parts you can, and mark the impact check
  undetermined rather than skipping it silently.
- **Report tool errors** plainly instead of looping.

## Changelog

- **1.3.0** — search `ignored` tests as well as `needs_attention`, and check
  whether an ignore's justification actually answers its test; handle empty
  *and* truncated `flaggedResources` explicitly; add fix classification and a
  mandatory impact check before any diff; never bundle account-wide controls
  with resource-level fixes; verify the exposure separately from the test
  result; handle resources not managed in IaC; capture rollback; report what
  could not be determined.
- **1.2.1** — id-stripping covers all three prefixes (`cloud/tests/`, `tests/`,
  `evidences/`), note that `products` is a mixed bucket, and drop the
  always-`limit: 50` advice — the default suffices and 50 costs more for the
  same recall.
- **1.2.0** — fix test discovery: search-first by keyword (`scrut_search_documents`
  returns tests), document the `cloud/tests/` id trap and the module-ordering
  bias that hid cloud tests behind hundreds of CAT tests, require paging to
  completion for failing-test lists, and handle `flaggedResourcesTruncated` /
  `detailSource: "list_fallback"`. Defers to the new `scrut-find-tests` skill.
- **1.1.0** — remove `scrut_run_test` dependency; verification is user-driven
  in Scrut (optional `scrut_get_test` poll after user re-runs). MCP re-runs
  deferred pending performance improvements.
- **1.0.0** — initial release.
