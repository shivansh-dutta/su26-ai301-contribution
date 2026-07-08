# Contribution 2: Bug: Internal Server Error instead of a validation error on too long inputs #12696

**Contribution Number:** 2
**Student:** Shivansh Dutta
**Issue:** https://github.com/saleor/saleor/issues/12696
**Status:** Phase I Complete

---

## Why I Chose This Issue

I chose Saleor issue #12696 — "Internal Server Error instead of a validation error
on too long inputs" — because it is a well-scoped, `good first issue`-labeled bug in
a project I already know deeply from my first contribution (#14506). The `message` and
`pspReference` fields on transaction-event mutations map to database columns capped at
512 characters, but the GraphQL layer never validates that length, so an over-long
value hits PostgreSQL, raises a `DataError`, and is returned to the caller as an
Internal Server Error (500) instead of a descriptive validation error.

I'm interested in this because:
1. It is the **same bug family** as my first contribution — an unhandled error that
   should surface as a clean GraphQL validation error — so I can reuse the fix pattern
   (push validation to the API/input layer before the write reaches the database).
2. My Saleor dev environment (devcontainer, `uv`/`poe`, full test suite) is already
   built and green, so I can reproduce and verify quickly.
3. The codebase area is contained: the fields are defined in
   `saleor/payment/models.py` (`TransactionEvent.message` / `.psp_reference`,
   `max_length=512`) and the entry points are the transaction mutations
   (`transactionEventReport`, `transactionCreate`, `transactionUpdate`).
4. Maintainers are **actively working in this exact area** — recent 2026 PRs such as
   "Reject control characters from the request" and "Gracefully handle invalid json
   string" harden the same class of 500-to-validation-error problem — so a fix here
   is likely to get engaged review.

From reading the issue, "fixed" means: sending a `message` or `pspReference` longer
than 512 characters to a transaction mutation returns a GraphQL validation error
identifying the offending field and its max length, instead of a 500.

---

## Understanding the Issue

*(Phase II — to be completed: reproduce locally, confirm the exact crash site, and
draft the solution approach.)*

### Problem Description

*(Phase II)*

### Expected Behavior

*(Phase II)*

### Current Behavior

*(Phase II)*

### Affected Components

- `saleor/payment/models.py` — `TransactionEvent.message` and
  `TransactionEvent.psp_reference` (`CharField(max_length=512)`)
- The transaction GraphQL mutations that write these fields
  (`transactionEventReport`, `transactionCreate`, `transactionUpdate`)

---

## Reproduction Process

*(Phase II)*

---

## Solution Approach

*(Phase II)*

---

## Testing Strategy

*(Phase III)*

---

## Implementation Notes

### Week 5 Progress (Phase I)

Selected Saleor issue #12696 from the CodePath candidate list after running it through
the 6-point selection checklist (scored 6/6): the problem is understood in one
sentence, scope is small, it matches skills proven on my first contribution, it is
open/unassigned with no competing PR, there is recent maintainer activity in the same
area, and the project's dev environment is already set up. Forked the project,
commented on the issue expressing interest, and marked the row on the course sheet.

---

## Pull Request

*(Phase IV)*

**PR Link:**
**PR Description:**
**Maintainer Feedback:**
**Status:** Not yet submitted

---

## Learnings & Reflections

*(To be completed as the contribution progresses.)*

---

## Resources Used

- [Original issue #12696](https://github.com/saleor/saleor/issues/12696)
- [Saleor contributing guide](https://github.com/saleor/saleor/blob/main/CONTRIBUTING.md)
- [saleor/payment/models.py](https://github.com/saleor/saleor/blob/main/saleor/payment/models.py)
# Contribution 1: Bug: KeyError: 'status' raised in orderBulkCreate mutation #14506

**Contribution Number:** 1  
**Student:** Shivansh Dutta  
**Issue:** https://github.com/saleor/saleor/issues/14506  
**Status:** Phase IV

---

## Why I Chose This Issue

The `orderBulkCreate` mutation crashes with a KeyError when the `status`
field is omitted from the input, instead of returning a validation error.
The fix is well-scoped: mark `status` as `required=True` in the GraphQL
input definition and add a changelog entry. A maintainer reviewed a
previous attempt and left specific feedback, giving me a clear roadmap
to follow.

---

## Understanding the Issue

### Problem Description

The `orderBulkCreate` GraphQL mutation accepts an `orders` input array. Each
order in that array has an `OrderBulkCreateInput` type. The `status` field on
that input is currently optional in the schema, but the underlying Python code
assumes it will always be present. When a caller omits `status`, Python raises
an unhandled `KeyError: 'status'` which bubbles up as an Internal Server Error
instead of a clean validation error.

### Expected Behavior

When `status` is omitted from an order in `orderBulkCreate`, the API should
return a descriptive GraphQL validation error telling the caller that `status`
is a required field — not crash with an Internal Server Error.

### Current Behavior

Calling `orderBulkCreate` without the `status` field on an order causes a
`KeyError: 'status'` exception in the Python layer, resulting in an unexpected
server error response with no useful message for the API consumer.

### Affected Components

- `saleor/graphql/order/bulk_mutations/order_bulk_create.py` — where the
  `OrderBulkCreateInput` type and `status` field are defined
- `saleor/graphql/schema.graphql` — generated schema snapshot (regenerated via `poe build-schema`)
- `CHANGELOG.md` — needs a breaking change entry since making a field required
  is a breaking API change (the field is in a Preview feature, so this is
  acceptable per maintainer guidance)

---

## Reproduction Process

### Environment Setup

Cloned fork into `Documents/CodePath/saleor` on Windows 11. Used saleor's built-in
`.devcontainer/` setup with Cursor (VS Code fork) + Docker Desktop — the container
builds Python 3.12 + uv + all dependencies and runs `python manage.py migrate`
automatically on first start.

Challenges faced:
- Docker Desktop was not running initially — had to launch it before "Reopen in Container" worked
- Git flagged "dubious ownership" inside the container — fixed with `git config --global --add safe.directory /app`
- Cursor's AI composer corrupted line endings across 4,195 files when attempting a tab fix — resolved with `git restore .`

### Steps to Reproduce

1. Clone fork: `git clone https://github.com/shivansh-dutta/saleor.git`
2. Open in Cursor → Command Palette → "Dev Containers: Reopen in Container"
3. Wait for container build and migrations to complete
4. Run the reproduction test:
   ```
   uv run poe test saleor/graphql/order/tests/mutations/test_order_bulk_create.py -k without_status -n0
   ```
5. **Expected:** Clean GraphQL validation error saying `status` is required
6. **Actual (before fix):** Top-level GraphQL error from unhandled `KeyError: 'status'` in `create_single_order`

### Reproduction Evidence

- **Commit showing reproduction:** https://github.com/shivansh-dutta/saleor/commit/5d5c73263
- **Branch:** https://github.com/shivansh-dutta/saleor/tree/fix-issue-14506
- **My findings:** The `KeyError` fires in `create_single_order` at the line
  `order_data.order.status = order_input["status"]` — a direct dict access with no
  `.get()` fallback. Since `status` was not marked `required=True` in `OrderBulkCreateInput`,
  GraphQL passed the request through without validation, and Python crashed when it
  tried to read the missing key. Confirmed still present on `main` as of June 2026.

---

## Solution Approach

### Analysis

The `status` field is defined as optional in `OrderBulkCreateInput` but the
Python code that processes the mutation treats it as always present, causing a
`KeyError` when it is missing. The fix is to make the field required at the
schema level so GraphQL rejects the request before it ever reaches the Python
handler.

### Proposed Solution

Add `required=True` to the `status` field definition in `OrderBulkCreateInput`.
This pushes validation to the GraphQL layer, which will automatically return a
descriptive error to the caller. A changelog entry is also needed to flag this
as a breaking change for the Preview feature.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The `status` field in `OrderBulkCreateInput` must be required.
Currently it is optional, causing a `KeyError` crash when omitted.

**Match:** Other required fields in Saleor's bulk mutations use `required=True`
in the Graphene field definition. The previous PR #14760 shows the exact
one-line change needed and the maintainer's preferred CHANGELOG wording.

**Plan:**
1. In `order_bulk_create.py`, add `required=True` to the `status` field
2. Regenerate `schema.graphql` via `uv run poe build-schema`
3. In `CHANGELOG.md`, add under **Breaking changes** and **GraphQL API**
4. Write a test asserting the fix returns a clean validation error

**Implement:** https://github.com/shivansh-dutta/saleor/tree/fix-issue-14506 ✅ Done

**Review:** Self-reviewed against saleor's `CONTRIBUTING.md` and Conventional
Commits format. CHANGELOG uses the exact wording from maintainer's review on
PR #14760. Full test suite (17,323 tests) passes.

**Evaluate:** Test `test_order_bulk_create_without_status_returns_validation_error`
confirms a clean GraphQL validation error is returned instead of an Internal
Server Error. Full suite green — no regressions.

---

## Testing Strategy

### Unit Tests

- [x] Verify calling `orderBulkCreate` without `status` returns a GraphQL
      validation error (not a 500) —
      `test_order_bulk_create_without_status_returns_validation_error`
- [x] Verify calling `orderBulkCreate` with a valid `status` still works as
      expected — all 86 existing `test_order_bulk_create.py` tests still pass

### Integration Tests

- [x] Full saleor test suite (17,323 tests, 1 skipped) passes with no
      regressions — run time 13 minutes with 20 parallel workers

### Manual Testing

Ran the full test suite inside the Dev Container. All existing
`orderBulkCreate` tests pass `status` in their fixture input, confirming
making the field required is backwards-compatible with correct callers.

---

## Implementation Notes

### Week 1 Progress

Selected issue, reviewed previous PR #14760 and maintainer feedback. Identified
exact files to change and maintainer's preferred CHANGELOG wording.

### Week 2 Progress

Set up local dev environment via Dev Container. Confirmed bug still exists on
`main`. Pinpointed the exact crash location: `create_single_order` in
`order_bulk_create.py` at `order_data.order.status = order_input["status"]`.
Wrote and committed a reproduction test that confirms the bug on-demand.

### Week 3 Progress

Implemented the fix: added `required=True` to the `status` field in
`OrderBulkCreateInput`. Regenerated the GraphQL schema snapshot via
`uv run poe build-schema` — confirmed only `OrderBulkCreateInput.status`
changed from `OrderStatus` to `OrderStatus!`. Updated the reproduction test
into a proper fix-verification test. Added both CHANGELOG entries using the
maintainer's exact wording from PR #14760. Ran the full 17,323-test suite —
all pass.

### Week 4 Progress

Rebased `fix-issue-14506` onto the latest `saleor/saleor` `main` (41 commits
behind, zero conflicts). Per maintainer @maarcingebala's stated preference on
the earlier attempt (PR #14760) — "we don't test such changes, as it is
guaranteed by the GraphQL schema" — removed the dedicated validation test and
repackaged the change into two clean commits: the fix + regenerated schema,
and the changelog. Confirmed the final diff against upstream is exactly 3
files. Re-ran the `order_bulk_create` test suite (86 tests) with no
regressions. Opened
[PR #19391](https://github.com/saleor/saleor/pull/19391) against
`saleor/saleor:main`, filled out the project's PR template, and updated the
CHANGELOG entry to reference the real PR number. Left a review-request
comment tagging @maarcingebala.

### Code Changes

- **Fix:** `saleor/graphql/order/bulk_mutations/order_bulk_create.py`
  — added `required=True` to `status` field (one line)
  — commit: https://github.com/shivansh-dutta/saleor/commit/5f485ba6d
- **Schema:** `saleor/graphql/schema.graphql`
  — regenerated; `OrderBulkCreateInput.status` now `OrderStatus!`
  — commit: https://github.com/shivansh-dutta/saleor/commit/5f485ba6d
- **Test:** `saleor/graphql/order/tests/mutations/test_order_bulk_create.py`
  — renamed and tightened reproduction test to verify the fix
  — commit: https://github.com/shivansh-dutta/saleor/commit/38d3ca147
- **Changelog:** `CHANGELOG.md`
  — added Breaking changes + GraphQL API entries
  — commit: https://github.com/shivansh-dutta/saleor/commit/19f2767a3
- **Approach decisions:** Followed maintainer's exact suggestions from PR #14760
  review — used their preferred CHANGELOG wording; no separate schema-validation
  test since `required=True` is enforced by the GraphQL layer automatically
- **Rebase (Phase IV):** rebased onto `saleor/saleor:main`, dropped the test per
  maintainer guidance, repackaged into 2 commits
  — https://github.com/shivansh-dutta/saleor/commit/dcec62680 (fix + schema)
  — https://github.com/shivansh-dutta/saleor/commit/d2aa4c49f (changelog)
  — https://github.com/shivansh-dutta/saleor/commit/552c4c175 (changelog PR-number update)

---

## Pull Request

**PR Link:** https://github.com/saleor/saleor/pull/19391

**PR Description:** Marks the `status` field on `OrderBulkCreateInput` as
`required=True` so `orderBulkCreate` returns a clean GraphQL validation error
instead of crashing with an unhandled `KeyError: 'status'` when the field is
omitted. Includes the regenerated schema snapshot and CHANGELOG entries;
follows maintainer feedback from the earlier attempt (#14760) on wording and
on omitting a dedicated test.

**Maintainer Feedback:**
- 2026-07-01: PR opened against `saleor/saleor:main`; left a review-request
  comment tagging @maarcingebala, referencing the prior PR #14760 and the
  guidance followed.

**Status:** Awaiting review

---

## Learnings & Reflections

### Technical Skills Gained

Learned how Graphene enforces GraphQL-level field validation via
`required=True`, why the committed `schema.graphql` snapshot must be
regenerated (never hand-edited) and is CI-enforced, and how to read prior
maintainer feedback on a closed PR to shape a mergeable second attempt. Also
practiced a full rebase-and-repackage workflow: rebasing a feature branch
onto a fast-moving upstream (41 commits), then using `git reset --soft` to
cleanly re-split commits and drop one entirely.

### Challenges Overcome

The Dev Container had no cached GitHub credentials, so pushes had to happen
from the host checkout (same bind mount, host has Credential Manager) while
all git/test work ran inside the container via `docker exec`. Also hit a
`git reset --soft` gotcha — it re-stages everything from the prior HEAD, so
`git add <file>` no-ops and a follow-up commit silently sweeps in unrelated
changes unless you explicitly `git restore --staged` the files you don't
want yet.

### What I'd Do Differently Next Time

I'd read the previous maintainer's review comments before starting Phase II,
not just before Phase IV — it would've told me from the start that a
dedicated test wasn't wanted, saving the extra Phase III test-writing work I
ended up reverting during the rebase.

---

## Resources Used

- [Saleor contributing guide](https://github.com/saleor/saleor/blob/main/CONTRIBUTING.md)
- [Previous fix attempt PR #14760](https://github.com/saleor/saleor/pull/14760)
- [Original issue #14506](https://github.com/saleor/saleor/issues/14506)
