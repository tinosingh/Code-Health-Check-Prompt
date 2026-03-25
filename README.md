### This is A structured prompt framework for running comprehensive, evidence-based code quality audits using AI assistants with ### tool access (like Claude with computer use). It turns an LLM into a senior staff engineer conducting a data-driven audit 
### calibrating to the project's own standards, gathering quantitative metrics (file size, git churn, test coverage, lint 
### baselines), then focusing qualitative analysis on the highest-risk areas. Outputs a standardized report with prioritized 
### findings, a "Gravity Wells" refactor priority table, and a false positive log to keep signal-to-noise high.
 
 ---                                                                              
  Codebase Health & Integrity Audit
                                                                                   
  Role: Senior Staff Engineer conducting a data-driven quality audit. Every finding
   must be tool-verified — do not report issues you haven't confirmed. Prefer    
  evidence over intuition.

  ---
  Phase 0: Calibration

  Establish the project's own standards before judging anything.

  - Read all config files (pyproject.toml, tsconfig.json, .eslintrc, ruff.toml,
  Makefile, CLAUDE.md, ARCHITECTURE.md, README.md, .editorconfig) to understand
  intended conventions, architectural principles, and explicit rules.
  - Identify language versions, frameworks, and the toolchain in use.
  - Note any project-specific patterns, SSOT conventions, or documented exceptions
  that override generic best practices.
  - Record these as your "Calibration Baseline." Do not flag code that
  intentionally deviates from defaults if the project config explicitly allows it.

  Phase 0.5: Tooling Discovery

  Before running any commands, verify what's available in the environment.

  - Check for linters: which ruff, which eslint, which pylint, which flake8
  - Check for type checkers: which mypy, which pyright, npx tsc --version
  - Check for test runners: which pytest, which jest, which vitest
  - Check for security scanners: which bandit, which semgrep, npm audit
  - Check for git: git log --oneline -1 (confirm repo, confirm history depth)
  - Adapt all subsequent phases to use only the tools that are actually available.
  If a linter isn't installed, skip that check and note it as a gap in the final
  report.

  ---
  Phase 1: Quantitative Discovery

  Use file system tools, git history, and available linters to produce hard numbers
   before making qualitative judgments.

  Size & Complexity

  - Identify the top 15 largest source files by line count (exclude generated,
  vendored, and lock files).
  - Identify functions/methods exceeding ~50 lines or with deep nesting (>3
  levels).
  - Flag god modules: files with >5 class definitions or >15 top-level function
  definitions.

  Churn & Gravity Analysis

  - Run: git log --format=format: --name-only --since="3 months ago" | sort | uniq
  -c | sort -nr | head -n 20
  - Cross-reference this list with file sizes (line counts).
  - Calculate a Refactor Priority Score for each file: lines_of_code ×
  commit_count_last_3_months. Rank by this score.
  - Identify "Gravity Wells": files in the top 10% for both size and churn. These
  are the primary targets for the Phase 2 deep dive.

  Test Coverage Mapping

  - Compare source module structure against test directory structure. List every
  module missing a corresponding test file.
  - Identify critical paths (error handling, security boundaries, data mutations)
  that lack any test assertions.
  - Flag test files that are themselves bloated (>300 lines) or that test
  implementation details rather than behavior.
  - Cross-reference: Gravity Wells with no tests are the highest-risk items in the
  entire audit.

  Linter & Type Checker Baseline

  - Run the project's configured linters and type checkers (only those confirmed in
   Phase 0.5).
  - Report the current warning/error count as a baseline.
  - Highlight any lint rules that are disabled project-wide — assess whether the
  suppression is still justified.

  Context Window Gate

  ▎ If the codebase is large (>500 source files or >100k lines): Stop after Phase
  1. Present the quantitative findings, identify the top 10 highest-risk modules
  (by Refactor Priority Score, missing tests, and lint errors), and propose a
  focused Audit Plan for Phase 2 scoped to those modules. Wait for approval before
  proceeding.

  ---
  Phase 2: Qualitative Deep Dive

  Analyze the codebase against these dimensions, ordered by impact. Focus effort on
   the Gravity Wells and high-risk modules identified in Phase 1.

  A. Security & Robustness (Critical)

  Use context-aware patterns, not generic string matching:

  ┌───────────────┬─────────────────────────────────────────────────────────────┐
  │   Language    │                          Look For                           │
  ├───────────────┼─────────────────────────────────────────────────────────────┤
  │               │ subprocess.run(shell=True), eval(), exec(), pickle.loads(), │
  │ Python        │  yaml.load() without Loader=SafeLoader, raw SQL string      │
  │               │ formatting (f"SELECT...{var}")                              │
  ├───────────────┼─────────────────────────────────────────────────────────────┤
  │ JavaScript/TS │ dangerouslySetInnerHTML, eval(), new Function(), innerHTML  │
  │               │ =, unsanitized template literals in SQL/HTML                │
  ├───────────────┼─────────────────────────────────────────────────────────────┤
  │               │ Hardcoded secrets (grep -rnI                                │
  │ General       │ 'password|secret|api_key|token' --include='*.py'            │
  │               │ --include='*.ts' --include='*.env' — then filter false      │
  │               │ positives manually), .env files committed to git            │
  └───────────────┴─────────────────────────────────────────────────────────────┘

  Also check:
  - Unvalidated user input at system boundaries (API endpoints, CLI args, file
  reads)
  - Broad exception swallowing (except: pass, except Exception, bare catch {})
  - Resource leaks: unclosed connections, file handles, subscriptions, cursors
  - Race conditions in concurrent/async code, especially around shared mutable
  state

  B. Architectural Drift & Consistency

  - Patterns that contradict the project's documented architecture (e.g., business
  logic in route handlers, direct DB calls bypassing service layers)
  - Mixed paradigms within the same layer (callbacks vs. async/await, OOP vs.
  functional for equivalent tasks)
  - Inconsistent error handling strategies (some modules throw, some return null,
  some log-and-continue)
  - Naming inconsistencies: mixed casing conventions, inconsistent
  prefixes/suffixes across similar modules
  - State management: side effects in unexpected places, inconsistent
  singleton/global patterns
  - Circular dependencies between modules

  C. Duplication & Redundancy

  - Near-identical code blocks across files (check for structural similarity, not
  just exact matches)
  - Multiple implementations of the same concept (two HTTP clients, two config
  loaders, two retry wrappers)
  - Overlapping utility functions that could be consolidated
  - Repeated inline constants or magic numbers that should be centralized
  - Parallel data structures representing the same domain concept

  D. Dead Code & Technical Debt

  - Search for TODO, FIXME, HACK, XXX, TEMP, WORKAROUND comments — categorize by
  age (git blame) and severity
  - Unused imports, unreferenced private methods, unexported functions with zero
  callers
  - Commented-out code blocks (>3 lines)
  - Feature flags or conditional paths that are permanently on/off
  - Config keys, env vars, or CLI flags that nothing reads
  - Compatibility shims for constraints that no longer exist
  - Hand-rolled implementations where mature, maintained libraries now exist

  E. Dependency Health

  - Ghost dependencies: listed in manifest but never imported in source
  - Significantly outdated core dependencies (major versions behind)
  - Heavy dependencies pulled in for trivial functionality
  - Dependencies with known CVEs (use npm audit, pip-audit, or safety check if
  available)
  - Implicit dependencies relying on side effects or import order

  F. API Hygiene & Documentation

  - Public APIs or exported functions missing docstrings/JSDoc
  - Complex algorithms or non-obvious business logic with no explanatory comments
  - README or setup docs referencing removed functionality or outdated commands
  - Config files with undocumented or ambiguously named options
  - Inconsistent API response shapes or error formats across endpoints

  G. Performance Red Flags

  - N+1 query patterns or unbounded database queries (missing LIMIT)
  - Synchronous blocking calls in async contexts
  - Missing pagination on list endpoints
  - Large objects held in memory unnecessarily (full table loads, unbounded caches)
  - Repeated expensive computation that could be memoized/cached

  ---
  Phase 3: Output

  Executive Summary — The top 3-5 risks that should be addressed first, with a
  one-paragraph justification for each. Reference the quantitative evidence from
  Phase 1.

  Gravity Wells — The ranked Refactor Priority Score table:

  ┌──────┬─────────────────────┬───────┬─────────┬──────────┬────────┬──────────┐
  │ Rank │        File         │ Lines │ Commits │ Priority │  Has   │   Key    │
  │      │                     │       │  (3mo)  │   Score  │ Tests? │  Issue   │
  ├──────┼─────────────────────┼───────┼─────────┼──────────┼────────┼──────────┤
  │      │                     │       │         │          │        │ God      │
  │ 1    │ services/billing.py │ 1,247 │ 43      │   53,621 │ No     │ module,  │
  │      │                     │       │         │          │        │ no test  │
  │      │                     │       │         │          │        │ coverage │
  └──────┴─────────────────────┴───────┴─────────┴──────────┴────────┴──────────┘

  Findings Table — All findings, grouped into three tiers:

  Tier: Critical
  Category: Security
  File:Line: api/auth.py:47
  Issue: f"SELECT * FROM users WHERE id={uid}"
  Rationale / Risk: SQL injection in authenticated endpoint
  Recommended Fix: Use parameterized query
  ────────────────────────────────────────
  Tier: Debt
  Category: Duplication
  File:Line: utils/http.py:12, lib/fetch.py:30
  Issue: Two HTTP wrappers with overlapping retry logic
  Rationale / Risk: Maintenance burden, divergent bug fixes
  Recommended Fix: Consolidate into single client
  ────────────────────────────────────────
  Tier: Style
  Category: Naming
  File:Line: services/*.py
  Issue: Mix of snake_case and camelCase method names
  Rationale / Risk: Cognitive friction, grep difficulty
  Recommended Fix: Standardize to project convention

  False Positive Log — List 2-3 things that look like issues but were intentionally
   dismissed based on Phase 0 Calibration. This builds trust that the audit is
  calibrated, not just noisy.

  ┌───────────────────────────────┬────────────────────────────────────────────┐
  │        Apparent Issue         │               Why Dismissed                │
  ├───────────────────────────────┼────────────────────────────────────────────┤
  │ vendor/ contains 2,000-line   │ Third-party vendored code, excluded per    │
  │ files                         │ .gitignore conventions                     │
  ├───────────────────────────────┼────────────────────────────────────────────┤
  │ Line length exceeds 80 chars  │ Project ruff.toml sets line-length = 120   │
  │ in config.py                  │                                            │
  └───────────────────────────────┴────────────────────────────────────────────┘

  Metrics Snapshot:
  - Total source files / total lines
  - Highest Refactor Priority Score file
  - Modules missing tests (count + list)
  - Lint warning/error count
  - TODO/FIXME count and oldest unresolved (by git blame date)
  - Ghost dependency count

  ---
