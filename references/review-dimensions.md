# Review Dimensions Reference

Each review agent focuses on one dimension. This document defines the scope, what to look for, and output expectations for each.

## 1. Correctness

**Goal:** Verify the implementation matches the specification and acceptance criteria.

**What to check:**
- Each acceptance criterion in the issue/spec is satisfied by the code
- Logic errors, off-by-one errors, null/undefined handling
- Edge cases: empty inputs, boundary values, concurrent access
- Incorrect assumptions about data shape or API contracts
- Return values and error propagation match caller expectations

**Severity guide:**
- **Critical:** Acceptance criterion not met; logic error that would cause incorrect behaviour in production
- **Warning:** Edge case not handled; assumption that may break under specific conditions
- **Suggestion:** Defensive coding improvement; clearer assertion

## 2. Quality

**Goal:** Evaluate code cleanliness, readability, and maintainability.

**What to check:**
- Naming: variables, functions, classes are clear and consistent with project conventions
- Structure: appropriate abstraction level, no god functions, single responsibility
- SOLID principles where applicable
- Duplication: copy-paste code that should be extracted
- Readability: could a new team member understand this without extra context?
- Consistency with surrounding codebase style

**Severity guide:**
- **Critical:** Code is unmaintainable or fundamentally misstructured
- **Warning:** Naming confusion, moderate duplication, unclear intent
- **Suggestion:** Minor style inconsistency, slightly verbose code

## 3. Architecture

**Goal:** Ensure the change fits within the existing system architecture.

**What to check:**
- Layer violations (e.g., UI directly querying database, controller containing business logic)
- Unexpected dependencies (importing from wrong layer or module)
- Pattern deviations from existing codebase conventions
- Structural anti-patterns (circular dependencies, god modules)
- Module boundary integrity
- Database access patterns (N+1 queries, missing indexes for new queries)

**Severity guide:**
- **Critical:** Layer violation that creates coupling debt; circular dependency introduced
- **Warning:** Pattern deviation from established conventions; questionable dependency direction
- **Suggestion:** Opportunity to improve module organisation

## 4. Test Coverage

**Goal:** Verify that new and changed behaviours are adequately tested.

**What to check:**
- Every new function/method has corresponding tests
- Changed behaviour has updated tests
- Tests are meaningful (not just "it doesn't crash" smoke tests)
- Edge cases covered: null, empty, boundary, error paths
- Integration points tested (API contracts, database queries)
- Test isolation: tests don't depend on external state or ordering
- No test code in production paths

**Severity guide:**
- **Critical:** Core behaviour entirely untested; tests pass for wrong reasons
- **Warning:** Missing edge case tests; changed behaviour without test update
- **Suggestion:** Test could be more descriptive; minor gap in coverage

## 5. Security

**Goal:** Identify vulnerabilities following OWASP top 10 and secure coding practices.

**What to check:**
- **Injection:** SQL injection, XSS, command injection, template injection
- **Auth/Authz:** Missing authentication checks, privilege escalation, broken access control
- **Secrets:** Hardcoded credentials, API keys in code, secrets in logs
- **Input validation:** Unsanitised user input, missing length/type checks at boundaries
- **Data exposure:** Sensitive data in responses, verbose error messages, PII in logs
- **CSRF/SSRF:** Missing CSRF tokens, unrestricted server-side requests
- **Dependencies:** Known vulnerable packages (if visible in diff)
- **Cryptography:** Weak algorithms, improper key management

**Severity guide:**
- **Critical:** Exploitable vulnerability (injection, auth bypass, secrets exposure)
- **Warning:** Missing validation at a boundary; potential for data leakage
- **Suggestion:** Hardening opportunity; defence-in-depth improvement
