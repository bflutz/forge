---
name: firebase-emulator-loop
description: |
  Runs Firebase Local Emulator Suite as a pre-push feedback loop: configures the emulators block in `firebase.json`, wires `firebase emulators:exec` scripts, runs the project's test suite against emulated Firestore / Storage / Auth / Functions, and iterates fix-then-rerun until green. Uses the Firebase CLI's emulator-exec pattern (auto-start, run command, auto-shutdown, propagate exit code).

  Use this skill when the user wants to test against the emulator, run a pre-push check, set up local Firebase tests, validate security rules before deploy, or says things like "test this without touching prod", "run it against the emulator first", "loop the tests until they pass". Triggers on phrases like "firebase emulator", "emulators:exec", "pre-push", "local firebase tests". Auto-triggers on detection of `firebase.json` that lacks an `emulators` block.

  Do NOT use for production deploys (use `firebase deploy` directly) or for CI workflow scaffolding (use [[ci-gate-template]] which embeds emulator-exec inside a GitHub Actions job). Do NOT use for writing the security rules themselves — only for exercising them. Pair with [[firebase-rules-unit-tests]] which scaffolds the actual test files this loop executes.

metadata:
  type: skill
  domain: firebase, testing, devops
  version: 1.0.0
---

# Firebase Emulator Loop

Pre-push validation harness using the Firebase Local Emulator Suite. The loop runs `firebase emulators:exec '<test command>'` — the CLI auto-starts the configured emulators, runs the command, propagates exit code, and shuts down cleanly. No prod credentials touched.

---

## Expert Vocabulary Payload

**Emulator Suite:** emulators:exec (one-shot), emulators:start (long-running), emulator UI, seeded data (`--import`/`--export-on-exit`), `FIREBASE_EMULATOR_HUB`, Firestore emulator host, Storage emulator host, Auth emulator host, Functions emulator
**Process Wiring:** exit-code propagation, port allocation, port collision, `firebase-tools` minimum version, project alias (`--project demo-x`), demo project (no creds required)
**Rules + Test Integration:** `@firebase/rules-unit-testing` `initializeTestEnvironment`, `clearFirestore()`, `connectFirestoreEmulator`, `RulesTestEnvironment.withSecurityRulesDisabled` (seed only), `assertSucceeds` / `assertFails`
**Feedback Loop:** baseline run, fix-rerun cycle, deterministic seed, flake isolation, stderr capture, `firebase-debug.log`

---

## Anti-Pattern Watchlist

### 1. Running tests against production / staging credentials
- **Detection:** `firebase emulators:exec` invoked without `--project demo-<name>`, or test code that calls `initializeApp` with real Firebase config instead of `initializeTestEnvironment`.
- **Why it fails:** A bug in the test deletes or mutates real user data. Firebase's own security checklist stresses this precisely because emulator isolation is what makes the rest of the loop safe.
- **Resolution:** Always pass `--project demo-<projectname>` (the `demo-` prefix tells the CLI no real project exists; no creds needed). Tests must use `initializeTestEnvironment({ projectId: 'demo-x' })`, never the production Firebase config.

### 2. Long-running `emulators:start` for one-shot CI
- **Detection:** Scripts use `emulators:start` + background process + manual kill, instead of `emulators:exec '<cmd>'`.
- **Why it fails:** Manual lifecycle management leaks processes, hangs CI, and swallows exit codes. `emulators:exec` is the documented pattern; it handles startup, exec, and shutdown atomically and propagates the inner command's exit code.
- **Resolution:** Use `emulators:exec` for any one-shot run (local pre-push, CI). Reserve `emulators:start` for interactive development against the Emulator UI.

### 3. Port collisions across the workspace
- **Detection:** Two projects in the workspace both default to Firestore port 8080 / Auth port 9099 / Storage port 9199. Running two `emulators:start` at once collides.
- **Why it fails:** Hard-to-diagnose "address in use" errors; tests pass alone but fail when another emulator is running.
- **Resolution:** Allocate per-project port blocks. See `references/emulator-ports.md` for the workspace map. Each project gets a 100-port band offset from defaults.

### 4. No `singleProjectMode` → cross-test pollution
- **Detection:** `emulators` block lacks `"singleProjectMode": true`; tests share emulator state and fail intermittently when run in different orders.
- **Why it fails:** Without single-project mode, the emulator allows multiple projectIds and persists data across them, masking missing `clearFirestore()` calls.
- **Resolution:** Set `"singleProjectMode": true` in the `emulators` block. Pair with `await testEnv.clearFirestore()` in `beforeEach`.

### 5. Loop without a stopping condition
- **Detection:** Script re-runs `emulators:exec` after every code change with no exit on success, no max-iteration cap, no human-in-the-loop.
- **Why it fails:** A genuinely broken test loops forever consuming CPU; a flaky test masks itself by passing on retry.
- **Resolution:** The loop runs once per push, not continuously. For pre-push hook use `husky` + a single `emulators:exec` call; if it fails, push is rejected and the developer fixes locally.

---

## Behavioral Instructions

### Phase 1: Detect & confirm

1. Read `firebase.json`. IF an `emulators` block already exists: report current port allocation, skip to Phase 3. ELSE: proceed to Phase 2.
2. Identify which Firebase services this project uses by scanning `firebase.json` keys (`firestore`, `storage`, `functions`, plus presence of `auth` usage via `firebase/auth` imports).
3. Confirm with the user which services to emulate. Default: every service in `firebase.json` plus Auth if the codebase imports it.

### Phase 2: Configure `firebase.json` emulators block

4. Pick a port band for this project from `references/emulator-ports.md`. IF no entry exists for this project, add one before allocating.
5. Add the `emulators` block to `firebase.json`. Required shape:
   ```json
   "emulators": {
     "auth":      { "port": <band+99> },
     "firestore": { "port": <band+80> },
     "storage":   { "port": <band+199> },
     "functions": { "port": <band+5> },
     "ui":        { "enabled": true, "port": <band+0> },
     "singleProjectMode": true
   }
   ```
   Omit any service the project does not use.
6. Add `firebase-tools` to `devDependencies` IF not already present (`>=14.0.0`). Do NOT add as a dependency.

### Phase 3: Wire npm scripts

7. Add scripts to `package.json`:
   - `"emulators:start": "firebase emulators:start --project demo-<projectname>"`
   - `"emulators:exec":  "firebase emulators:exec --project demo-<projectname> 'npm run test:rules'"`
   - IF the project has a broader test suite that should also run against the emulator, add `"test:emulated": "firebase emulators:exec --project demo-<projectname> 'npm test && npm run test:rules'"`.
8. Substitute `<projectname>` with a slug derived from the directory name (lowercase, hyphens). The `demo-` prefix is mandatory.

### Phase 4: Execute the loop

9. Run `npm run emulators:exec` once.
10. IF exit code 0: report success and stop.
11. IF non-zero: read the failing test output. Identify the rule violation or test assertion that failed. Propose a single fix (rules change OR test change — name which). Confirm with the user before applying.
12. Apply the fix, re-run `npm run emulators:exec`. Repeat steps 10-12 up to **5 iterations**. After 5, stop and surface the state to the user — do not loop indefinitely (anti-pattern #5).

### Phase 5: Verify the integration

13. Confirm the test command exits non-zero when a rule is deliberately broken (insert a temporary `allow read: if false;` in one rule, re-run, verify failure, revert).
14. Confirm `firebase-debug.log` is in `.gitignore`.
15. Suggest pairing with [[ci-gate-template]] to enforce this loop on every PR.

---

## Output Format

```yaml
project: <name>
firebase_services_emulated: [firestore, storage, auth, functions]
port_band: <number>
scripts_added: [emulators:start, emulators:exec, ...]
loop_result: success | failed_after_N_iterations | aborted
iterations: <number>
files_modified:
  - firebase.json
  - package.json
follow_ups:
  - <next step, e.g. "pair with ci-gate-template">
```

---

## Examples

### Example 1: First-time setup on a Firebase project with no emulator config

Input: the Atlas project, `firebase.json` has `firestore` and `storage` blocks only.

Action:
1. Detect no `emulators` block.
2. Allocate port band 9000-9099 from `references/emulator-ports.md`.
3. Add `emulators` block with `firestore: 9080`, `storage: 9199`, `auth: 9099`, `ui: 9000`, `singleProjectMode: true`. Skip `functions` (the project's functions live in a separate `functions/firebase.json`).
4. Add scripts `emulators:start`, `emulators:exec`, `test:rules`.
5. Run `npm run emulators:exec` → fails (no rules tests yet).
6. Stop. Tell user: "Wire [[firebase-rules-unit-tests]] before re-running the loop."

### Example 2: Re-running the loop after a rule change

Input: A developer edits `firestore.rules` to add a new collection, runs the skill again.

Action:
1. Detect `emulators` block already present, skip to Phase 4.
2. Run `npm run emulators:exec`.
3. Test fails: "expected allow, got deny for /alerts/{docId} as admin".
4. Read the rule. The catch-all `match /{collection}/{docId}` has `allow write: if isAdmin()`. The new collection has a more specific match that's missing the admin write rule.
5. Propose fix: either add the missing rule, or remove the more-specific match so it falls through to the catch-all.
6. Apply fix, re-run, green. Stop.

### Example 3: BAD — looping forever on a flaky test

A test fails intermittently due to clock-dependent assertions. The skill should NOT re-run hoping it passes; that masks the flake. Instead, after iteration 2 with the same test failing for different reasons, stop and surface: "Test `expires-after-1-day` looks flaky (different failure modes across iterations). Fix the test before continuing the loop."

---

## Questions This Skill Answers

- "Set up the Firebase emulator suite for this project"
- "Run my rules tests against the emulator"
- "Test this without touching production"
- "Pre-push check for Firebase"
- "Why are my emulator ports colliding?"
- "Add emulators:exec to package.json"
- "Loop the tests until they pass"
- "How do I run security rules tests locally?"
- "I want to validate my rules before deploy"
- "Configure firebase.json for the local emulator"
- "Wire up firebase-tools for testing"
- "Make a feedback loop for firebase rule changes"
