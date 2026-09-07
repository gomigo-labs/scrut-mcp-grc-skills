# Test volume: why agents miss tests, and how to fix it

Findings from probing a live **714-test** Scrut org to reproduce the reported
behavior: *agents connected to the
Scrut MCP miss whole areas of the test library and answer from a partial
view.*

The behavior is real, reproducible, and mostly a **server-side API shape
problem**. The skills in this repo work around it; the fixes in
[Server-side fixes](#server-side-fixes) remove it.

## Measured baseline

`scrut_list_tests(limit: 0)` — one call, exact counts over the whole library:

```
total_count: 714
by_status:  Ok 276 | Needs Attention 124 | Ignored 314
```

Note that **`Ignored` (314) is the largest bucket** — 44% of the library. An
agent that treats "not failing" as "healthy" overstates posture by a wide
margin.

## Root cause: ordering + a 200-item cap and no way to filter

`scrut_list_tests` returns **CAT module tests in alphabetical order of
`moduleType`, then legacy cloud/CSPM findings appended last**. Probing offsets
pins the layout:

| Offsets | `moduleType` | Nature | Count |
|---|---|---|---|
| 0–99 | `accessReview` | Per-vendor account reviews | ~100 |
| 100–~469 | `evidence` | Evidence published/updated | ~370 |
| ~470 | `integrations` | SaaS checks (Slack, JumpCloud MFA) | few |
| ~471–491 | `people` | Endpoint, training, acknowledgements | ~20 |
| 492–~59x | `policy` | Policy published/updated | ~100 |
| ~59x–595 | `vendor` | Vendor security assessments | few |
| **596–713** | **`test`** | **Cloud/CSPM (`isExistingCSPM: true`)** | **118** |

Because the CAT block is alphabetical, `integrations` — a small but
security-relevant module — sits buried at offset ~470, behind 370 evidence
rows. It is reachable only by a near-complete sweep or by keyword search.

Combined with `limit` defaulting to **50** and capping at **200**, this means:

- **`scrut_list_tests()`** → 50 rows, *all* `accessReview` vendor account
  reviews. Zero cloud tests.
- **`scrut_list_tests(limit: 200)`** — the maximum — → offsets 0–199, still
  entirely inside `evidence`. **Still zero cloud tests.**
- Reaching a single cloud test requires `offset: 596`, or four sequential
  maximum-size calls.

The 596 boundary is exact, confirmed by single-item probes: offset **595** is
`moduleType: "vendor"` (`Vendor List Maintained`), offset **596** is the first
`moduleType: "test"`, and offset **713** — the last item, `has_more: false` —
is also `moduleType: "test"`. So `moduleType == "test"` is a clean **suffix**
of the ordering, which is what makes the client-side boundary seek in
[`scrut-find-tests`](../skills/scrut-find-tests/) sound: a ~10-probe binary
search with `limit: 1` (~700 tokens) locates the first cloud offset at any
library size, without the tail-seek's assumption that the cloud block is under
200.

The status filter does not rescue it. Of the **124** `needs_attention` tests,
the cloud ones are items **#122, #123, #124**:

```
offset 120 → vendor_company-completes-security-reviews-for-relevant-vendors
offset 121 → ELBv2 deletion protection            (moduleType: test)
offset 122 → IAM user Inactive key rotation       (moduleType: test)
offset 123 → IAM Policy embedded in an IAM Group  (moduleType: test)
```

So `scrut_list_tests(test_status: "needs_attention")` at default limit returns
50 vendor account-review rows and **omits every failing cloud control**. This
is precisely the reported failure, and the previous `scrut-fix-test` skill
(v1.1.0) documented exactly that call as step 1 — its own worked example
("fix our failing S3 test") could not have worked.

### Contributing factor: no filter dimensions

`scrut_list_tests` accepts only `test_status`, `entity_id`, `tenant_id`,
`limit`, `offset`, and two output toggles. There is **no** `query`,
`module_type`, `cloud_provider`, `relevance`, `framework_id`, or `control_id`
filter — even though `moduleType`, `isExistingCSPM` and `isRelevant` are all
returned on every item. The data is there; it just isn't selectable.

## Secondary findings

### 1. Keyword search over tests exists, but is undiscoverable

`scrut_search_documents` **does** match tests — they come back in the
`products` array with `type: "test"`:

```jsonc
// scrut_search_documents(query: "S3 bucket encryption")
"products": [
  { "id": "cloud/tests/f72b9863-…", "name": "S3 bucket encryption", "type": "test" },
  { "id": "cloud/tests/d307efd2-…", "name": "S3 bucket versioning",  "type": "test" }
]
```

This is the single most useful path for "is there a test for X" — one call
instead of four, ~200 tokens instead of ~75K. But the tool's own description
advertises only "trust-vault documents, policies and evidence", and its name
says *documents*, so an agent looking for tests never tries it.

`limit` applies per source, and tests share the `products` bucket with long
policy/evidence content excerpts that can outrank them — so a low `limit` does
suppress tests. But the cliff sits **below** the default, not at it:

| `scrut_search_documents(query: "MFA", …)` | Tests returned |
|---|---|
| `limit: 4` | **0** |
| **default (`limit: 10`)** | **4** (`MFA on root account`, `Slack MFA Enforcement Check`, `MFA on JumpCloud Users`, `IAM assume role lacks external ID and MFA`) |
| `limit: 50` | **4** — the same four |

An earlier draft of this document claimed the default of 10 suppresses tests;
a second verification pass disproved it. The default returned every matching
test for both `"MFA"` and `"S3 bucket encryption"`, and raising `limit` to 50
bought **no** additional recall. The practical risk is therefore smaller than
first reported — but the *cost* risk runs the other way: blanket `limit: 50`
advice adds up to 40 non-test entries whose `name` fields are document-body
excerpts of 150–250 tokens each. Recall still depends on the query resembling
a test *title*: `"MFA"` also surfaced policy prose, while a resource-plus-
property phrasing ranks the test first.

### 2. The returned test id is not the id `scrut_get_test` accepts

Search returns a path-prefixed id in one of two shapes;
`scrut_get_test` accepts only the **stripped** form. The prefixed form fails
**silently**:

```jsonc
scrut_get_test("cloud/tests/d307efd2-…")            → { "data": null }  // no error
scrut_get_test("f72b9863-…")                        → full detail       // works
scrut_get_test("tests/integrations_slack-mfa-enabled") → { "data": null } // no error
scrut_get_test("integrations_slack-mfa-enabled")    → full detail       // works
```

An agent that copies the id verbatim gets an empty object and reasonably
concludes the test does not exist. Silent nulls are worse than errors.

### 2b. `products` entries are not what their field names suggest

Three traps in the one array, all verified live:

- **A third id form.** Alongside `cloud/tests/<uuid>` and `tests/<slug>` there
  is `evidences/<uuid>` — and the **same UUID is returned twice in the same
  response**, once as `type: "test"` and once as `type: "evidence"`
  (`Security validation for change tickets`). An agent that does not
  deduplicate by bare UUID reports one control as two findings.
- **The bucket is mixed.** `products` also carries `type: "risk"` entries
  (`MFA Not Enabled for CRM System and AWS`). Filtering on `type` is
  mandatory, not defensive.
- **`name` is not a name.** For `policy` and `evidence` entries, `name` holds
  a chunk of document *body* — often a full policy section, 150–250 tokens.
  Only `type: "test"` entries have a `name` that is a title. This is both a
  rendering hazard (an agent echoing `products[].name` prints a wall of prose)
  and the reason a high `limit` is expensive.

### 2c. Two sources are unreliable, and one count is wrong

- **`vault` fails on every search.** Every probe returned
  `source_errors: [{source: "vault", upstreamStatus: 500, upstreamMessage:
  "Forbidden", service: "kaiService"}]`, so `vaultDocuments` is always empty on
  this org. Trust-vault documents are currently unsearchable, and an agent
  reading only the empty bucket concludes they do not exist.
- **Search `total_count` does not match the query.** `"MFA"` and
  `"S3 bucket encryption"` — unrelated queries — both returned
  `total_count: 13`, while `products` held 10 entries. It should not be quoted
  as a match count.

### 3. `include_details: true` is a context bomb

Full upstream objects carry HTML-laden `description` and `remediation` bodies
— measured at roughly **2,000 tokens per test**. Across 714 tests that is
**~1.4M tokens**, far beyond any context window. Even the lean form costs
~100 tokens/item, so a complete sweep is ~75K tokens in 4 calls.

### 4. A single `scrut_get_test` can exceed the response limit

`scrut_get_test` has no field-selection or pagination parameters, so one test
can blow the entire call. `integrations_slack-mfa-enabled` returned
**128,387 characters** and was rejected outright:

| Field | Size | Share |
|---|---|---|
| `testHistory` (485 entries) | 94,593 chars | 74% |
| `flaggedResources` (100 of 245, already truncated) | 40,443 chars | 25% |
| everything else (title, status, controls, remediation, description) | ~1,600 chars | 1% |

The run history — almost never the reason anyone opened the test — is
three-quarters of the payload, and `testHistory` appears unbounded. The
agent-visible symptom is a hard failure on a perfectly ordinary request, with
no parameter available to make it succeed.

Note also that `flaggedResources` was **already** capped at 100 of 245, with
`flaggedResourcesTruncated: true` — so even the successful path can quietly
under-report affected resources.

### 5. Entity scoping can hide tests

CAT tests default to **Organization Wide**, not to all entities. On this org
there are three entities. A test scoped to a specific product entity rather
than Organization Wide will not appear in the default listing, which reads as
"the test doesn't exist."

## Server-side fixes

In rough priority order. The first two remove most of the problem.

1. **Add a `query` parameter to `scrut_list_tests`** — substring match on
   title and `targetEntity`. This is the highest-value change: it turns "find
   the encryption test" from a 4-call, 75K-token sweep into one small call.
   The client-side name filter already used by `scrut_search_documents` for
   policies/evidence is a sufficient implementation.

2. **Add a `module_type` filter** (`accessReview`, `evidence`, `integrations`,
   `people`, `policy`, `vendor`, `test`) plus a convenience `cloud_only` /
   `is_existing_cspm` toggle. Agents overwhelmingly want the cloud and
   integration tests; today those are the two hardest to reach. Also add
   `relevance` so `isRelevant: 0` tests can be excluded.

3. **Extend the `limit: 0` summary with a `by_module` facet.** `by_status` and
   `by_assignee` already come free; `by_module` would let an agent see in one
   cheap call that 118 cloud tests exist and target them directly — and would
   make the ordering bias self-evident rather than invisible.

4. **Fix the id round-trip.** Either have `scrut_search_documents` return the
   bare `testId` for `type: "test"` entries, or have `scrut_get_test` accept
   and normalize both prefixed forms (`cloud/tests/<uuid>` and
   `tests/<slug>`). Failing that, return an error rather than
   `{"data": null}` so the agent can self-correct — a silent null is
   indistinguishable from a missing test.

5. **Document test search in `scrut_search_documents`'s description.** One
   sentence — "also matches continuous tests, returned in `products` with
   `type: "test"`" — makes an existing capability discoverable. Cheapest fix
   on this list.

6. **State the ordering in `scrut_list_tests`'s description**, and warn that
   cloud/CSPM tests sort last so a single page is not a representative sample.
   Tool descriptions are the only guidance an agent gets by default.

7. **Reconsider the 200 cap, or strip HTML from `include_details`.** The cap
   is defensible given item size; the HTML bodies are not. Plain-text or
   truncated `description`/`remediation` would cut detail cost several-fold.

8. **Cap or paginate `testHistory` in `scrut_get_test`**, or add an
   `include_history` / `include_resources` toggle. An unbounded 485-entry
   history that makes the whole call fail is the one defect here that leaves
   an agent with *no* working path to the data.

9. **Give tests their own result bucket in `scrut_search_documents`, or add a
   `type` filter.** The default `limit` of 10 does return tests, so this is a
   cost and clarity fix rather than a recall fix: today the only way to be sure
   of test recall is to raise `limit` and pay for up to 40 document-body
   excerpts. A `type: "test"` filter removes that trade-off outright. Also
   return the bare `testId` (see fix 4), deduplicate the `evidences/<uuid>`
   entry that is emitted twice under two `type`s, and rename or trim the `name`
   field for content-chunk entries — it currently holds document body, not a
   name.

10. **Surface `Ignored` prominently in summaries.** At 44% of this library it
    materially changes posture reporting.

11. **Fix the `vault` search source.** It returns `500 Forbidden`
    (`kaiService`) on every query for this org, so trust-vault documents are
    silently unsearchable. Also make search's `total_count` query-specific —
    it returned the same value for unrelated queries.

## Client-side mitigations (shipped in this repo)

Available now, no server change required:

- **[`scrut-find-tests`](../skills/scrut-find-tests/)** — a discovery skill
  that routes by question shape: keyword search via `scrut_search_documents`
  for "is there a test for X", `limit: 0` for counts, and a verified
  page-to-completion sweep grouped by `moduleType` when completeness matters.
  Documents the id trap and the `include_details` cost.
- **[`scrut-fix-test`](../skills/scrut-fix-test/) v1.2.0** — discovery step
  rewritten to search-first; no longer picks a test from a biased first page.

These reduce the failure but cannot eliminate it: without a server-side
`query` or `module_type` filter, any request that genuinely needs a complete
view still costs a multi-call sweep, and any agent operating without these
skills loaded still reads page one and answers from it.
