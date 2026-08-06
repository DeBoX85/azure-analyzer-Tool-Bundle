# Project Context

- **Owner:** martinopedal
- **Project:** azure-analyzer - Security Analyst & Recommendation Engine
- **Stack:** PowerShell, Pester, JSON, Security Scanners (Gitleaks, Trivy, Zizmor, Scorecard, Maester)
- **Created:** 2026-04-15

## Core Context

- **Security Invariants:** All outbound calls HTTPS-only. Host allow-list enforced for clones. Output sanitized via Remove-Credentials. 300s process execution timeout.
- **Pester & StrictMode:** Under Set-StrictMode -Version Latest, accessing non-existent properties on PSCustomObject or Hashtable throws PropertyNotFoundException. Always stamp expected fields (FindingKey, Suppressed, SuppressionReason) across all code paths.
- **Pester 5 Lifecycle Rules:** All lifecycle blocks (BeforeAll, AfterEach, etc.) MUST be inside Describe blocks - never at script root.
- **Test Isolation:** Reset $LASTEXITCODE in inally or AfterAll when testing non-zero exits to avoid leaking into downstream tests.

## Recent History

### 2026-05-13 - v1.7.2 Validation Audit
- Exercised 5 execution modes (subscription, tenant, repository, ADO, direct wrapper). Verified tool execution produces real esults.json + ntities.json with Schema 3.1.
- Scanned 44 fake-success patterns (0 matches). Direct wrapper invocation verified.
- Added LiveTool.StateIsolation.Tests.ps1 regression guard.

### 2026-08-05 - Native False-Positive Suppression List (#1229)
- **Suppression Key:** SHA-256 over source|rule|entity, first 16 hex chars, lowercased and trimmed. FindingRow.Id was rejected because 37 normalizers fallback to random GUIDs.
- **Marked, Not Dropped:** Suppressed findings remain in esults.json. Only severity counts and default report views exclude them, showing a visible suppressed count.
- **StrictMode Fix:** Fixed production bug at Invoke-AzureAnalyzer.ps1:1633 where correlator findings block omitted FindingKey/Suppressed/SuppressionReason, throwing PropertyNotFoundException under Set-StrictMode -Version Latest.
- **Files Shipped:** modules/shared/Suppression.ps1 (356 lines), 	ests/shared/Suppression.Tests.ps1 (26 tests), docs/consumer/suppression-list.md. PR #1251 merged at 4704a3c. Issue #1229 closed. Credited @haflidif in CHANGELOG.

### 2026-08-05 - Team Update
- Windows CI restored on GitHub-hosted runners (PR #1250 / #1173).
- Native false-positive suppression list shipped (PR #1251 / #1229).
- Backlog of 48 tool-pin PRs deduped down to 16 keepers.
### 2026-08-05 - Interactive Single-File HTML Report (#1230)

**Design:** Opt-in alongside the static renderer. When `-Interactive` is not set, output is byte-identical to the static renderer -- enforced by `tests/samples/SampleDrift.Tests.ps1`.

**Identity:** `FindingKey` (SHA-256 from #1229) is the ONLY client-side identity for marking. No second identity scheme. The JSON export uses `{"key":"...","reason":"..."}`, directly consumable by `Import-SuppressionList`.

**Payload shape:** `data-fk` attribute per `tr.row` element. Empty string when not interactive, so the attribute does not appear in static output. #1231 can use this as its dedup hook without redesign.

**FP filtering:** CSS-only via `html.fp-hide` and `html.fp-only` classes toggled on `document.documentElement`. Adjacent sibling selector `tr.row.fp-marked + tr.expand` hides expand rows correctly without touching the existing JS filter.

**Count updates:** ORIG severity counts are server-baked as a JS constant. Live adjusted counts are computed from `data-fk` + `fpState` in the browser; no server round-trip.

**Export round trip:** Export suppression JSON -> drop next to scan -> `-SuppressionFile`. The exported file is valid `Import-SuppressionList` input immediately. Reason auto-includes UTC timestamp.

**Gotchas hit:**
- The `create` tool writes files with literal `\n` (two-char backslash-n) rather than actual LF newlines on Windows. Files created via `create` must be rewritten with `[IO.File]::WriteAllText()` to get real newlines.
- CRLF flip detection: `git diff --numstat` showing a large diff on a small change is the signal. Fix: `[IO.File]::ReadAllText` + `-replace "\r\n","\n"` + `WriteAllText`.
- The `$html` here-string injection points: CSS before `</style>`, toolbar after Export CSV button, thead before `</tr></thead>`, footer script before `</body>`. All use empty-string gating so non-interactive path produces byte-identical output.
- PS double-quoted here-strings with JS: safe when JS has no `$` chars. Optional chaining `?.` is literal in PS strings.

**Files shipped:** New-HtmlReport.ps1 (+Interactive switch), Invoke-AzureAnalyzer.ps1 (+InteractiveReport switch), tests/samples/InteractiveReport.Tests.ps1 (16 tests), docs/consumer/interactive-report.md. PR #1252. Issue #1230. Credited @haflidif in CHANGELOG.


### Issue #1231 - Interactive Report Payload Shrink (string interning)

**Branch**: `feat/1231-payload-shrink` | **PR**: #1257

**Design decisions made:**
1. String interning via 5 lookup tables (rule, entity, sub, tool, status). Each TR row carries integer indices instead of verbatim strings.
2. Preamble script `window._T` emitted before the main IIFE; `_dv(el, key)` resolver function covers all consumers.
3. Post-generation regex patch on the IIFE: `.dataset.rule` etc. replaced with `_dv(r,'rule')` after `` is generated. This avoids restructuring the large static JS block.
4. Virtual DOM paging: evaluated, deferred. Too risky to combine with interning in one PR. Left as open improvement.
5. data-severity (short constant) and data-id/data-fk (unique per row) are not interned.

**Gotchas:**
- The `edit` and `create` tools produce literal `\n` sequences (ASCII 92+110) not real LF on Windows. The working fix: restore from git via cmd redirect (`git cat-file blob HEAD:file > file`), strip CRLF with `[IO.File]::WriteAllText(f, (t -replace CR+LF, LF))`, then apply changes using PowerShell `.Replace()` with `[char]10` for newlines and `[char]39` for single quotes in replacement strings.
- The `@'...'@` here-string approach works reliably when embedded in the `command` parameter as real newlines (JSON newlines in the tool call), but breaks when nested inside the resulting PS session if the original string contains tricky quote combinations.
- The `-replace` regex replacement: use PS single-quoted replacement string `'_dv(,'''')'` where `''` is an escaped single quote, and .NET regex expands ``/`` as capture groups.
- The `` heredoc is built AFTER the `if ()` block that populates the intern tables, so the lookup table preamble CAN be built in that block and interpolated into the heredoc correctly.

**Tests added**: 7 new payload shrink tests in `InteractiveReport.Tests.ps1` (23 total, all pass). SampleDrift 1/1 pass.