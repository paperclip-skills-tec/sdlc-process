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

* `main` is production-ready. No direct pushes. All changes via PR.
* Feature branches: `feature/TEC-NNN-short-description`
* Hotfix branches: `hotfix/TEC-NNN-short-description`
* Commit messages include the Paperclip issue identifier (e.g., `TEC-123: add user export endpoint`)
* PRs reference the Paperclip issue identifier in the title
* Squash-merge preferred for clean history
* Parallel development uses git worktrees for isolation
* Edits to **company skills** (anything under `skills/<skill-name>/` in the hosted company-skills repo) follow the dedicated **Company Skill Repository Workflow** below — feature branch + PR, never a local fast-forward to `main`.

### Shared working-tree protocol

Working trees are per-agent by default. When two agents edit the same checkout in the same heartbeat, one agent's `git status` snapshot can be invalidated by the other's branch switch, stash, or commit between commands — and a `git push` then lands on the wrong branch. Motivation: [TEC-1065](/TEC/issues/TEC-1065) — interim skill-level mitigation for the [TEC-1044](/TEC/issues/TEC-1044) incident, where a branch swap between `git status` and `git commit` landed unrelated work on `fix/tec-997-bootstrap-icons-font` and contaminated PR #26 on `origin`.

1. **One agent per working tree per heartbeat.** This is the default assumption for every git command in this skill. If you are not sure another agent is using the same checkout, assume they are not — but verify before committing (rule 3).
2. **Per-issue worktree when the checkout is shared.** When `PAPERCLIP_EXECUTION_WORKSPACE_ID` is **null** AND `git status` shows another agent's branch checked out (i.e. the current branch is not the one you intend to work on, or the working tree contains uncommitted changes you did not stage), you MUST add a per-issue worktree before any code edit:
   ```bash
   git worktree add -b <branch-name> ~/.paperclip/worktrees/<issue-identifier>/ <base-ref>
   cd ~/.paperclip/worktrees/<issue-identifier>/
   ```
   Use the `superpowers:using-git-worktrees` skill for the full mechanics. The worktree path is keyed on the Paperclip issue identifier (e.g. `~/.paperclip/worktrees/TEC-1068/`) so concurrent issues never collide.
3. **Branch-identity check around every commit and push.** Shared-checkout branch swaps are silent. Before `git commit` AND before `git push`, capture the current branch and abort if it has moved since you last ran `git status`:
   ```bash
   EXPECTED_BRANCH="skills/sdlc-process/shared-working-tree-protocol"  # whatever you intended
   ACTUAL_BRANCH="$(git rev-parse --abbrev-ref HEAD)"
   if [ "$ACTUAL_BRANCH" != "$EXPECTED_BRANCH" ]; then
     echo "ABORT: HEAD is on $ACTUAL_BRANCH, expected $EXPECTED_BRANCH" >&2
     exit 1
   fi
   ```
   Run this immediately before each mutating command, not once at the start of the heartbeat. Treat a mismatch as a hard stop — do not "fix" it by checking out the expected branch, because that would also throw away whatever the other agent staged.

### Cross-agent contamination recovery

Use this protocol when the shared-working-tree checks above fail late — i.e. a commit has already landed on a branch you did not intend, or a push has already contaminated `origin`. Motivation: same [TEC-1065](/TEC/issues/TEC-1065) / [TEC-1044](/TEC/issues/TEC-1044) incident; recovery is expected to take the rest of a heartbeat, so report blocked early rather than improvising.

1. **Detect.** Compare commit history on the branch you intended to work on with the branch the commit actually landed on:
   ```bash
   git log <expected-branch> --oneline -10
   git log <actual-target>  --oneline -10
   ```
   If the SHA referencing your issue identifier appears on `<actual-target>` but not on `<expected-branch>`, you have a misrouted commit. If the misrouted SHA is also on `origin/<actual-target>`, you have an `origin` contamination as well — both must be addressed.
2. **Cherry-pick onto the intended branch.** Bring the work onto the branch it should have been on, then push:
   ```bash
   git switch <expected-branch>          # or: git worktree add -b <expected-branch> ~/.paperclip/worktrees/<issue-identifier>/ <base-ref>
   git cherry-pick <sha>                 # the misrouted commit's SHA
   git push origin <expected-branch>
   ```
   Resolve cherry-pick conflicts the normal way; do not skip the cherry-pick to "save time" — the audit trail must show the work landed on the right branch.
3. **Revert the contamination on the wrong branch.** If the misrouted commit was pushed to `origin/<actual-target>`, revert it there so the affected PR's diff returns to its intended state:
   ```bash
   git fetch origin                       # ensure local <actual-target> isn't stale before reverting
   git switch <actual-target>             # or: git worktree add ~/.paperclip/worktrees/<issue-identifier>-revert/ <actual-target>
   git revert <sha> --no-edit
   git push origin <actual-target>
   ```
   The worktree alternative mirrors step 2's pattern — if the shared checkout you're recovering in is the one that was just contaminated, switching branches inside it re-enters the failure mode the protocol is recovering from. The `git fetch origin` defends against the parallel-push race: a stale local `<actual-target>` would otherwise non-fast-forward (or worse, push back over a concurrent commit). Do not force-push to delete history on a shared branch — revert.
4. **Disclose on the affected PR's source issue.** If a stray commit reached `origin/<other-branch>`, post a comment on the Paperclip issue that owns that branch's PR, describing:Disclosure is mandatory even if the revert returned the branch to its prior state — the reviewer needs to know the history is non-linear.
   * the contaminating SHA and what it actually contained,
   * which branch/PR it contaminated and which branch it now lives on,
   * the revert SHA on the contaminated branch,
   * and `[@reviewer]` mentions for every reviewer required by that PR's review tier — skill PRs: Dev Lead + QA Engineer (per [Company Skill Repository Workflow](#company-skill-repository-workflow)); application PRs: per the [Review Tiers](#review-tiers) table — so they re-check the diff before merging.
5. **Stash hygiene.** Stashes are global to the repo, not per-branch, so a stash created by another agent will appear in your `git stash list`. Never apply a stash by label alone:
   ```bash
   git stash list                        # see all entries, including ones you did not create
   git stash show -p stash@{N}           # inspect the diff before doing anything with it
   ```
   Only `git stash pop` / `git stash apply` after the diff matches what you expect. If a stash label looks like yours but the diff does not, treat it as another agent's work and leave it alone.

## Company Skill Repository Workflow

This applies whenever an agent edits a company skill in the hosted skills repo (one repo per company, each skill as a `skills/<skill-name>/` subdirectory — see [TEC-1039](/TEC/issues/TEC-1039#document-plan)). It does **not** apply to product/application repos, which use the standard branching rules above.

Skills are high-leverage prompt code that every agent in the company depends on. A bad skill diff silently degrades every subsequent heartbeat. The workflow below trades a few minutes of review latency for a durable audit trail and human checkpoint.

### Hard rules

1. **Never fast-forward `main` locally.** Even though the local clone has a writable `main`, all changes ship via PR on the hosted remote.
2. **One skill change per branch / per PR.** Don't bundle unrelated skill edits — review surface stays small, rollback stays cheap.
3. **Branch name:** `skills/<skill-name>/<short-description>` — e.g. `skills/sdlc-process/add-pr-workflow-section`. The skill name in the branch matches the directory name. Include the Paperclip issue identifier in the commit message and PR title (not necessarily the branch).
4. **Two-reviewer rule:** every skill PR requires **Dev Lead + QA Engineer** approval before merge, regardless of size. Branch protection on `main` enforces 1 review at the GitHub level; the second is a process expectation (and is what `CODEOWNERS` will eventually automate). The detailed reviewer checklist lives in the QA-owned skill-PR review doc — see [TEC-1048](/TEC/issues/TEC-1048).
5. **No secrets in commits or PR descriptions.** Skill bodies are prompt text — every reviewer reads them. Treat the diff as if it were already public.

### GitHub credential contract

Agents do not push using their own credentials. A user-supplied GitHub Personal Access Token, scoped to the company-skills repo, is injected into the heartbeat environment per run, mirroring the `PAPERCLIP_API_KEY` pattern. The PAT carries the supplying user's GitHub identity, so its commits are authored by that user — no separate bot login or email override is required.

| Env var               | Source                                                        | Scope                                                                     | Purpose                                                                                                   |
| --------------------- | ------------------------------------------------------------- | ------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `GITHUB_SKILLS_TOKEN` | User-supplied GitHub PAT, injected by Paperclip per heartbeat | `repo:write` on the company-skills repo only (no other Deltek orgs/repos) | `git push` and `gh` authentication for skill PRs; the PAT also supplies the GitHub commit author identity |
| `PAPERCLIP_AGENT_ID`  | Already injected (existing)                                   | n/a                                                                       | Used in the per-agent commit trailer for attribution                                                      |

If `GITHUB_SKILLS_TOKEN` is **absent**, the workflow is not available in your environment yet — do not fall back to your own personal token. Comment on the issue with `blocked: GITHUB_SKILLS_TOKEN not injected for this run` and PATCH to `blocked` so WS5 owners can finish provisioning.

The token is **runtime-only**. Do not write it to disk, do not pass it on command lines (use `gh auth login --with-token` via stdin, or `GH_TOKEN` in the environment), and do not echo it in comments or logs.

### Commit attribution

Every skill commit must carry both trailers, exactly as written:

```
Co-Authored-By: Paperclip <noreply@paperclip.ing>
X-Paperclip-Agent-Id: <agent-id>
```

* `Co-Authored-By` is the existing Paperclip rule (see the `paperclip` skill) and surfaces the Paperclip-system identity in the GitHub UI alongside the human PAT-holder who appears in the commit author field.
* `X-Paperclip-Agent-Id` is the per-agent attribution trailer. Use the literal value of `$PAPERCLIP_AGENT_ID` — this is what lets GitHub commit data join Paperclip's run audit later (every Paperclip API mutation already carries `X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID`; the same agent id appears on the run record). This trailer is required even though the GitHub commit author is the PAT-holder — the author field identifies *whose* PAT pushed, not *which* agent ran.
* Both trailers go at the **end** of the commit message, separated from the body by a blank line, in the order shown above.

### End-to-end worked example

Scenario: agent `a30415ac-99a8-485b-a49b-146c0d9d8407` (Full-Stack Developer) edits the `sdlc-process` skill under issue `TEC-1050`.

```bash
# 0. Confirm the GitHub PAT is present this heartbeat.
test -n "$GITHUB_SKILLS_TOKEN" || { echo "GITHUB_SKILLS_TOKEN not injected; cannot push"; exit 1; }

# 1. From a fresh checkout / worktree of the company-skills repo, branch off main.
cd "$COMPANY_SKILLS_REPO"
git fetch origin
git switch -c skills/sdlc-process/add-pr-workflow-section origin/main

# 2. Make edits to skills/sdlc-process/SKILL.md (or other files under skills/<skill>/).
$EDITOR skills/sdlc-process/SKILL.md

# 3. Stage and commit. The PAT carries its own GitHub author identity — no
#    git config user.name / user.email override is needed. Trailers go at the
#    END of the message, separated from the body by a blank line.
git add skills/sdlc-process/SKILL.md
git commit -m "$(cat <<EOF
TEC-1050: document push/PR workflow for company skill edits

- New "Company Skill Repository Workflow" section
- GITHUB_SKILLS_TOKEN env var contract
- Per-agent commit trailer

Co-Authored-By: Paperclip <noreply@paperclip.ing>
X-Paperclip-Agent-Id: ${PAPERCLIP_AGENT_ID}
EOF
)"

# 4. Authenticate gh with the PAT (stdin only — never on the command line).
printf '%s' "$GITHUB_SKILLS_TOKEN" | gh auth login --with-token

# 5. Push the branch.
git push -u origin skills/sdlc-process/add-pr-workflow-section

# 6. Open the PR. Title carries the Paperclip identifier; reviewers are Dev Lead + QA.
gh pr create \
  --base main \
  --head skills/sdlc-process/add-pr-workflow-section \
  --title "TEC-1050: document push/PR workflow for company skill edits" \
  --reviewer "@deltek/dev-lead,@deltek/qa-engineer" \
  --body "$(cat <<'EOF'
## Summary
Adds the Company Skill Repository Workflow section to `skills/sdlc-process/SKILL.md`, defines the `GITHUB_SKILLS_TOKEN` env var contract, and documents the per-agent commit trailer.

## Paperclip
- Issue: [TEC-1050](/TEC/issues/TEC-1050)
- Parent: [TEC-1039](/TEC/issues/TEC-1039#document-plan)

## AI tool disclosure
Drafted by Full-Stack Developer (Claude). Reviewed against the parent plan before push.

## Reviewers
- Dev Lead (architecture / consistency with parent plan)
- QA (per the WS4 skill-PR review checklist — [TEC-1048](/TEC/issues/TEC-1048))
EOF
)"

# gh prints the PR URL on success — paste it into the Paperclip issue handoff comment.
```

After `gh pr create` returns, the PR URL is the handoff artifact. The IC's heartbeat does **not** merge the PR — that's a reviewer action gated on the two approvals.

### Local-only repo carve-out (transition period)

Until the company-skills repo is provisioned and `GITHUB_SKILLS_TOKEN` is being injected (WS5), the company-skills repo is local-only. During the transition, follow the **local-only repo carve-out** in the [Pre-handoff branch + commit verification](#pre-handoff-branch--commit-verification) on-fail table: commit your work to a feature branch, leave a checkpoint note that the repo is local-only, cite the local commit SHA in the handoff, and proceed. Do not silently push to a personal remote.

Once `GITHUB_SKILLS_TOKEN` is being injected, the carve-out no longer applies — every skill edit must ship via PR.

### What this section does **not** cover

* The actual repo provisioning, branch protection rules, and bot account setup → tracked under **WS5** of [TEC-1039](/TEC/issues/TEC-1039#document-plan).
* The reviewer checklist (frontmatter sanity, secret-leak scan, cross-skill consistency, etc.) → tracked under **WS4** ([TEC-1048](/TEC/issues/TEC-1048)).
* The CI workflow that runs on every skill PR → tracked under **WS3** ([TEC-1051](/TEC/issues/TEC-1051)).
* Migrating existing local skill repos into the hosted repo with preserved history → **WS1** ([TEC-1049](/TEC/issues/TEC-1049)).

## Coding Standards

**General:**

* No credentials, secrets, or PII in version control, AI prompts, or shared logs
* External libraries assessed for licensing before adoption

**Backend (Node.js / Express):**

* All SQL uses parameterized queries — no string interpolation
* Input validation at all API boundaries
* Route handlers unit-tested with mocked service layer
* Service layer unit-tested with mocked DB layer

**Frontend (React / Vite):**

* Bootstrap 5 component conventions
* Accessible: semantic HTML, aria labels, keyboard-navigable
* No unsafe innerHTML without explicit DOMPurify sanitization
* Components tested with React Testing Library

**Database:**

* Migrations include both `up` and `down` scripts
* `down` script tested locally before PR submission
* No raw string interpolation in SQL

## Quality Gates

### G1: Planning Complete (before checkout)

* [ ] Acceptance criteria written and approved
* [ ] Complexity classification set (Low / Medium / High)
* [ ] Security threat assessment added (if touching auth, data, or external APIs)
* [ ] Dependencies set via `blockedByIssueIds`
* [ ] Large features decomposed into subtasks (\<\=3 heartbeats each)
* [ ] Database migration plan reviewed for reversibility

### G2: Implementation Ready (before code review)

* [ ] All acceptance criteria addressed (self-certify in issue comment)
* [ ] All unit tests passing
* [ ] Coverage: >\=80% line, >\=70% branch on new code
* [ ] `npm audit` clean: 0 high or critical vulnerabilities
* [ ] No ESLint errors
* [ ] `down` migration tested locally (if applicable)
* [ ] No secrets, credentials, or PII in code or commit history
* [ ] **Client–server payload-shape verified** (required for Enhanced / Security / Critical tiers; recommended for Standard): any new or modified client component that fetches a server endpoint (wizard, picker, list-fetch, table) has EITHER (a) a contract test that uses the real route handler OR a fixture captured from it, OR (b) a Playwright smoke that exercises the component against the running server. Synthetic hand-rolled row shapes alone do NOT satisfy this gate. Motivation: [TEC-1001](/TEC/issues/TEC-1001) PR #16 column-alias mismatch passed unit tests but failed end-to-end.
* [ ] **Pre-handoff branch + commit verification passed** (required for any code-implementation issue before PATCHing `in_progress → in_review`). Run the bash check in the **"Pre-handoff branch + commit verification"** section below; a failure means do NOT patch to `in_review`. Motivation: [TEC-1019](/TEC/issues/TEC-1019) — recurring failure where ICs handed off without committed work, costing QA a full review heartbeat on phantom changes.
* [ ] Checkpoint comment posted
* [ ] PR description includes Paperclip issue link and AI tool disclosure

### G3: Review Approved (before integration testing)

* [ ] Code review checklist complete
* [ ] QA Engineer approval
* [ ] Dev Lead approval (for Enhanced, Security, or Critical tiers)
* [ ] Board review/approval (for Enhanced+ tiers)
* [ ] No unresolved blocking review comments
* [ ] CI passing

### G4: Testing Complete (before deployment)

* [ ] All integration tests green
* [ ] E2E smoke suite green on staging
* [ ] Security spot-check passed
* [ ] No open P0 or P1 bugs
* [ ] Swagger contract validated
* [ ] Database migration dry-run confirmed (if applicable)

### G5: Deployment Verified (before marking done)

* [ ] Post-deploy smoke suite green
* [ ] No errors in logs for 10 minutes post-deploy
* [ ] Rollback path confirmed working
* [ ] Board informed (for user-facing changes)

### G6: Release Complete (before closing a release issue)

* [ ] All PRs for this release merged to `main` with G3 approval
* [ ] Annotated tag created on `main` and pushed to `origin`
* [ ] GitHub Release created from that tag (a tag without a GitHub Release is **incomplete**)
* [ ] Release notes complete per template (Changes, Migrations, Breaking Changes, Known Issues)
* [ ] GitHub Release URL posted to the Paperclip release issue
* [ ] Board informed

## Release Management

A tag on `origin` without a matching GitHub Release is not a valid release — it is invisible to the team. This section defines the mandatory process.

### When to cut a release

- Board decision (sprint completion, milestone delivery)
- P1/P2 hotfix that must reach production

### Semantic versioning

`vMAJOR.MINOR.PATCH` — increment MAJOR for breaking changes (Board confirmation required), MINOR for new backwards-compatible features, PATCH for bug fixes. Pre-release: `vX.Y.Z-rc.N`.

### Pre-release checklist (Dev Lead confirms before tagging)

- [ ] All feature branches for this release merged to `main`
- [ ] All merged PRs have passed G3 (Review Approved) — no unreviewed or changes-requested PRs
- [ ] CI green on `main` (tests, coverage, `npm audit`)
- [ ] E2E smoke suite green on staging
- [ ] No open P0/P1 bugs
- [ ] Release notes drafted
- [ ] Board Release Gate approval obtained

### Release steps

```bash
# 1. Confirm all PRs merged + approved (pre-release checklist above)

# 2. Create annotated tag on main
git tag -a vX.Y.Z -m "Release vX.Y.Z — [one-line summary]"
git push origin vX.Y.Z

# 3. Create GitHub Release (mandatory — not optional)
gh release create vX.Y.Z \
  --title "vX.Y.Z — [one-line summary]" \
  --notes-file release-notes.md

# 4. Post the GitHub Release URL to the Paperclip release issue
```

### Release notes template

```markdown
## vX.Y.Z — YYYY-MM-DD

### Summary
[1–2 sentences]

### Changes
- [TEC-NNN] Feature: [description]
- [TEC-NNN] Fix: [description]

### Database Migrations
[List migrations with run order, or "None."]

### Breaking Changes
[MAJOR bumps only, or "None."]

### Known Issues
[Any known issues not resolved, or "None."]
```

### Emergency hotfix release

* Branch: `hotfix/TEC-NNN-short-description` off `main`
* Expedited review: ≤2 hours P1, ≤4 hours P2
* PATCH version increment
* GitHub Release created same day — no exceptions
* Full release notes required even for single-line hotfixes
* Board informed immediately (post-facto Release Gate acceptable for P1)

## Pre-handoff branch + commit verification

**Mandatory** before an IC PATCHes a code-implementation issue from `in_progress` to `in_review`. Catches the failure mode described in [TEC-1019](/TEC/issues/TEC-1019), where ICs handed off issues without their work being committed/pushed — so QA woke up, pulled, and found nothing on the named branch. This is the **consumer-side** version of the gate (per the [TEC-1022 plan](/TEC/issues/TEC-1022#document-plan), Surface 1).

### When to run

Run before the PATCH that moves the issue to `in_review`, in the same active execution workspace where the work was done.

### The check

Copy-run this bash block, replacing `<cwd>` with the workspace path (or omit the `-C <cwd>` if you are already inside the working tree) and `<issue-identifier>` with the actual issue identifier (e.g. `TEC-1029`):

```bash
IDENT="<issue-identifier>"
NEEDLE="$(printf '%s' "$IDENT" | tr '[:upper:]' '[:lower:]')"

# 1. Branch present locally or on origin?
MATCHED_REF="$(git -C <cwd> for-each-ref \
  --format='%(refname)' refs/heads/ refs/remotes/origin/ \
  | grep -iE "(^|[^0-9])${NEEDLE}([^0-9]|\$)" \
  | head -n1)"

if [ -z "$MATCHED_REF" ]; then
  echo "FAIL: no branch references ${IDENT}"
  exit 1
fi

# 2. At least one commit on the matching branch references the identifier?
if ! git -C <cwd> log "$MATCHED_REF" --grep="${IDENT}" -i --pretty=%H -n1 \
     | grep -q .; then
  echo "FAIL: branch ${MATCHED_REF} found but no commit references ${IDENT}"
  exit 1
fi

echo "PASS: ${IDENT} verified on ${MATCHED_REF}"
```

**Match rule:** non-digit boundary regex on the lowercased identifier. This is intentional: `tec-1019` does NOT match `tec-10199`. Substring matches across digit boundaries would cause false positives on near-numbered issues.

### On fail — do NOT PATCH to `in_review`

| Failure                                                                                                                                                      | What it means                                                                                                                                                                                                                                                                                    | Required action                                                                                                                                                                                                                                                                                                         |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `no branch references ${IDENT}` and the workspace **exists**                                                                                                 | Work isn't yet committed onto a branch named for this issue. **Local-only repo carve-out:** if this repository is intentionally local-only (no `origin` / no push target) and a local commit in the active working tree already references `${IDENT}`, this failure is non-blocking for handoff. | Default path: commit the work onto a branch named for this issue (e.g. `feature/TEC-NNN-short-description`), push it, then re-run the check. **Carve-out path (local-only repo):** add a checkpoint note that the repo is local-only and cite the local commit SHA referencing `${IDENT}` before proceeding to handoff. |
| `branch ${REF} found but no commit references ${IDENT}` and the workspace **exists**                                                                         | Branch exists but no commit message references the identifier.                                                                                                                                                                                                                                   | Either amend / add a commit that references the identifier in the message, push, then re-run; or, if commits exist but the message lacks the identifier, add a follow-up commit (e.g. `TEC-NNN: include identifier in handoff trail`).                                                                                  |
| Any failure and the **workspace was lost mid-flight** (matches the [TEC-1005](/TEC/issues/TEC-1005) failure mode — execution disappeared, working tree gone) | Recovery requires recreating the workspace and recommitting.                                                                                                                                                                                                                                     | PATCH the issue to `blocked` with the unblock action: `"execution workspace lost; recreate workspace and recommit"`. Do NOT silently re-PATCH to `in_review`.                                                                                                                                                           |

The skill must NOT instruct you to PATCH `in_review` when this check fails. Treat a failed check as the same class of blocker as a failing test.

### Bypass cases (do NOT run the check on these)

The check must not false-positive on legitimate hand-offs that don't have a code working tree. Skip when ANY of the following apply:

* **Plan-only or research issues** — no working tree was opened, no code change is expected. The hand-off is a plan document or comment, not a branch.
* **Tasks routed back to a user/board** for review (not to QA Engineer). User/board reviewers don't pull the branch the way QA does, so the check protects no downstream cost.
* **Doc-only edits made directly without a workspace** — e.g. an Obsidian-vault edit, an issue-document edit, or a comment-only deliverable. There is no branch by design.
* **The issue's deliverable is a Paperclip artifact** (an approval, a comment with structured payload, an issue document) and the issue description does not call for code changes.

If you are unsure whether a bypass applies, default to running the check. A false-fail is cheaper than a missed handoff.

## Issue Lifecycle Closeout

Status transitions carry downstream consequences — wakes, reviewer assignments, Watchdog scans. Use the checklists and payloads below to transition correctly.

### Transitioning to `in_review`

Use when handing off completed work for review. The reviewer target depends on issue shape.

**Checklist:**

* [ ] All acceptance criteria self-certified in a comment
* [ ] Pre-handoff branch + commit verification passed (code issues only — see [above](#pre-handoff-branch--commit-verification))
* [ ] Checkpoint comment posted with summary of changes
* [ ] PR opened and linked (code issues only)

**Routing precedence:**

1. **Explicit `Hand-off:` in the issue description** — follow it exactly.
2. **Child task** (`parentId` set) — assign to QA Engineer (`d158fe7b-f2d2-48d1-ba20-18ea21d94d06`), or Dev Lead (`5a301e20-07a8-4a34-b113-4818437c02ac`) if QA is not appropriate.
3. **Top-level issue** (no `parentId`, `createdByUserId` set) — assign to Dev Lead and let them route. Never assign a top-level issue directly to `local-board` from IC handoff.

**Payload:**

```bash
scripts/paperclip-issue-update.sh --issue-id "$PAPERCLIP_TASK_ID" --status in_review <<'MD'
## Ready for review

- [summary of what was done]
- PR: [link]
MD
```

Or via curl:

```json
PATCH /api/issues/{issueId}
Headers: X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID
{
  "status": "in_review",
  "assigneeAgentId": "<reviewer-agent-id>",
  "comment": "## Ready for review\n\n- [summary]\n- PR: [link]"
}
```

### Transitioning to `done`

Use when the issue is fully complete with no follow-up needed.

**Checklist:**

* [ ] All acceptance criteria met
* [ ] Completion comment posted (see [TEC-1119](#tec-1119-completion-comment) requirements below)
* [ ] No unresolved blocking review comments
* [ ] Work product committed / merged / delivered

**Payload:**

```bash
scripts/paperclip-issue-update.sh --issue-id "$PAPERCLIP_TASK_ID" --status done <<'MD'
## Done

- [what was completed]
- [outcome or artifact link]
MD
```

Or via curl:

```json
PATCH /api/issues/{issueId}
Headers: X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID
{
  "status": "done",
  "comment": "## Done\n\n- [what was completed]\n- [outcome or artifact link]"
}
```

### Transitioning to `blocked`

Use when the issue cannot proceed until a specific external action occurs.

**Checklist:**

* [ ] Blocker is specific: name the unblock owner and the exact action needed
* [ ] Use `blockedByIssueIds` when another Paperclip issue is the blocker — do not rely on free-text alone
* [ ] For external blockers (missing env var, pending approval, infrastructure), include owner, expected resolution date, and unblock condition in the comment
* [ ] **Dedup check**: if your most recent comment on this issue was a blocked-status update and no one has replied since, do NOT re-comment — skip the issue entirely and only re-engage when new context arrives (comment, status change, event wake)

**Payload:**

```json
PATCH /api/issues/{issueId}
Headers: X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID
{
  "status": "blocked",
  "blockedByIssueIds": ["<blocker-issue-id>"],
  "comment": "Blocked: [description]\n\n- **Owner:** [who must act]\n- **Action:** [what they must do]\n- **Expected:** [when / condition for unblock]"
}
```

### TEC-1119 Completion Comment {#tec-1119-completion-comment}

Before your session exits on a `done` issue, you must post a final comment containing structured completion output. The Paperclip Watchdog scans for this — issues marked `done` without a valid completion comment are auto-flagged. See [TEC-1119](/TEC/issues/TEC-1119).

**A valid completion comment must include one of:**

* A heading line: `## Done`, `## Summary`, `## Completed`, `## Result`, `## Output`, or `## Finished`
* A status line starting with: `Done`, `Completed`, or `Finished`

**Must NOT contain blocker language:** `blocked by`, `waiting on`, `unresolved`, `cannot proceed`, or similar phrases that contradict the `done` status.

**Example:**

```markdown
## Done

- Implemented feature X; tests pass
- PR opened and linked
```

### Blocked-Task Dedup Rule

When waking on a `blocked` issue, check the comment thread before taking action. If **all** of these are true, skip the issue — do not checkout, do not re-comment:

1. Your most recent comment was a blocked-status update
2. No one has replied since that comment
3. No new event (status change, blocker resolution) triggered this wake

Only re-engage when new context arrives: a reply, a `issue_blockers_resolved` wake, or a status change by another agent.

## Code Review Checklist

### BLOCKING — must pass before merge

**Correctness:**

* Logic matches acceptance criteria
* Edge cases handled (null/undefined, empty collections, boundary values)
* No silent failure — errors caught are logged or surfaced
* Async operations properly awaited; no unhandled promise rejections

**Security:**

* No raw string interpolation in SQL — parameterized queries only
* User input validated/sanitized at all API boundaries
* No sensitive data (passwords, tokens, PII) in logs or error responses
* Auth middleware applied to all protected routes
* No hardcoded credentials, API keys, or secrets
* `npm audit`: 0 high/critical vulnerabilities

**Test Quality:**

* Tests cover happy path and at least 2 meaningful edge cases per function
* Tests would actually fail if the implementation were wrong (no tautological tests)
* No `it.skip` or `xit` without a linked Paperclip issue
* Integration tests hit real database — not mocks

### Non-blocking

**Error Handling:** Specific `try/catch` handling; correct HTTP status codes; no stack traces in user-facing errors.

**Performance:** No N+1 queries; no sync blocking in Express handlers; pagination on unbounded lists.

**Code Quality:** No unused variables/imports/dead code; no copy-paste duplication; env vars for config.

**AI-Generated Code:** Package names verified against npm registry; no deprecated API usage; no redundant duplication.

## Review Tiers

| Tier         | Change Type                                                                      | Reviewer(s)   | Board?                                                  |
| ------------ | -------------------------------------------------------------------------------- | ------------- | ------------------------------------------------------- |
| **Standard** | Bug fixes, test updates, docs, refactors                                         | QA Engineer   | No                                                      |
| **Enhanced** | New features, multi-module, new APIs, DB schema                                  | QA + Dev Lead | Board review                                            |
| **Security** | Auth, access control, crypto, credentials                                        | QA + Dev Lead | Board approval                                          |
| **Critical** | Architecture, new services, infra, breaking changes                              | QA + Dev Lead | Board approval + docs                                   |
| **Skill PR** | Any edit to a company skill (`skills/<skill-name>/…` in the company-skills repo) | QA + Dev Lead | No (unless the skill itself is Security/Critical scope) |

**Skill PRs always require both Dev Lead and QA Engineer approval**, regardless of diff size — see [Company Skill Repository Workflow](#company-skill-repository-workflow). Skills run on every heartbeat, so the blast radius of a bad merge is global.

## Security — OWASP Top 10 Checkpoints

Apply on every PR touching auth, authorization, data access, file upload, or external APIs:

| Risk                      | Checkpoint                                                 | Blocking? |
| ------------------------- | ---------------------------------------------------------- | --------- |
| A01 Broken Access Control | Auth middleware on every protected route; RBAC server-side | Yes       |
| A02 Crypto Failures       | bcrypt cost >\=12; HS256+ JWTs; no MD5/SHA1                | Yes       |
| A03 Injection             | All SQL parameterized; no `eval`; no `exec` with user data | Yes       |
| A04 Insecure Design       | Security reviewed at design phase                          | No        |
| A05 Misconfig             | No debug in prod; restrictive CORS; `helmet.js`            | No        |
| A06 Vulnerable Components | `npm audit` on every PR; 0 high/critical                   | Yes       |
| A07 Auth Failures         | Rate-limited login; session expiry; server-side logout     | Yes       |
| A08 Integrity             | Lockfile committed; no `--ignore-scripts`                  | No        |
| A09 Logging Failures      | Auth events logged; no PII in logs                         | No        |
| A10 SSRF                  | No external URLs from raw user input; allowlist            | Yes       |

## AI-Specific Requirements

* AI-generated code has \~40% security vulnerability rate and \~1.7x defect rate — all output requires review
* Package recommendations must be verified against npm registry (\~20% may be hallucinated)
* Only approved tools for code generation
* Secrets in AI-generated code must be flagged and removed before commit
* AI tools may not process credentials, PII, or sensitive data in prompts
* Tests must be verified to actually fail when implementation is wrong (anti-tautological)
* All test imports verified to exist (anti-hallucination)

## Escalation Path

```
IC Agent -> Dev Lead -> Chief of Staff -> Board
```

* Set `blocked` status with blocker comment before escalating
* Never skip levels
* Board escalations require a recommendation
* Unresolved after 2 heartbeats? Escalate to next in chain

## Approval Tiers

| Tier             | When                                                               | Approver                            |
| ---------------- | ------------------------------------------------------------------ | ----------------------------------- |
| **Agent-level**  | Bug fixes, test updates, docs, refactors, research                 | Dev Lead / Chief of Staff / QA      |
| **Board review** | New features, architecture, integrations, process, hiring, deploys | Board                               |
| **Board + docs** | Security changes, data model changes, breaking changes, new tech   | Board + rationale + risk assessment |

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

***

*Source: Paperclip AI SDLC Process v2.1 ([TEC-792](/TEC/issues/TEC-792)). Release Management added in [TEC-1718](/TEC/issues/TEC-1718). Full document in Obsidian vault at `02_Reference/AI & Strategy/AI SDLC Process.md`.*

---

*TEC Custom Skill — maintained by the Deltek Technical Services Engineering team.*