# Contribution 1: Bug: KeyError: 'status' raised in orderBulkCreate mutation #14506

**Contribution Number:** 1  
**Student:** Shivansh Dutta  
**Issue:** https://github.com/saleor/saleor/issues/14506  
**Status:** Phase I

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

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. Set up saleor locally and obtain an auth token with `MANAGE_ORDERS` permission
2. Call the `orderBulkCreate` mutation without including the `status` field on the order input
3. Observe: the API returns `"Internal Server Error"` instead of a validation error

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

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

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

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

### Code Changes

- **Files to modify:** `saleor/graphql/order/bulk_mutations/order_bulk_create.py`,
  `CHANGELOG.md`
- **Key commits:** [Links to important commits]
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
