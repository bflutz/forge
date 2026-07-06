---
name: ci-gate-template
description: |
  Generates a `.github/workflows/pr-checks.yml` GitHub Actions workflow that gates merges to the default branch on lint, typecheck, test, and (for Firebase projects) security-rules-test jobs running against the Firebase emulator. Composes with [[github-branch-protection]] — the workflow's job names become the required status checks. Supports Node + npm/pnpm projects; Firebase-aware via `firebase-tools` install + `emulators:exec`.

  Use this skill when the user wants to set up CI, gate merges, add GitHub Actions, run tests on PR, block merges on failing checks, or says things like "make sure tests pass before merge", "add CI", "run the build on PR", "gate the master branch". Triggers on phrases like "github actions", "workflow file", "pr-checks.yml", "required status check", "ci workflow". Auto-triggers on detection of a project lacking `.github/workflows/` directory when the user mentions PRs or merging.

  Do NOT use for deployment workflows (this is a PR-gate, not a deploy pipeline). Do NOT use for branch protection configuration itself (use [[github-branch-protection]]). Do NOT use for non-GitHub CI (CircleCI, GitLab — out of scope; this workspace standardises on GitHub Actions).

metadata:
  type: skill
  domain: github, ci, devops
  version: 1.0.0
---

# CI Gate Template

Produces a single workflow file that runs as a required status check on pull requests targeting the default branch. The workflow is structured so each gate is a separate **job**, not a single step in a monolithic job — separate jobs surface as separate status-check names in GitHub, which [[github-branch-protection]] can require individually.

Pairs with [[firebase-emulator-loop]] (which makes `npm run emulators:exec` exist) and [[firebase-rules-unit-tests]] (which writes the test files the rules-test job runs).

---

## Expert Vocabulary Payload

**Workflow Anatomy:** `on: pull_request`, `on: push`, jobs vs steps, `runs-on`, `actions/checkout@v4`, `actions/setup-node@v4`, matrix strategy, `needs:` (job dependencies), `if:` conditional, `concurrency` group, cancel-in-progress
**Caching + Speed:** `actions/cache@v4`, `setup-node` built-in npm cache, pnpm-store cache, restore-keys, partial cache hit, cold-start vs warm-start
**Status Check Surface:** check name = job name (when no matrix) or `<job> (<matrix-key>)`, required vs optional, contexts in branch protection, GitHub merge queue compatibility
**Firebase in CI:** `firebase-tools` install (`npm i -g firebase-tools` or use as devDep), `emulators:exec` exit-code propagation, Java requirement (`actions/setup-java`), emulator cache `~/.cache/firebase/emulators`

---

## Anti-Pattern Watchlist

### 1. Single mega-job hiding partial failures
- **Detection:** One job runs lint + typecheck + test + rules-test sequentially. The status-check surface is one name; branch protection can only require all-or-nothing.
- **Why it fails:** Can't distinguish "lint failed" from "rules-test failed" at the protection layer. Can't surface partial green to humans. Re-running re-does everything.
- **Resolution:** Split into separate jobs per gate. Use `needs:` only when a real dependency exists (e.g. nothing needs to run before lint).

### 2. `on: push` without `on: pull_request`
- **Detection:** Workflow only triggers on push to main. PRs from branches don't run it.
- **Why it fails:** The whole point of the gate is to run on PRs BEFORE merge. Triggering only on push to main is a post-merge audit, not a gate.
- **Resolution:** Use `on: pull_request: { branches: [main] }` (substitute `master` if that's the project's default — check via `gh repo view --json defaultBranchRef`). Optionally also `on: push: { branches: [main] }` to re-verify the merge commit itself.

### 3. Required cache miss penalty
- **Detection:** Workflow installs npm deps fresh every run with no cache.
- **Why it fails:** Adds 30-120s to every PR; developers stop trusting fast feedback and stop running locally.
- **Resolution:** Use `actions/setup-node@v4` with `cache: 'npm'` (or `'pnpm'`). For pnpm, also configure `pnpm/action-setup` before setup-node.

### 4. Missing Java for Firebase emulator
- **Detection:** Workflow runs `firebase emulators:exec` without first setting up Java.
- **Why it fails:** Firestore + Storage emulators are JVM-based. Without Java, emulator fails to start with a cryptic "command not found" or jar-missing error.
- **Resolution:** Add `actions/setup-java@v4` with `distribution: temurin`, `java-version: 17` before any `firebase` step.

### 5. `npm ci` vs `npm install` confusion
- **Detection:** Workflow uses `npm install` (resolves fresh, may drift) instead of `npm ci` (deterministic, fails on lockfile drift).
- **Why it fails:** PR may pass CI on resolved-fresh deps and fail in prod when the lockfile is honoured. Defeats the lockfile's purpose.
- **Resolution:** Always `npm ci` (or `pnpm install --frozen-lockfile`). Treat lockfile drift as a build failure surface, not a thing to paper over.

### 6. Required check name drift
- **Detection:** Job renamed in workflow file; branch protection still references the old name; PRs can't merge because the "required check" never reports.
- **Why it fails:** A silent contract between two files in different repos (`.github/workflows/pr-checks.yml` and the branch-protection API state).
- **Resolution:** When a job is renamed, immediately re-run [[github-branch-protection]] to update `required_status_checks.contexts`. Better: don't rename jobs casually.

### 7. Secrets in workflow logs
- **Detection:** Workflow `echo`s env vars that include secrets, or prints `firebase deploy` token usage.
- **Why it fails:** Logs are visible to anyone with read access on the repo. GitHub auto-masks `${{ secrets.X }}` but only for direct references — once echoed via shell expansion in some shells, masking can be bypassed.
- **Resolution:** Never echo secrets. The PR-gate workflow this skill produces does NOT need any secrets (rules tests run against the emulator with no real creds). If a future workflow needs secrets, that's a separate skill.

---

## Behavioral Instructions

### Phase 1: Inspect the project

1. Detect package manager: `package-lock.json` → npm, `pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn. (We support npm and pnpm; if yarn, ask user to pick npm/pnpm.)
2. Read `package.json` scripts. Confirm which gates the project supports: `lint`, `typecheck`, `test`, `test:rules`. Skip any gate the project doesn't have.
3. Detect Firebase: presence of `firebase.json`. IF present: include the `rules-test` job and Java setup.
4. Read `.github/workflows/`. IF a `pr-checks.yml` already exists: read it, propose a diff rather than overwrite. IF the directory doesn't exist: it will be created.
5. Identify the default branch (`gh repo view --json defaultBranchRef --jq .defaultBranchRef.name`). Used in the `on:` block.

### Phase 2: Generate the workflow

6. Start from `references/workflow.yml`.
7. Substitute placeholders: Node version (read from `.nvmrc` or `engines.node` in package.json — default to `20` if unset), package manager, default branch, package manager install command.
8. Include only the jobs that match the project's scripts. Omit `rules-test` if not a Firebase project.
9. The workflow file path is `.github/workflows/pr-checks.yml`. Create the directory if needed.

### Phase 3: Verify locally before committing

10. Lint the YAML — read it back and confirm no tab/space mixing and no duplicate keys.
11. Confirm with the user the exact job names that will appear in GitHub's check surface; these are what [[github-branch-protection]] needs to reference.
12. Suggest a sequence to the user: (a) merge this workflow to default branch, (b) open a throwaway PR to confirm jobs run, (c) THEN apply branch protection via [[github-branch-protection]]. Do NOT apply protection before the workflow has run once.

### Phase 4: Hand off

13. Report: file path, job names, package manager assumed, Firebase yes/no, follow-up sequence.

---

## Output Format

```yaml
workflow_file: .github/workflows/pr-checks.yml
default_branch: <name>
package_manager: npm | pnpm
node_version: <version>
jobs_emitted:
  - lint
  - typecheck
  - test
  - rules-test   # only if Firebase
firebase_aware: true | false
follow_ups:
  - Merge workflow to default branch
  - Open throwaway PR to confirm jobs run
  - Run [[github-branch-protection]] with these job names as required contexts
```

---

## Examples

### Example 1: Atlas risk dashboard (Next.js + Firebase + vitest, npm)

Input: `package.json` has `lint`, `typecheck`, `test` scripts; `firebase.json` exists; lockfile is `package-lock.json`.

Action:
1. Detect npm, Node 20 default, Firebase yes.
2. Generate workflow with jobs: `lint`, `typecheck`, `test`, `rules-test`. The `rules-test` job sets up Java 17, installs firebase-tools via npx, runs `npm run emulators:exec`.
3. Write to `.github/workflows/pr-checks.yml`.
4. Tell user: required-check names will be `lint`, `typecheck`, `test`, `rules-test`. Pass these to [[github-branch-protection]] after the workflow lands on the default branch.

### Example 2: Non-Firebase Next.js project

Input: `package.json` has `lint`, `typecheck`, `test`; no `firebase.json`.

Action:
1. Detect npm, no Firebase.
2. Generate workflow with `lint`, `typecheck`, `test` jobs only. Omit Java setup and `rules-test` job.
3. Same handoff sequence.

### Example 3: BAD — single job, post-merge only

WRONG workflow:
```yaml
on: push
  branches: [main]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - run: npm install && npm run lint && npm test
```

Three failures:
- `on: push` not `pull_request` — runs after merge, not as a gate.
- Single job — branch protection can't require lint and test separately.
- `npm install` not `npm ci` — non-deterministic.

RIGHT workflow: see `references/workflow.yml`.

---

## Questions This Skill Answers

- "Set up CI for this project"
- "Add GitHub Actions to run tests on PR"
- "Gate merges on lint and tests"
- "Make sure tests pass before merge"
- "Add a workflow file"
- "Run the Firebase emulator in CI"
- "Block merges if the build fails"
- "Set up pr-checks.yml"
- "I need a required status check"
- "Run vitest in GitHub Actions"
- "How do I run firebase emulators:exec in CI?"
- "Add typecheck to the CI"
- "Why aren't my PR checks running?"
