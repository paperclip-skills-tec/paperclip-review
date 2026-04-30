# Trigger Examples

Quick reference for how the review tier is determined in common scenarios.

## Full 5-Agent Review

These always trigger full review regardless of other conditions:

| Scenario | Trigger |
|----------|---------|
| Critical bug fix touching 1 file | Priority is `critical` |
| High-priority feature adding 2 files | Priority is `high` |
| Medium task editing `src/auth/middleware.ts` and 1 other file | Security-relevant file detected |
| Medium task touching 4 files, none security-related | File count >= 3 |
| Low task editing `payments/stripe.ts` | Security-relevant file detected (overrides low priority) |

**Note:** Security-relevant file detection overrides priority-based tier for `low` tasks. A low-priority task touching auth code still gets full review.

## Single-Agent Review

| Scenario | Trigger |
|----------|---------|
| Medium task editing 1 component file | Medium priority, < 3 files, no security files |
| Medium task editing 2 utility files | Medium priority, < 3 files, no security files |

## Checklist Only

| Scenario | Trigger |
|----------|---------|
| Low task fixing a typo in a comment | Low priority, no security files |
| Low task updating a README | Low priority, no security files |

## Security-Relevant Path Patterns

Files matching these patterns trigger full review:

```
auth/, authorization/, authz/, permissions/, rbac/, acl/
payments/, billing/, stripe/
secrets/, crypto/, encryption/
middleware/auth*, middleware/session*
data-access/, dal/
.env*, credentials*, tokens/
```

Matching is case-insensitive and checks any segment of the file path.
