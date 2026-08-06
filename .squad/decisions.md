# Squad Decisions

> Entries older than 30 days archived to `decisions-archive.md` (2026-08-05).

## Workflow & Process

### Workflow Simplification Directive (2026-05-13)
- **Decision:** Effective immediately for v1.7.0 and beyond. Streamline agent session workflow to reduce context overhead and accelerate routine iteration while maintaining safety and quality gates.
- **Rationale:** Multi-agent dispatch, 3-model gates, Comment Triage Loops, and Reviewer Rejection Lockout proved high-overhead for routine work. Core safety gates (CI green, fail-first tests, security invariants) are sufficient and more durable than analytical critique loops.
- **Changes:**
  - **Dropped:** Multi-agent dispatch for routine solo work, 3-model gates per PR, Comment Triage Loop on every Copilot thread, Reviewer Rejection Lockout for routine work.
  - **Kept (non-negotiable):** CI green as merge gate, fail-first regression tests for every bug fix, security invariants (Remove-Credentials, Schema, HTTPS-only, host allow-list, 300s timeout), Pester baseline enforcement.
  - **Duck only when stuck or before high-blast-radius changes:** Swift haiku (not Opus extra-high), brief focused reasoning, document blockers in PR sticky comment + mirror squad issue.
  - **Bug detection preference:** Fail-first regression tests over analytical critique loops. Tests are durable, critiques ephemeral.
- **Status:** Active (approved by martinopedal)

## Active Decisions

### Canonical Entity IDsin Test Fixtures (2026-04-18)
- **Decision:** Wrapper and normalizer fixtures must use canonical entity ID shapes expected by `ConvertTo-CanonicalEntityId` (`Subscription` as bare GUID, `Repository` as `host/owner/repo`).
- **Rationale:** Strict `New-FindingRow` validation now enforces canonical IDs, and non-canonical fixture data causes false-negative unit test failures unrelated to wrapper behavior.
- **Implementation:** Updated fixtures/tests for azure-cost, defender-for-cloud, gitleaks, scorecard, trivy, plus subscription-ID handling in Azure Cost/Defender normalizers.
- **Status:** Active

### CI Failure Watchdog Automation (2026-04-17)
- **Decision:** Implement CI failure triage as a dedicated `workflow_run` watchdog plus an opt-in local PowerShell watcher that share the same dedup contract using hash: first 12 chars of `sha256("{workflow}|{first-error-line}")`.
- **Rationale:** Converts failed runs into actionable backlog items. Prevents issue spam by grouping repeats by deterministic workflow+error signature. Keeps behavior consistent between GitHub-hosted and local polling loops.
- **Security:** Self-trigger loops blocked with workflow-name exclusion. Error lines sanitized before issue generation. Workflow payload values passed through environment variables.
- **Implementation:** `.github/workflows/ci-failure-watchdog.yml`, `tools/Watch-GithubActions.ps1`, tests, docs updated.
- **Status:** Active

### Issue #127 Fix: CI Failure Watchdog Event Registration (2026-04-17)
- **Decision:** Fixed `.github/workflows/ci-failure-watchdog.yml` by adding missing `workflows:` key to `workflow_run` trigger.
- **Root Cause:** `workflow_run` event payload does not include `head_branch`; referencing it in job condition caused parse-time workflow failure preventing job initialization.
- **Chosen Fix:** Minimal and safe - applied job condition `if: github.event.workflow_run.conclusion == 'failure' && github.event.workflow_run.name != 'CI failure watchdog'` with proper trigger registration.
- **Validation:** 2 post-merge live runs returned `conclusion=success` with `event=workflow_run`.
- **PR:** #154 (SHA 0f287ad)
- **Status:** Active

### Error Sanitization Boundary (2026-04-18)
- **Decision:** Sanitize at error-capture time (in `catch` blocks), not write-time. Every exception message assigned to a `Message` property must wrap with `Remove-Credentials`.
- **Rationale:** Single boundary enforcement prevents bypasses. If we sanitized only at write-time, future developers might write new output paths and forget. Keeps error messages in result objects always safe.
- **Pattern:** `$result.Message = "Context: $(Remove-Credentials $_.Exception.Message)"`
- **Enforcement:** Grep audit for `Exception.Message|Error.Message|.Message`, Pester tests (6 scenarios: SAS URI, bearer token, connection string, GitHub PAT, null, multi-secret), CI gate (398/398 tests).
- **Status:** Active

### PR #116 Re-Gate Extension (2026-04-18)
- **Decision:** Apply Falco-established dot-source pattern to all missing runspace boundaries. Dot-source `shared/Sanitize.ps1` in parallel-runspace callsites. Add dot-source + fallback stub inside `Invoke-AzureAnalyzer.ps1` before invoking wrappers.
- **Rationale:** PowerShell 7 `ForEach-Object -Parallel` creates isolated runspaces where parent-scope functions are not inherited. Guarantees `Remove-Credentials` exists at runtime in each worker runspace.
- **Validation:** `Invoke-Pester -Path .\tests -CI` → 398 passed, 0 failed.
- **Status:** Active

### PR #120 Revision for Issue #126 Gate (2026-04-17)
- **Decision:** Applied five corrections to wrapper error paths: (1) parser safety via `"${testNumber}: $testDesc"` string formatting; (2) test hard-fail with `$ErrorActionPreference = 'Stop'`; (3) retry API alignment with `-MaxAttempts` canonical (backward-compat `-MaxRetries` mapped); (4) sanitization invariant on all 17 wrappers; (5) `PartialSuccess` semantics for multi-target scans with mixed success/failure.
- **Rationale:** Prevents parse regressions, improves test signal, reduces secret leakage, preserves findings during partial outages.
- **Status:** Active

### PR Review Gate Model Selection (2026-04-17)
- **Decision:** For PR review-gate triage, use three diverse models: `claude-opus-4.6`, `gpt-5.3-codex`, `goldeneye`. All 3 must approve before merge.
- **Rationale:** Single-model review under-captures edge cases. Trio gives overlap on core correctness while preserving disagreement signal. Avoids homogeneous failure modes.
- **Operating Rule:** Ingest PR reviews, generate model-specific prompt bundle, merge three responses into deterministic consensus. Read + comment + plan-write only (no auto-approve/dismiss).
- **Status:** Active

### Issue-First Workflow Directive (2026-04-18)
- **Decision:** Do not ship ad-hoc PRs; always work on issues first. When an issue is fully planned (acceptance criteria, design, scope confirmed), then create code and implement the issue plan. PRs reference and implement issue plans, never the other way around.
- **Rationale:** Consistency across the squad. The issue is the contract; the PR is the implementation. Forces planning before code.
- **Status:** Active

### Rubberduck-Gate in Required Checks (2026-04-21)
- **Decision:** Branch protection enforces BOTH `Analyze (actions)` AND `rubberduck-gate` (strict=true), not just `Analyze (actions)`.
- **Impact:** With strict=true, each PR merge invalidates downstream PRs in a batch. Requires `gh pr update-branch` + ~90s CI wait per subsequent merge.
- **Operational Guidance:** Run Dependabot batches sequentially, not in parallel. Update coordinator runbook to reflect dual-gate requirement.
- **Discovery:** Found during Dependabot batch #288-#292 processing (2026-04-21).
- **Status:** Active

### Upload-Artifact v7 Matrix Safety Pattern (2026-04-21)
- **Decision:** `actions/upload-artifact@v7` is safe in this repo because (a) zero `download-artifact` consumers, (b) both upload sites use unique artifact names (sbom-{sha}, scheduled-scan-{run_id}).
- **Future Watchpoint:** Any new matrix consumer of `actions/upload-artifact@v7` MUST suffix artifact name with matrix variable (v5+ no longer merges same-named artifacts across matrix legs).
- **Pattern Example:** `name: sbom-${{ matrix.os }}-${{ github.sha }}` instead of `name: sbom`
- **Status:** Documented

### GitHub-Script v9 Compatibility Pattern (2026-04-21)
- **Decision:** `actions/github-script@v9` removed `require('@actions/github')`. The `getOctokit` function is now injected as a parameter, not defined via `const` or `let`.
- **Incompatible Pattern:** `const getOctokit = require('@actions/github').getOctokit;` or redeclaring with `const getOctokit = ...`
- **Compatible Pattern:** Use the injected `getOctokit` directly: `const octokit = getOctokit();`
- **Audit Result:** Zero instances of incompatible patterns found in 9 inline-script consumers. Safe to deploy.
- **Status:** Documented

### Dependabot Stale Version Comment Quirk (2026-04-21)
- **Decision:** Dependabot sometimes bumps the action SHA to a newer version but leaves the version comment tag at the previous release.
- **Example:** PR #290 bumped codeql-action to SHA matching v4.35.2 but left comment as `v4.35.1`. PR #291 bumped action-gh-release to v3.0.0 SHA but left comment as `v2.2.0`.
- **Mitigation:** Always `git diff` before merging Dependabot PRs. Fix stale comments with a follow-up commit on the Dependabot branch (before merge) to maintain SHA-pin policy accuracy.
- **Status:** Documented

---

## 2026-08-05: Windows Runners & Finding Suppression

### 2026-08-05 - Move windows-latest to GitHub-hosted runners

**Context:** The self-hosted `public-win` VMSS pool had no registered runner since ~2026-06-02. The `runs-on` dispatch expression routed `windows-latest` jobs to `["self-hosted","public-win"]`, causing all Windows CI jobs to queue indefinitely and get cancelled for two consecutive months.
**Decision:** Move `windows-latest` to GitHub-hosted runners (falling through to `|| matrix.os`). The repo is public, so GitHub-hosted Windows minutes are free and unlimited. The `public-linux` ACA pool stays for Linux jobs.
**Consequences:** PR #1250 merged at `e09ad611`. Issue #1173 closed. `docs/operations/runners.md` updated. First green Windows CI validation in two months.

### 2026-08-05 - Finding-key scheme and marked-not-dropped suppression policy

**Context:** Issue #1229 requested native false-positive / accepted-risk suppression. The key requirement was identifying findings stably across re-scans.
**Decision:**
- Key = SHA-256 over `source|rule|entity`, first 16 hex chars, lowercased and trimmed.
- Entry forms in suppression file: machine-generated `key` or human-readable `source` + `ruleId` + `entityId` triple.
- Suppressed findings remain in `results.json` (stamped with `FindingKey`, `Suppressed`, `SuppressionReason`). Only severity counts and default report views exclude them, displaying a visible suppression count.
- Every finding block in `Invoke-AzureAnalyzer.ps1` (including correlators) MUST stamp `FindingKey`, `Suppressed`, and `SuppressionReason` to ensure StrictMode safety under `Set-StrictMode -Version Latest`.
**Consequences:** PR #1251 merged at `4704a3c`. Issue #1229 closed. `modules/shared/Suppression.ps1` shipped.

## Governance

- All meaningful changes require team consensus
- Document architectural decisions here
- Keep history focused on work, decisions focused on direction


## 2026-08-06: CI Restoration, Haflidi Completion, and Platform Learnings

### 2026-08-06 - Fork-PR guards for privileged CI workflows

**Context:** Three CI workflows failed on every external-contributor fork PR due to GitHub's fork-PR permission model. Fork PRs receive a read-only GITHUB_TOKEN and all repository secrets resolve to empty strings.
**Decision:** Apply job-level if: fork guards to pr-auto-rebase.yml, issue-resolution-verify.yml, and pr-auto-rerun-on-push.yml. For pull_request-only workflows: if: github.event.pull_request.head.repo.full_name == github.repository. For mixed-trigger workflows: negative-short-circuit pattern passing non-PR events unconditionally. continue-on-error: true was explicitly rejected — it papers over hard failures and hides real errors.
**Consequences:** PR #1253 merged. Issues #1240, #1242, #1243, #1249 closed.
**Author:** Forge

### 2026-08-06 - Runspace pool provisioning approach (#1225)

**Context:** Worker runspaces created ad-hoc (ForEach-Object -Parallel) do not reliably inherit modules loaded by the orchestrator, causing silent finding drops (#1218). The serial fallback (AZURE_ANALYZER_MAX_PARALLEL=1) was the guaranteed-correct workaround at the cost of parallelism.
**Decision:** Opt-in parallel via InitialSessionState StartupScripts. New-WorkerSessionState builds a default session and adds every modules/shared/*.ps1 via StartupScripts. Activation: -UseRunspacePool switch or $env:AZURE_ANALYZER_USE_RUNSPACE_POOL=1. Serial remains the default. Flip to parallel default when: (1) one production tenant scan succeeds with the pool at scale, (2) Windows CI green for 5+ consecutive runs with pool active, (3) no finding-loss regressions.
**Consequences:** PR #1255 merged. Issue #1225 closed. modules/shared/WorkerPool.ps1 shipped.
**Author:** Forge

### 2026-08-06 - Interactive report design (#1230)

**Context:** Issue #1230 requested an interactive HTML report layer with FP-marking and suppression export.
**Decision:** Opt-in via -Interactive / -InteractiveReport switch; static output is byte-identical when absent (enforced by 	ests/samples/SampleDrift.Tests.ps1). FindingKey (16-hex SHA-256) is the only client-side identity, ensuring export round-trips directly into Import-SuppressionList. Export format: {"key":"...","reason":"..."} per entry with schemaVersion: "1.0". FP filtering via CSS classes on html.fp-hide/html.fp-only (toggles document.documentElement.classList). data-fk attribute holds the FindingKey as the hook for #1231.
**Consequences:** PR #1252 merged. Issue #1230 closed. docs/consumer/interactive-report.md added.
**Author:** Sentinel

### 2026-08-06 - Interactive report payload shrink (#1231)

**Context:** Issue #1231 requested shrinking the interactive HTML report payload via stable hashed IDs and string deduplication.
**Decision:** Five intern tables (rule, entity, subscription, tool, status) replace verbatim string attributes with integer indices on each TR row. window._T lookup table emitted once before the IIFE. Post-generation regex patches the IIFE (.dataset.rule → _dv(r,'rule')) only when -Interactive is true. Virtual DOM paging deferred — string interning delivers the primary saving. data-severity and data-fk intentionally not interned (negligible savings / inherently unique).
**Consequences:** PR #1257 in flight at session end. Issue #1231 targeted.
**Author:** Sentinel

### 2026-08-06 - Batch tool pin bumps into one PR per run

**Context:** 	ools/Update-ToolPins.ps1 was hardcoded to open one PR per tool, producing 48 open PRs collapsing to 16 unique tools. git merge-tree empirically proved adjacent rows in the generated docs/reference/tool-catalog-contributor.md hard-conflict; conflicted PRs run zero pull_request workflows.
**Decision:** Refactor Update-ToolPins.ps1 to produce one batched PR per run on chore/bump-tool-pins-<yyyyMMdd>. Phase 1: collect all pin changes. Phase 2: single branch/commit/PR. All three doc generators run once after all manifest writes. 31 duplicate PRs and 16 superseded per-tool PRs were closed.
**Consequences:** PR #1254 merged. Old per-tool branch name pattern retired.
**Author:** Forge

### 2026-08-06 - Zero registered self-hosted runners was the true CI root cause

**Context:** gh api repos/martinopedal/azure-analyzer/actions/runners returned 	otal_count: 0 while 24 of 26 workflows targeted [self-hosted, public-linux]. This is a user account, not an org; repo-level runners are the only possible source. The repo is public so GitHub-hosted runners are free and unlimited.
**Decision:** Migrate all 26 workflow files to GitHub-hosted runners (PR #1256, merged 697652ef). Acceptance gate going forward: Select-String -Path "C:\git\azure-analyzer\.github\workflows\*.yml" -Pattern "self-hosted|public-linux|public-win" MUST return nothing. Never reintroduce self-hosted runner references without first verifying active runners exist via gh api repos/.../actions/runners.
**Consequences:** All 47 queued workflow runs unblocked. Three blocked squad PRs (#1252, #1254, #1255) could merge.
**Author:** Squad (Coordinator)

### 2026-08-06 - Tool-pin PRs are now batched, not one-per-tool

**Context:** Per root cause analysis (see batch tool pin decision above). git merge-tree proved: rows within 1 line of each other hard-conflict; gap >= 2 merges clean. A conflicted PR runs zero pull_request workflows, making it a permanently dead PR.
**Decision:** Generator rewritten in PR #1254 to emit a single batched PR per run. 31 duplicate PRs and 16 superseded per-tool PRs were closed. The chore/bump- prefix is preserved for closes-link-required.yml exemption.
**Status:** Active — Update-ToolPins.ps1 is the authoritative implementation.
**Author:** Squad (Coordinator)

### 2026-08-06 - Closes #N does NOT fire when added to a PR body after merge

**Context:** PR #1250 contained Closes #1173 and merged cleanly, yet #1173 stayed OPEN because the keyword was added post-merge. GitHub only processes Closes keywords present at merge time.
**Decision:** RULE: after every merge, explicitly verify the linked issue closed with gh issue view <n> --json state. Never assume a Closes keyword retroactively applied.
**Status:** Active
**Author:** Squad (Coordinator)

### 2026-08-06 - Issues can land with zero labels, making them invisible to Ralph

**Context:** Issues #1225, #1229, and #1230 arrived with no labels despite uto-label-issues being configured to add squad on open. Ralph's --label squad scan silently skipped them. Open question: why did uto-label-issues not fire?
**Decision:** Workaround: gh issue edit <n> --add-label squad. Note: gh pr edit --add-label fails on ead:org scope error; issue edit works. Investigate uto-label-issues trigger failures separately.
**Status:** Active (root cause unresolved)
**Author:** Squad (Coordinator)

### 2026-08-06 - Set-Location does not change [Environment]::CurrentDirectory

**Context:** .NET APIs ([IO.File]::ReadAllText/WriteAllText, Select-String -Path) resolve relative paths against the PROCESS working directory, not the PowerShell location set by Set-Location. This caused ~5 rounds of phantom inconsistency, silently wrote conflict fixes into the wrong worktree, and truncated Invoke-AzureAnalyzer.ps1 and New-HtmlReport.ps1 to 1 line each (repaired via git checkout --).
**Decision:** MANDATORY: always use absolute paths with [IO.File] and Select-String -Path. All Scribe-class file operations must start with C:\git\azure-analyzer\.
**Status:** Active — enforced as squad invariant
**Author:** Squad (Coordinator)

### 2026-08-06 - CHANGELOG.md is the universal merge-conflict point

**Context:** Every feature PR inserts a bullet immediately after ### Added or ### Fixed, guaranteeing an adjacent-insert conflict against any other merged PR.
**Decision:** Resolution recipe: read into System.Collections.Generic.List[string], verify marker layout by index, build second list with .Add([string]\) per element, then RemoveRange/InsertRange. WARNING: passing @(...) directly to InsertRange throws because System.Object[] will not convert to IEnumerable[string]. CRITICAL: CHANGELOG.md line ~529 contains a literal <<<<<<< ours as documented prose — never blanket-strip conflict markers; match (?m)^<<<<<<< HEAD|^>>>>>>>  specifically.
**Status:** Active
**Author:** Squad (Coordinator)

### 2026-08-06 - Platform model catalog changed; fallback chains stale

**Context:** claude-sonnet-4.5 and claude-haiku-4.5 no longer exist and spawns against them fail hard. gemini-3.6-flash failed silently for Scribe-class work — returned an empty turn and wrote zero files (confirmed by filesystem check).
**Decision:** Use claude-sonnet-4.6 for code work (4 successful agents this session). Do not use gemini-3.6-flash for file-writing agents. Update squad fallback chains to remove claude-sonnet-4.5 and claude-haiku-4.5. Frontier Fallback Chain: claude-opus-4.7 → claude-opus-4.6-1m → gpt-5.4 → gpt-5.3-codex → goldeneye.
**Status:** Active — fallback chains require update
**Author:** Squad (Coordinator)

### 2026-08-06 - Path-filtered required check reporting equired=[] is expected

**Context:** Analyze (actions) (CodeQL, language: [actions]) only runs when .github/workflows/** changes. Pure-PowerShell PRs legitimately show 8 checks instead of 25-26 and report no required check. Branch protection treats a path-filtered skip as satisfied.
**Decision:** Do NOT block a merge on this. Related: CI rollup must GROUP BY check name and take the newest per name — GET /commits/{sha}/check-runs returns superseded runs too; a cancelled-then-rerun check appears twice and naive counting reports phantom failures.
**Status:** Active
**Author:** Squad (Coordinator)