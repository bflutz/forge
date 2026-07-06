---
name: code-review-checklist
description: |
  Runs the workspace's standard code review checklist against a project, covering React/TypeScript hygiene, Firebase security boundary (Firestore rules, App Check, callable functions), AI integration safety (Genkit, Claude API), security headers, and general repo hygiene. Also evaluates the quality of external audits or recommendation documents using the audit-receiving discipline checklist — distinguishing legitimate findings from templated noise.

  Use this skill when the user wants to audit a project, run a self-review, evaluate an external code review or pen-test report, check for the same issues found in another project, or asks "are we doing X here too?", "is this audit legit?", "review the codebase". Triggers on phrases like "review this codebase", "audit this project", "run the checklist", "evaluate this audit", "are these recommendations legit?", "check for the same issues".

  Do NOT use for active feature development, bug fixes, or implementing changes — those are normal coding work, not audits. Do NOT use for Claude Code configuration audits (skills, subagents, slash commands) — those have dedicated `audit-skill`, `audit-subagent`, `audit-slash-command` skills.
---

# Code Review Checklist

Executes the workspace's standard code review checklist against a project, or evaluates an external audit document for quality and substance. Sources the checklist from `_governance/code-review-checklist.md`.

The checklist has four tiers: React/TypeScript hygiene, Firebase/AI specifics, audit-receiving discipline, and general repo hygiene. Tier applicability depends on the project's stack — pick the right tiers before running, don't run all four blindly.

---

## Expert Vocabulary Payload

**Code Hygiene:** component decomposition, separation of concerns, line-count threshold, domain grouping, barrel exports, dead code detection, import discipline, feature-based structure
**Firebase Security:** Firestore Security Rules, Storage Rules, App Check enforcement, callable function auth context, IAM roles, Secret Manager, custom claims, security boundary, role-based access
**Performance:** bundle analysis, code splitting, lazy loading, render optimisation, memoisation tradeoffs, Core Web Vitals, performance budget, baseline measurement
**Audit Quality:** templated audit, evidence requirement, scope coverage, recommendation viability, baseline measurement, per-file findings, cross-verification
**AI Safety:** prompt injection, output validation, cost guardrails, rate limiting, system prompt isolation, persona stability

---

## Anti-Pattern Watchlist

### Blanket Acceptance

- **Detection:** Accepting every recommendation in an external audit without verifying claims against the actual repo. Symptom: "the audit said X, so we should do X."
- **Why it fails:** Templated audits routinely include findings that don't apply to the specific stack — `src/pages/` recommendations for App Router projects, "add CSP" when CSP already exists, "expose env vars" when Firebase web config is designed to be public. Acting on them wastes engineering time and can introduce regressions.
- **Resolution:** For each audit finding, locate the specific file or measurement that supports it before accepting. If the audit can't supply that, push back rather than implementing. Cross-check at least 3 random findings as a reliability sample.

### Checklist Theatre

- **Detection:** Running the checklist mechanically and ticking boxes without engaging with what each item means in context. Symptom: every item ticked, no recommendations produced, report is empty.
- **Why it fails:** The checklist's value is surfacing real issues, not in being completed. A clean run is suspicious — most projects have at least some Tier 1 issues.
- **Resolution:** For each item, capture EVIDENCE (file path, command output, line count) not just a tick. If a tier produces no findings, double-check by reading at least one randomly chosen file in scope and re-running one or two checks.

### Wrong-Tier Application

- **Detection:** Running Tier 2 (Firebase) on a project that doesn't use Firebase, or skipping Tier 2 on a project that does. Symptom: many "N/A" items or many "missing" items the project actually doesn't need.
- **Why it fails:** Tier applicability is stack-dependent. Running the wrong tiers either wastes time or misses real issues.
- **Resolution:** Use the Applicability Matrix in `_governance/code-review-checklist.md`. Identify the project's stack first (Vite / Next / CRA / Firebase / Genkit / etc.) before choosing tiers. Confirm scope with the user before starting.

### Performance Without Measurement

- **Detection:** Recommending memoisation, code-splitting, or bundle reduction targets without first running a bundle analysis or measuring render cost. Symptom: "this needs `useMemo`" with no profiler trace.
- **Why it fails:** Blanket React optimisations frequently make things slower (memo overhead exceeds re-render cost) and waste engineering effort. Real optimisation needs real measurement.
- **Resolution:** Before recommending performance changes, require a baseline: bundle-analyzer output, React DevTools profiler trace, or Web Vitals measurement. No measurement → no recommendation. Surface the missing baseline as the first action item.

### Confusing Public Config For Vulnerability

- **Detection:** Flagging Firebase `apiKey`, `authDomain`, or `projectId` in client bundles as a security finding.
- **Why it fails:** These values are designed to ship in the client bundle. The security boundary is Firestore Rules + App Check, not config secrecy. A finding here is a tell that the reviewer doesn't understand the Firebase model.
- **Resolution:** When this finding appears (in a self-review or external audit), reject it explicitly and redirect attention to the actual security boundary: `firestore.rules`, `storage.rules`, callable function auth, App Check status.

---

## Behavioral Instructions

### Phase 1: Identify project context

1. Read `CLAUDE.md` (project-level if present, otherwise workspace-level) to identify the stack.
2. Run a quick file tree survey:
   - `package.json` for dependencies and framework version
   - Presence of `firestore.rules`, `apphosting.yaml`, `next.config.*`, `vite.config.*`, `database.rules.json`
   - Presence of `functions/`, `src/ai/`, `src/firebase/`, `middleware.ts`
3. Determine which tiers apply by referencing the Applicability Matrix in `_governance/code-review-checklist.md`.
4. Confirm scope with the user before proceeding: "Running tiers [X, Y] on [project]. Continue?"

### Phase 2: Tier 1 — React / TypeScript hygiene

Skip if the project doesn't have a React or modern JS frontend.

5. **Component size pass.** Run line counts across component files. List any files > 300 lines. Identify top 5 by size. For each, classify: split-now / accept-with-reason / refactor-later.
6. **Folder structure pass.** List `src/lib/` or `src/utils/` contents. Identify domain mixing. Check whether components are grouped by feature or flat. Note dedicated `types/`, `services/`, `constants/` directories — present or absent.
7. **Loading / error pass.** Grep for async action patterns and loading state usage. Look for error boundary at root layout. Note any CRUD action without visible loading state.
8. **Form handling pass.** Identify form libraries in use. If forms use raw `useState`, list them and recommend a pattern (React Hook Form + Zod is the workspace default).
9. **Styling pass.** Grep for arbitrary Tailwind values: `bg-[`, `text-[`, `w-[`, `h-[`. Count occurrences and list files.
10. **Tests pass.** Locate test files. Run coverage if a script exists. Record current % vs target if any.

### Phase 3: Tier 2 — Firebase / Backend / AI

Skip if the project doesn't use Firebase, Cloud Functions, or AI integrations.

11. **Firestore rules.** Read `firestore.rules`. Check for fall-through wildcards, role helpers, per-collection rules. Verify deployment to staging AND prod.
12. **Storage rules.** Same checks for `storage.rules` if Storage is used.
13. **Cloud Functions.** For each callable in `functions/src/`: confirm auth check, role check where applicable, input validation. For each HTTP function: confirm CORS deliberate, not wildcarded.
14. **App Check.** Look for `app-check.ts` or equivalent initialisation. Status: enabled / in-progress / off-with-reason.
15. **Security headers.** Read `next.config.*` or hosting config. Verify CSP, X-Frame-Options, X-Content-Type-Options, Referrer-Policy, Permissions-Policy. Verify `X-XSS-Protection` is NOT set.
16. **AI flows.** Identify entry points (Genkit flows, Claude API calls, OpenAI calls). Check for input validation, output validation, cost guardrails, system prompt isolation.

### Phase 4: Tier 3 — External audit evaluation

Run only if reviewing an external audit document, not for self-audits.

17. **Document quality scan.** Product name correct throughout? Section uniqueness? Typos in headers? Coherent author/issuer metadata?
18. **Evidence scan.** For each finding, does it cite a specific file / line / measurement?
19. **Scope scan.** Did the audit cover the actual risk surface (rules, functions, AI, middleware) or only frontend?
20. **Recommendation viability.** Do recommendations match the actual stack? Any recommendations targeting frameworks not in use? Any duplicating work already done?
21. **Cross-check.** Pick 3 random findings and verify each against the repo. Record pass/fail. If 2+ fail, flag the audit's overall reliability as questionable.

### Phase 5: Tier 4 — General hygiene

22. Repository state, documentation, dependencies. See checklist for details.

### Phase 6: Produce report

23. Compile a markdown report (see Output Format below).
24. For each finding: location, evidence (cmd output / file:line), severity (high/medium/low), recommendation, rough effort estimate.
25. Present to user. WAIT for direction before any modification — this skill produces analysis, not changes.

---

## Output Format

```markdown
# Code Review Report: [Project Name]

**Date:** [ISO 8601]
**Tiers run:** [list]
**Mode:** self-audit | external-audit-review

## Summary

[3-5 sentence overview: top 3 findings by severity, overall posture, audit reliability if external]

## Tier 1 Findings

### [Short name] — severity: high / medium / low
- **Evidence:** [file:line / command output]
- **Issue:** [what it is, in one sentence]
- **Recommendation:** [what to do]
- **Effort:** [hours / days estimate]

[...]

## Tier 2 Findings

[same structure]

## Tier 3 Findings — External audit evaluation (if applicable)

### Audit document quality

- Product name: [correct / WRONG: shows X instead of Y]
- Duplicate sections: [none / list]
- Typos: [none / list]
- Author metadata: [coherent / issues]

### Per-finding verification (sample)

| Audit finding | Verified? | Evidence |
|---------------|-----------|----------|
| [...] | yes/no/partial | [file:line] |

### Scope gaps

[Areas the audit ignored that we'd expect them to cover]

### Recommendation conflicts

[Recommendations that don't match our actual stack or duplicate work already done]

## Tier 4 Findings

[same structure]

## Items deliberately accepted

[Things flagged by the checklist but accepted as OK in this context — record reason]

## Recommended next actions

[Top 3-5 items in priority order, suitable for converting into a TODO list]
```

---

## Examples

### Example 1: Internal self-audit

User: "Run the code-review-checklist on Acme Dashboard"

Action:
1. Read `Projects/acme-dashboard/CLAUDE.md` to confirm stack (React 19 + CRA + Firebase Realtime DB).
2. Determine tiers: 1, 2, 4. Note that Tier 2 maps to Realtime DB rules (`database.rules.json`), not Firestore.
3. Confirm scope with user.
4. Run line-count on `src/components/`, list components over 300 lines.
5. Read `database.rules.json` for security review (Realtime DB equivalent of firestore.rules).
6. Check for App Check, error boundaries, form libraries, bundle analysis.
7. Produce report with ranked findings.

### Example 2: External audit review

User: "Review this audit they sent us — does it actually apply to our codebase?"

Action:
1. Read the audit document.
2. Skip Phase 1-3 if project context is already known; jump to Phase 4 (Tier 3).
3. Document quality scan: check for templated content, wrong product names, duplicate sections, typos.
4. For each finding in the audit: verify against repo. Mark as legitimate / wrong-stack / outdated / out-of-scope.
5. Cross-check 3 random findings as a reliability sample.
6. Produce a parallel doc: their findings vs reality, plus what they didn't review that we'd expect.

### Example 3: Cross-project pattern check

User: "We just found component bloat in Atlas — are we doing the same in our other dashboard projects?"

Action:
1. For each named project: skip full audit, jump to the specific Tier 1 check (component size).
2. Run `wc -l` across each project's component directory.
3. Produce a comparison table: project, top 5 largest components, recommendation per project.
4. Don't expand scope to other tiers unless the user asks.

---

## References

- `<workspace>/_governance/code-review-checklist.md` — the source-of-truth checklist with all tier items and the applicability matrix (keep a workspace-local copy)
- `<workspace>/_governance/Workspace_Governance_SOP.md` — sibling governance document
- Origin: distilled from receiving and evaluating an external vendor audit, April 2026
