---
name: firebase-rules-unit-tests
description: |
  Scaffolds security-rule unit tests using `@firebase/rules-unit-testing` against the Firebase Local Emulator Suite. Produces `firestore.rules.test.ts` and `storage.rules.test.ts` files that exercise allow/deny matrices per collection using `initializeTestEnvironment`, `assertSucceeds`, `assertFails`. Wires the tests into the project's existing test runner (vitest, jest, mocha) and into `firebase emulators:exec`.

  Use this skill when the user wants to test Firestore rules, test storage rules, write security rules unit tests, validate `firestore.rules`, check that admin/viewer/owner permissions actually work, or says things like "test the rules", "make sure unauthenticated reads fail", "cover the admin-only paths". Triggers on phrases like "rules-unit-testing", "security rules test", "firebase rules test", "assertSucceeds", "assertFails". Auto-triggers on detection of a `firestore.rules` or `storage.rules` file with no matching `*.rules.test.ts` neighbour.

  Do NOT use for writing the rules themselves (that's normal Firestore rules authoring). Do NOT use for emulator configuration (use [[firebase-emulator-loop]] which sets up the emulators block and exec scripts that this skill's tests run inside). Do NOT use for production rule deployment (use `firebase deploy --only firestore:rules`).

metadata:
  type: skill
  domain: firebase, testing, security
  version: 1.0.0
---

# Firebase Rules Unit Tests

Scaffolds executable security-rule tests using the official `@firebase/rules-unit-testing` package. Tests assert that the rules permit what they should and deny what they shouldn't, across authenticated / unauthenticated / role-specific identities. Pairs with [[firebase-emulator-loop]] for execution and [[ci-gate-template]] for CI enforcement.

This is the **preventive** counterpart to the existing `code-review-checklist` skill (which is diagnostic — it spots gaps but doesn't fix them). Where code-review-checklist asks "do your rules deny anonymous writes?", this skill produces a test that fails the build if they don't.

Canonical reference: https://firebase.google.com/docs/ai-assistance/prompt-catalog/write-security-rules.

---

## Expert Vocabulary Payload

**Rules Testing API:** `initializeTestEnvironment`, `RulesTestEnvironment`, `RulesTestContext`, `withSecurityRulesDisabled` (seed only), `unauthenticatedContext`, `authenticatedContext`, `assertSucceeds`, `assertFails`, `clearFirestore`, `clearStorage`
**Identity Modelling:** custom claims, auth token shape, `request.auth.uid`, `request.auth.token`, `get(/databases/.../documents/users/...)` role lookup, role hierarchy (viewer / admin / super-admin)
**Test Topology:** allow/deny matrix, positive case, negative case, boundary case, cross-collection lookup, `beforeEach` seeding, deterministic test data
**Rule Failure Modes:** fall-through wildcard, missing role check, missing auth check, ownership confusion, `get()` recursion blow-up, list vs get distinction

---

## Anti-Pattern Watchlist

### 1. Positive-only test suite
- **Detection:** Every test calls `assertSucceeds`. Zero `assertFails`.
- **Why it fails:** The rules could be `allow read, write: if true;` and your tests would still pass. The whole point of security rules is what they DENY; tests must prove denials.
- **Resolution:** For every collection, write at least one positive and one negative test. Cover: unauthenticated denial, wrong-role denial, ownership-violation denial.

### 2. Seeding with rules enabled
- **Detection:** Test setup writes seed data using a regular `authenticatedContext` and the test data setup itself respects the rules.
- **Why it fails:** Either you can only test code paths that the rules permit (tautology — rules let you set up what rules let you set up), or you have to use admin credentials in tests.
- **Resolution:** Seed inside `testEnv.withSecurityRulesDisabled(async (ctx) => { ... })`. This bypasses rules only for seeding; the actual assertions still hit the live rules.

### 3. Real project ID in tests
- **Detection:** `initializeTestEnvironment({ projectId: 'my-real-firebase-project' })` or env vars pointing at staging.
- **Why it fails:** A commonly flagged risk in security reviews. A misconfigured test could write to or wipe a real database.
- **Resolution:** Use `demo-<name>` project IDs (e.g. `demo-atlas`). The Firebase CLI treats `demo-` prefixed projects as emulator-only; the Admin SDK refuses to authenticate them against real Firebase.

### 4. Skipping `clearFirestore` between tests
- **Detection:** Tests share state; running test B alone passes, running A-then-B fails.
- **Why it fails:** State pollution makes tests order-dependent and hides real failures.
- **Resolution:** `beforeEach(async () => { await testEnv.clearFirestore(); })`. For Storage tests, `await testEnv.clearStorage()`.

### 5. Testing rules against the JS SDK instead of via the test environment
- **Detection:** Tests directly use `firebase/firestore` `setDoc` against a real Firebase config, hoping the emulator catches it.
- **Why it fails:** You're testing the SDK, not the rules. Mocking gets brittle, identity modelling is impossible, role lookups don't work.
- **Resolution:** Always go through `testEnv.authenticatedContext(uid, { role: 'admin' }).firestore()` or `testEnv.unauthenticatedContext().firestore()`. These return a Firestore SDK instance pre-wired to the emulator with the test identity.

### 6. Forgetting `get()` recursion in role checks
- **Detection:** Rules use `get(/databases/.../users/$(request.auth.uid)).data.role` to check role. Tests don't seed the `/users/{uid}` document.
- **Why it fails:** The `get()` call fails inside the rule, the rule denies, the test fails with a misleading "permission denied" instead of a clear "role doc missing" — wasting debug time.
- **Resolution:** In `beforeEach`, seed a `/users/{uid}` document for every test identity before exercising any other collection. See `references/rules-test-patterns.md` for the canonical seeding helper.

---

## Behavioral Instructions

### Phase 1: Read the rules

1. Read `firestore.rules`. Identify every `match` block and the role(s) it checks.
2. Identify every role helper function (`isAdmin`, `isOwner`, `isAuthenticated`, etc.).
3. Identify any cross-collection lookups (`get(...)`, `exists(...)`).
4. Repeat for `storage.rules` if present.

### Phase 2: Build the allow/deny matrix

5. For each match block, enumerate identities × operations:
   - Identities: unauthenticated, authenticated-no-role, viewer, admin, super-admin, owner-of-doc, other-user.
   - Operations: get, list, create, update, delete.
6. For each (identity × operation) cell, decide expected outcome from the rule. This matrix is the test plan.
7. IF the matrix has > 30 cells: prioritise. Cover every DENY case the rule intends (high value), then sample ALLOW cases (rules permitting access are usually less risky to under-test).

### Phase 3: Scaffold the test environment

8. Add `@firebase/rules-unit-testing` to `devDependencies` (`>=4.0.0`).
9. Create `tests/rules/helpers.ts` with a single `getTestEnv()` helper and a `seedUserDoc(ctx, uid, role)` helper. See `references/rules-test-patterns.md` for the canonical body.
10. The test runner is whatever the project already uses — read `package.json` scripts to identify (vitest / jest / mocha). Do NOT introduce a new runner.

### Phase 4: Write the tests

11. Create `tests/rules/firestore.rules.test.ts` (or `.spec.ts` matching project convention). One `describe` block per match block. One `it` per allow/deny matrix cell, named in the form `"<identity> CAN/CANNOT <operation> <collection>"`.
12. Use `assertSucceeds(promise)` for allows, `assertFails(promise)` for denies. Never just `await` — `assertFails` distinguishes "denied by rule" from "any error".
13. Use `testEnv.withSecurityRulesDisabled` ONLY in `beforeEach` for seed data. Never inside an `it`.
14. Create `tests/rules/storage.rules.test.ts` with the same shape if `storage.rules` exists.

### Phase 5: Wire into the runner

15. Add `"test:rules": "<runner> run tests/rules"` to `package.json` scripts.
16. IF the project uses vitest, add `tests/rules/**` to the test include pattern in `vitest.config.ts` (only if it isn't picked up by default globbing).
17. Add a one-line note in `tests/rules/README.md` (or project's existing tests README): "These tests run against the Firebase emulator. Run via `npm run emulators:exec` after [[firebase-emulator-loop]] has been wired."

### Phase 6: Verify

18. Run `npm run emulators:exec` (assuming [[firebase-emulator-loop]] is wired). All tests should pass.
19. Deliberately break one rule (e.g. flip `if isAdmin()` to `if true`), re-run, confirm a relevant `assertFails` test now correctly fails.
20. Revert the deliberate break. Final run should be green.

---

## Output Format

```yaml
project: <name>
rules_files_analysed: [firestore.rules, storage.rules]
match_blocks_covered: <count>
test_cases_written: <count>
allow_assertions: <count>
deny_assertions: <count>
files_added:
  - tests/rules/helpers.ts
  - tests/rules/firestore.rules.test.ts
  - tests/rules/storage.rules.test.ts
files_modified:
  - package.json
deliberate_break_verified: yes | no
follow_ups:
  - <e.g. "pair with ci-gate-template to gate PRs on test:rules">
```

---

## Examples

### Example 1: Atlas rules with role-based access

Input: `firestore.rules` defines `isAuthenticated`, `isAdmin`, `isSuperAdmin`. `/users/{uid}` lets users create their own viewer profile, super-admin manages others. `/admin_tool_runs/*` and `/analyst_chat_*` are admin-only. Catch-all `/{collection}/{docId}` is authenticated-read, admin-write.

Action:
1. Build matrix: 6 identities × 5 operations × 4 collection groups = ~120 cells. Prioritise denies.
2. Scaffold `tests/rules/helpers.ts` with `getTestEnv({ projectId: 'demo-atlas' })`.
3. Write `firestore.rules.test.ts` covering:
   - Unauthenticated CANNOT read anything (catch-all denial).
   - Authenticated viewer CAN read `/alerts/*`, CANNOT write.
   - Admin CAN write `/alerts/*`, CAN read `/admin_tool_runs/*`.
   - Non-super-admin admin CANNOT list `/users`.
   - User CAN create own `/users/{uid}` with role=viewer, CANNOT create with role=admin.
4. Write `storage.rules.test.ts`: unauthenticated AND authenticated CANNOT read or write (storage is fully denied by current rules).
5. Wire `test:rules` script using vitest (project default).
6. Run via emulator, verify, break-and-revert.

### Example 2: BAD — testing the wrong layer

A developer writes a test that calls a Next.js API route, which internally uses the Firebase Admin SDK. The test "covers the rules" because the API route enforces auth.

**Why it's wrong:** Admin SDK bypasses Firestore rules entirely. This tests the API route's middleware, not the rules. If the rules are wide-open and someone hits Firestore directly via the client SDK, the API-route test won't catch it.

**Correct approach:** Test rules at the rules layer — through `@firebase/rules-unit-testing` against the emulator using the client SDK shape. The API route gets its own integration test separately.

### Example 3: Owner-only pattern

Rule: `match /notes/{noteId} { allow read, write: if request.auth.uid == resource.data.ownerId; }`

Tests (the most representative — recency-biased to last):
- `unauthenticated CANNOT read /notes/n1` → `assertFails`
- `user-a CAN read /notes/n1 when ownerId=user-a` → `assertSucceeds`
- `user-b CANNOT read /notes/n1 when ownerId=user-a` → `assertFails`
- `user-a CANNOT update /notes/n1 to set ownerId=user-b (ownership transfer)` → `assertFails`

Seed `/notes/n1` with `ownerId: 'user-a'` inside `withSecurityRulesDisabled` before each test.

---

## Questions This Skill Answers

- "Test my Firestore security rules"
- "Add unit tests for firestore.rules"
- "Use @firebase/rules-unit-testing to verify my rules"
- "How do I test that admin-only writes actually require admin?"
- "Make sure unauthenticated users can't write to alerts"
- "Cover the storage rules with tests"
- "Verify the owner-only pattern works"
- "Set up assertSucceeds and assertFails for our rules"
- "I want a test that fails if someone weakens the admin check"
- "Scaffold rules tests for this Firebase project"
- "Write security rule tests in vitest"
- "Test the user role escalation rule"
- "Why are my rules tests passing but prod is wide open?"
