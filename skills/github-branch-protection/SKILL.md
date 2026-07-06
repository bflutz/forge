---
name: github-branch-protection
description: |
  Applies a branch protection rule to the default branch of a GitHub repo using the `gh api` CLI. Requires PRs (no direct push), requires N approving reviews, dismisses stale reviews on new commits, requires named status checks to pass, blocks force-pushes and branch deletion. Idempotent — re-running converges to the desired state. Solo-author mode variant requires PRs without requiring a separate approver.

  Use this skill when the user wants to protect main, protect master, require PR approval, set up branch protection rules, prevent direct pushes to the default branch, or says things like "stop me pushing to master", "require a PR", "gate the default branch", "make merges go through review". Triggers on phrases like "branch protection", "protected branches", "require status checks", "dismiss stale reviews".

  Do NOT use for CODEOWNERS configuration (separate file, separate skill). Do NOT use for repository-level settings unrelated to branches (merge button, default branch, visibility — those are different `gh api` endpoints). Pair with [[ci-gate-template]] so the required-status-check names match real workflow job names.

metadata:
  type: skill
  domain: github, devops, governance
  version: 1.0.0
---

# GitHub Branch Protection

Wraps the GitHub REST API endpoint `PUT /repos/{owner}/{repo}/branches/{branch}/protection` in a confirmable, idempotent skill. The endpoint is documented at https://docs.github.com/en/rest/branches/branch-protection (human-facing equivalent: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/managing-a-branch-protection-rule).

This is a **high-blast-radius shared-state action**. The skill ALWAYS confirms the exact payload with the user before sending. See workspace `CLAUDE.md` global guidance on shared-state actions.

---

## Expert Vocabulary Payload

**Branch Protection Primitives:** required pull request reviews, required approving review count, dismiss stale reviews, require code owner reviews, required status checks (strict / loose), require branches up-to-date, required conversation resolution, enforce admins, allow force pushes, allow deletions, lock branch
**GitHub REST API:** `PUT /repos/{owner}/{repo}/branches/{branch}/protection`, `Accept: application/vnd.github+json`, idempotent PUT, ruleset (newer alternative), classic branch protection (legacy but still default)
**Authorisation Surface:** `repo` OAuth scope, `admin:repo` scope, `gh auth status`, fine-grained PAT vs classic PAT, organisation vs user-owned repo, ruleset bypass actors
**Solo-Author Patterns:** PR-required-no-approver (require_pull_request_reviews omitted or count=0), self-approve-via-fork, codeowners-as-self

---

## Anti-Pattern Watchlist

### 1. Applying without dry-run
- **Detection:** Skill immediately calls `gh api ... -X PUT` without first printing the payload and waiting for confirmation.
- **Why it fails:** A typo in `required_status_checks.contexts` can lock a repo into requiring a check that doesn't exist; the only fix is a separate API call to unblock. Even when the request is explicit, the user owns the repo — confirm before mutating.
- **Resolution:** Always print the full request payload + the `gh api` command, then ask the user to confirm before sending. See `references/protection-payload.json` for templates.

### 2. Required status checks that don't exist yet
- **Detection:** Setting `required_status_checks.contexts: ["build", "test"]` before a workflow with those job names is on the default branch.
- **Why it fails:** GitHub will not auto-create the check; the protection requires a check that nothing produces, so PRs can never merge.
- **Resolution:** Ensure the workflow file (typically from [[ci-gate-template]]) is merged to the default branch FIRST, then trigger one run so GitHub knows the check name exists, THEN apply protection referencing those names. If applying before the workflow, set `required_status_checks: null` initially and re-run the skill to add them later.

### 3. Solo author locked out of own repo
- **Detection:** `required_approving_review_count: 1` applied on a repo where only one person ever pushes; PRs can never accumulate an approving review (you can't approve your own PR).
- **Why it fails:** PRs become unmergeable; the user disables protection in frustration, defeating the point.
- **Resolution:** For solo repos, use the "solo author" variant in `references/protection-payload.json`: requires PR (forces a merge-button click and CI run), but omits the approving-review requirement. As soon as a collaborator joins, switch to the standard variant.

### 4. Forgetting `enforce_admins`
- **Detection:** Protection applied without `enforce_admins: true`. Admins (which is usually the user) can still push directly to the protected branch, undermining the whole purpose.
- **Why it fails:** The user wanted a guardrail against their own habits, not a guardrail that explicitly exempts them.
- **Resolution:** Default to `enforce_admins: true`. Only relax if the user explicitly asks (e.g. emergency hotfix workflow).

### 5. Force-push left enabled
- **Detection:** `allow_force_pushes.enabled: true` in the payload, or the key omitted (default behaviour).
- **Why it fails:** Force-push lets anyone with write access rewrite history on the protected branch, voiding the PR audit trail.
- **Resolution:** Always set `allow_force_pushes: { enabled: false }` and `allow_deletions: { enabled: false }`.

### 6. Skipping `gh auth status`
- **Detection:** Skill assumes `gh` is authenticated and skips the preflight check.
- **Why it fails:** First failure mode for the entire workflow; you waste user time presenting a payload that's about to fail with 401.
- **Resolution:** Run `gh auth status` first. IF unauthenticated: stop and instruct user to run `gh auth login` (interactive — must run in their own shell with `!` prefix).

---

## Behavioral Instructions

### Phase 1: Preflight

1. Run `gh auth status`. IF unauthenticated: surface the message and stop. Tell the user to type `! gh auth login`.
2. Identify the repo and branch. Default branch comes from `gh repo view --json defaultBranchRef --jq .defaultBranchRef.name`. Confirm with the user before proceeding.
3. Identify how many active committers the repo has (`gh api repos/{owner}/{repo}/contributors --jq '. | length'`). IF 1: recommend the solo-author variant.
4. Check whether [[ci-gate-template]] has produced a workflow on the default branch. IF yes: collect the job names — these become `required_status_checks.contexts`. IF no: warn the user that protection will go in without status-check requirements and they should re-run the skill after the workflow lands.

### Phase 2: Build the payload

5. Start from `references/protection-payload.json` (standard or solo-author variant).
6. Populate `required_status_checks.contexts` with the actual workflow job names from Phase 1. IF none available: set `required_status_checks: null` and note it as a follow-up.
7. Set `enforce_admins: true` unless the user explicitly opts out.
8. Set `allow_force_pushes: { enabled: false }`, `allow_deletions: { enabled: false }`.

### Phase 3: Confirm with the user

9. Print the full JSON payload AND the `gh api` command that will be run. Use this format:
   ```
   I will apply this branch protection to {owner}/{repo}:{branch}:
   <indented payload>
   Command: gh api -X PUT repos/{owner}/{repo}/branches/{branch}/protection --input -
   Proceed?
   ```
10. WAIT for the user to confirm. Do not assume yes from ambiguous responses.

### Phase 4: Apply

11. Send the payload via `gh api -X PUT repos/{owner}/{repo}/branches/{branch}/protection --input <(echo '<payload>')`. On Windows use a temp file: write the payload to `%TEMP%\bp.json`, then `gh api -X PUT ... --input "$env:TEMP\bp.json"`.
12. Capture the response. IF 200: report success. IF 4xx: surface the GitHub error message verbatim; do not retry blindly.

### Phase 5: Verify

13. Run `gh api repos/{owner}/{repo}/branches/{branch}/protection` and confirm the response matches the intended state (required checks, approving review count, enforce_admins, no force-push).
14. Optionally suggest an unprotect/dry-run test: have the user push directly to the branch from a separate clone and observe the rejection. This is opt-in; do not do it automatically.

---

## Output Format

```yaml
repo: <owner>/<repo>
branch: <name>
variant: standard | solo-author | minimal
payload_applied:
  required_pull_request_reviews:
    required_approving_review_count: <n>
    dismiss_stale_reviews: <bool>
  required_status_checks:
    strict: <bool>
    contexts: [<job names>]
  enforce_admins: <bool>
  allow_force_pushes: { enabled: <bool> }
  allow_deletions: { enabled: <bool> }
verification: matches | drift_detected
follow_ups:
  - <e.g. "re-run after ci-gate-template merges to add required contexts">
```

---

## Examples

### Example 1: Solo-author Atlas repo, no CI yet

User: "Protect master on Atlas."

Action:
1. `gh auth status` — authenticated as the repo owner.
2. Default branch: `master`. Contributors: 1 (the owner). Recommend solo-author variant.
3. No `.github/workflows/` on master yet → `required_status_checks: null` for now.
4. Build payload from `references/protection-payload.json` solo-author template.
5. Print payload + command, ask for confirmation.
6. On confirm: apply via `gh api -X PUT`. Verify.
7. Follow-up: re-run after [[ci-gate-template]] lands a workflow.

### Example 2: Multi-author repo with CI

User: "Set up branch protection on Acme.Demo.Dashboard. We have a PR-checks workflow with jobs lint, typecheck, test."

Action:
1. Auth OK. Default branch: `main`. Contributors: 4. Use standard variant.
2. `required_status_checks.contexts: ["lint", "typecheck", "test"]`, `strict: true`.
3. `required_approving_review_count: 1`, `dismiss_stale_reviews: true`.
4. Confirm. Apply. Verify shows all three contexts and the review requirement.

### Example 3: BAD — applying without confirming

User: "Protect master."

WRONG: Skill immediately runs `gh api -X PUT ...` with a default payload. Result: required checks reference jobs that don't exist; user's PRs become unmergeable; user has to ask the skill to roll back.

RIGHT: Print payload first, wait for explicit "go ahead" / "yes" / "apply" from the user. Apply only after.

---

## Questions This Skill Answers

- "Protect main / master"
- "Require a PR before merging"
- "Set up branch protection on this repo"
- "Stop me from pushing directly to master"
- "Make merges go through review"
- "Require the CI to pass before merge"
- "Apply the GitHub branch protection rule"
- "I'm the only committer — how should I configure branch protection?"
- "Why are my PRs unmergeable after I added branch protection?"
- "Add required status checks to master"
- "Dismiss stale reviews when new commits are pushed"
- "Lock down the default branch"
- "Configure protected branches via gh CLI"
