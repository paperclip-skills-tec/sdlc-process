---
name: sdlc-process
description: Paperclip AI SDLC Process — mandatory quality gates, coding standards, security checkpoints, and heartbeat practices that all agents must follow during software development work. Use this skill whenever you are implementing code, reviewing code, creating PRs, running tests, deploying, or working on any software engineering task.
---

# Paperclip AI SDLC Process

This skill defines the mandatory development practices for all Paperclip agents. Follow these rules during every software development task.

## Heartbeat-Aware Development Practices

These are mandatory for every agent on every heartbeat involving code:

1. **Checkpoint comments**: At the end of each heartbeat, post a comment summarising what was done, what decisions were made, and what is next.
2. **Atomic work units**: Leave the repository in a coherent state at the end of every heartbeat. No partial implementations that break tests, no uncommitted changes.
3. **Explicit dependency tracking**: If you discover a dependency not captured in `blockedByIssueIds`, update the issue before exiting. Do not proceed through a blocker.
4. **Context re-read**: At the start of each heartbeat, read the full issue context (description, latest comments, linked documents) before continuing work.
5. **No assumption of prior context**: Do not rely on memory from a previous run.

## Branching & Commits

- `main` is production-ready. No direct pushes. All changes via PR.
- Feature branches: `feature/TEC-NNN-short-description`
- Hotfix branches: `hotfix/TEC-NNN-short-description`
- Commit messages include the Paperclip issue identifier (e.g., `TEC-123: add user export endpoint`)
- PRs reference the Paperclip issue identifier in the title
- Squash-merge preferred for clean history
- Parallel development uses git worktrees for isolation

## Coding Standards

**General:**
- No credentials, secrets, or PII in version control, AI prompts, or shared logs
- External libraries assessed for licensing before adoption

**Backend (Node.js / Express):**
- All SQL uses parameterized queries — no string interpolation
- Input validation at all API boundaries
- Route handlers unit-tested with mocked service layer
- Service layer unit-tested with mocked DB layer

**Frontend (React / Vite):**
- Bootstrap 5 component conventions
- Accessible: semantic HTML, aria labels, keyboard-navigable
- No unsafe innerHTML without explicit DOMPurify sanitization
- Components tested with React Testing Library

**Database:**
- Migrations include both `up` and `down` scripts
- `down` script tested locally before PR submission
- No raw string interpolation in SQL

## Quality Gates

### G1: Planning Complete (before checkout)

- [ ] Acceptance criteria written and approved
- [ ] Complexity classification set (Low / Medium / High)
- [ ] Security threat assessment added (if touching auth, data, or external APIs)
- [ ] Dependencies set via `blockedByIssueIds`
- [ ] Large features decomposed into subtasks (<=3 heartbeats each)
- [ ] Database migration plan reviewed for reversibility

### G2: Implementation Ready (before code review)

- [ ] All acceptance criteria addressed (self-certify in issue comment)
- [ ] All unit tests passing
- [ ] Coverage: >=80% line, >=70% branch on new code
- [ ] `npm audit` clean: 0 high or critical vulnerabilities
- [ ] No ESLint errors
- [ ] `down` migration tested locally (if applicable)
- [ ] No secrets, credentials, or PII in code or commit history
- [ ] **Client–server payload-shape verified** (required for Enhanced / Security / Critical tiers; recommended for Standard): any new or modified client component that fetches a server endpoint (wizard, picker, list-fetch, table) has EITHER (a) a contract test that uses the real route handler OR a fixture captured from it, OR (b) a Playwright smoke that exercises the component against the running server. Synthetic hand-rolled row shapes alone do NOT satisfy this gate. Motivation: [TEC-1001](/TEC/issues/TEC-1001) PR #16 column-alias mismatch passed unit tests but failed end-to-end.
- [ ] Checkpoint comment posted
- [ ] PR description includes Paperclip issue link and AI tool disclosure

### G3: Review Approved (before integration testing)

- [ ] Code review checklist complete
- [ ] QA Engineer approval
- [ ] Dev Lead approval (for Enhanced, Security, or Critical tiers)
- [ ] Board review/approval (for Enhanced+ tiers)
- [ ] No unresolved blocking review comments
- [ ] CI passing

### G4: Testing Complete (before deployment)

- [ ] All integration tests green
- [ ] E2E smoke suite green on staging
- [ ] Security spot-check passed
- [ ] No open P0 or P1 bugs
- [ ] Swagger contract validated
- [ ] Database migration dry-run confirmed (if applicable)

### G5: Deployment Verified (before marking done)

- [ ] Post-deploy smoke suite green
- [ ] No errors in logs for 10 minutes post-deploy
- [ ] Rollback path confirmed working
- [ ] Board informed (for user-facing changes)

## Code Review Checklist

### BLOCKING — must pass before merge

**Correctness:**
- Logic matches acceptance criteria
- Edge cases handled (null/undefined, empty collections, boundary values)
- No silent failure — errors caught are logged or surfaced
- Async operations properly awaited; no unhandled promise rejections

**Security:**
- No raw string interpolation in SQL — parameterized queries only
- User input validated/sanitized at all API boundaries
- No sensitive data (passwords, tokens, PII) in logs or error responses
- Auth middleware applied to all protected routes
- No hardcoded credentials, API keys, or secrets
- `npm audit`: 0 high/critical vulnerabilities

**Test Quality:**
- Tests cover happy path and at least 2 meaningful edge cases per function
- Tests would actually fail if the implementation were wrong (no tautological tests)
- No `it.skip` or `xit` without a linked Paperclip issue
- Integration tests hit real database — not mocks

### Non-blocking

**Error Handling:** Specific `try/catch` handling; correct HTTP status codes; no stack traces in user-facing errors.

**Performance:** No N+1 queries; no sync blocking in Express handlers; pagination on unbounded lists.

**Code Quality:** No unused variables/imports/dead code; no copy-paste duplication; env vars for config.

**AI-Generated Code:** Package names verified against npm registry; no deprecated API usage; no redundant duplication.

## Review Tiers

| Tier | Change Type | Reviewer(s) | Board? |
|------|------------|-------------|--------|
| **Standard** | Bug fixes, test updates, docs, refactors | QA Engineer | No |
| **Enhanced** | New features, multi-module, new APIs, DB schema | QA + Dev Lead | Board review |
| **Security** | Auth, access control, crypto, credentials | QA + Dev Lead | Board approval |
| **Critical** | Architecture, new services, infra, breaking changes | QA + Dev Lead | Board approval + docs |

## Security — OWASP Top 10 Checkpoints

Apply on every PR touching auth, authorization, data access, file upload, or external APIs:

| Risk | Checkpoint | Blocking? |
|------|-----------|-----------|
| A01 Broken Access Control | Auth middleware on every protected route; RBAC server-side | Yes |
| A02 Crypto Failures | bcrypt cost >=12; HS256+ JWTs; no MD5/SHA1 | Yes |
| A03 Injection | All SQL parameterized; no `eval`; no `exec` with user data | Yes |
| A04 Insecure Design | Security reviewed at design phase | No |
| A05 Misconfig | No debug in prod; restrictive CORS; `helmet.js` | No |
| A06 Vulnerable Components | `npm audit` on every PR; 0 high/critical | Yes |
| A07 Auth Failures | Rate-limited login; session expiry; server-side logout | Yes |
| A08 Integrity | Lockfile committed; no `--ignore-scripts` | No |
| A09 Logging Failures | Auth events logged; no PII in logs | No |
| A10 SSRF | No external URLs from raw user input; allowlist | Yes |

## AI-Specific Requirements

- AI-generated code has ~40% security vulnerability rate and ~1.7x defect rate — all output requires review
- Package recommendations must be verified against npm registry (~20% may be hallucinated)
- Only approved tools for code generation
- Secrets in AI-generated code must be flagged and removed before commit
- AI tools may not process credentials, PII, or sensitive data in prompts
- Tests must be verified to actually fail when implementation is wrong (anti-tautological)
- All test imports verified to exist (anti-hallucination)

## Escalation Path

```
IC Agent -> Dev Lead -> Chief of Staff -> Board
```

- Set `blocked` status with blocker comment before escalating
- Never skip levels
- Board escalations require a recommendation
- Unresolved after 2 heartbeats? Escalate to next in chain

## Approval Tiers

| Tier | When | Approver |
|------|------|----------|
| **Agent-level** | Bug fixes, test updates, docs, refactors, research | Dev Lead / Chief of Staff / QA |
| **Board review** | New features, architecture, integrations, process, hiring, deploys | Board |
| **Board + docs** | Security changes, data model changes, breaking changes, new tech | Board + rationale + risk assessment |

## Bug Fix Rule

New bugs must have a failing test added *before* the fix is written.

## Continuous Improvement (raise, don't unilaterally change)

This SDLC process is itself owned by a standing goal: **SDLC Continuous Improvements** (`5ac7a221-90b0-4c5c-892e-4420c0548f58`, [TEC-964](/TEC/issues/TEC-964)). Every agent is expected to spot opportunities to improve the SDLC during normal work — flaky gates, missing checks, manual steps that could be automated, postmortem learnings, tooling gaps, security blind spots, escalation friction.

When you spot one:

1. Create an Issue titled `SDLC: <short description>`
2. Set `goalId` to `5ac7a221-90b0-4c5c-892e-4420c0548f58`
3. Body must include: **Current state · Proposed change · Expected benefit · Risk · Affected tier(s)**
4. Assign to the Board for approval
5. Do **not** modify the SDLC process or this skill before Board approval — the current version remains the baseline until the Board accepts the change

Improvements ship through this loop. Drive-by edits to gates, tiers, or checkpoints are out of scope.

## Work Quality & Outcomes (improve the work, not just the way we work)

The SDLC loop above improves **how** we work. A peer goal — **Work Quality & Outcomes** (`6aee171a-d451-48b1-aa3f-56e6a3c2bdd9`) — exists to improve **what** we deliver: meeting briefs, executive summaries, action trackers, knowledge artefacts, agent prompts/skills, recommendation quality, time-to-action.

Both goals sit under the company-level **Continuous Improvement** umbrella (`65a80caa-bbc7-4c67-9a86-d4da1d511b14`, board-approved 2026-04-28 via [TEC-981](/TEC/issues/TEC-981)).

When you spot a deliverable that could be better:

1. Create an Issue titled `WORK: <short description>`
2. Set `goalId` to `6aee171a-d451-48b1-aa3f-56e6a3c2bdd9`
3. Body must include: **Current output · Observed gap · Proposed change · Expected user benefit · Affected deliverable(s)**
4. Assign to the Board for approval (`assigneeUserId: "local-board"`, `assigneeAgentId: null`)
5. Do **not** ship the change before Board approval — same governance bar as SDLC

Use `SDLC:` for process/tooling/checkpoint changes; use `WORK:` for output-quality changes. If unsure which applies, default to the one whose body template fits the gap better.

---

*Source: Paperclip AI SDLC Process v2.0 ([TEC-792]). Full document in Obsidian vault at `02_Reference/AI SDLC Process.md`.*
