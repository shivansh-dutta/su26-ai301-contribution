# Contribution 1: Bug: KeyError: 'status' raised in orderBulkCreate mutation #14506

**Contribution Number:** 1  
**Student:** Shivansh Dutta  
**Issue:** https://github.com/saleor/saleor/issues/14506  
**Status:** Phase II

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
   uv run poe test "saleor/graphql/order/tests/mutations/test_order_bulk_create.py::test_order_bulk_create_without_status_raises_keyerror" -n0
   ```
5. **Expected:** Clean GraphQL validation error saying `status` is required
6. **Actual:** Top-level GraphQL error from unhandled `KeyError: 'status'` in `create_single_order`

### Reproduction Evidence

- **Commit showing reproduction:** https://github.com/shivansh-dutta/saleor/commit/5d5c73263
- **Branch:** https://github.com/shivansh-dutta/saleor/tree/fix-issue-14506
- **My findings:** The `KeyError` fires in `create_single_order` at the line
  `order_data.order.status = order_input["status"]` — a direct dict access with no
  `.get()` fallback. Since `status` is not marked `required=True` in `OrderBulkCreateInput`,
  GraphQL passes the request through without validation, and Python crashes when it
  tries to read the missing key. Confirmed still present on `main` as of June 2026.

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
1. In `order_bulk_create.py`, find `OrderBulkCreateInput` and add `required=True`
   to the `status` field
2. In `CHANGELOG.md`, add under **Breaking changes**:
   `[Preview feature] The OrderBulkCreateInput.status input field in the
   orderBulkCreate mutation is now a required field.`
3. In `CHANGELOG.md`, add under **GraphQL API**:
   `Fix field validation of the orderBulkCreate mutation that requires the
   status field. - #14506 by @<username>`
4. Drop any schema-level test for this (maintainer noted GraphQL schema
   guarantees this automatically — no explicit test needed)

**Implement:** https://github.com/shivansh-dutta/saleor/tree/fix-issue-14506 (Phase III)

**Review:** Will self-review against saleor's `CONTRIBUTING.md` and commit message
conventions before opening PR. Will verify the CHANGELOG entry uses the exact
wording from maintainer's review on PR #14760.

**Evaluate:** Run the mutation without `status` and confirm a clean GraphQL
validation error is returned instead of an Internal Server Error.

---

## Testing Strategy

### Unit Tests

- [ ] Verify calling `orderBulkCreate` without `status` returns a GraphQL
      validation error (not a 500)
- [ ] Verify calling `orderBulkCreate` with a valid `status` still works as
      expected

### Integration Tests

- [ ] End-to-end mutation call without `status` field returns expected error
      response structure

### Manual Testing

[What you tested manually and results]

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

### Code Changes

- **Files to modify (Phase III):** `saleor/graphql/order/bulk_mutations/order_bulk_create.py`,
  `CHANGELOG.md`
- **Key commits:** https://github.com/shivansh-dutta/saleor/commit/5d5c73263 (reproduction test)
- **Approach decisions:** Following maintainer's exact suggestions from PR #14760
  review — using their preferred CHANGELOG wording and skipping the schema
  validation test they said was unnecessary

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Saleor contributing guide](https://github.com/saleor/saleor/blob/main/CONTRIBUTING.md)
- [Previous fix attempt PR #14760](https://github.com/saleor/saleor/pull/14760)
- [Original issue #14506](https://github.com/saleor/saleor/issues/14506)
