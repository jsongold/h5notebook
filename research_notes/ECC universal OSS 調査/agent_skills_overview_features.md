# addyosmani/agent-skills: overview, layout, install, feature inventory

Method note: all facts come from a shallow clone of https://github.com/addyosmani/agent-skills at commit 2686b620fc1fed2e8f60c704839c766b8594c6b6 (2026-09-25 UTC-7; latest release tag 0.6.11) plus `gh repo view` / `gh api` metadata, read on 2026-09-30. "Source" links below point to the repo (paths at that commit). Counts and line/byte totals were computed locally with wc/find on the clone.

## What it is, author, goal, license

### Takeaway
A markdown-first collection of 25 SDLC "skills" (SKILL.md workflows), 4 reviewer personas, 9 slash commands, 7 shared checklists and 2 opt-in hook scripts sets, created by Addy Osmani (MIT). It has no runtime code and no installer of its own: it is distributed as a Claude Code plugin/marketplace and as plain skill folders via other tools' installers.

### Cited Findings
- Tagline: "Production-grade engineering skills for AI coding agents." Skills "encode the workflows, quality gates, and best practices that senior engineers use" — [README](https://github.com/addyosmani/agent-skills/blob/main/README.md)
- Creator/owner is Addy Osmani (only listed team member, role "Creator") — [README Team section](https://github.com/addyosmani/agent-skills/blob/main/README.md); [.claude-plugin/marketplace.json](https://github.com/addyosmani/agent-skills/blob/main/.claude-plugin/marketplace.json) owner field
- License MIT, "Copyright (c) 2025 Addy Osmani" — [LICENSE](https://github.com/addyosmani/agent-skills/blob/main/LICENSE); GitHub API reports key `mit`
- Version 0.6.11 is duplicated across plugin.json, .claude-plugin/plugin.json, .claude-plugin/marketplace.json, .codex-plugin/plugin.json, .agents/plugins/marketplace.json; CI has a `validate-versions.js` script to keep them in sync — [plugin.json](https://github.com/addyosmani/agent-skills/blob/main/plugin.json), [CI workflow](https://github.com/addyosmani/agent-skills/blob/main/.github/workflows/test-plugin-install.yml)
- Rationale ("Why Agent Skills?"): agents default to the shortest path, skipping specs, tests, security review; skills enforce senior-engineer discipline. Draws on Google practices (Software Engineering at Google: Hyrum's Law, Beyonce Rule, test pyramid 80/15/5, Chesterton's Fence, trunk-based dev, Shift Left) — [README](https://github.com/addyosmani/agent-skills/blob/main/README.md)
- Scope statement: CLAUDE.md/AGENTS.md say they configure agents working on this repo itself and are "not meant to be copied into other projects"; reusable assets are `skills/` — [AGENTS.md](https://github.com/addyosmani/agent-skills/blob/main/AGENTS.md)
- Positioned against Superpowers (obra) and Matt Pocock's skills in docs/comparison.md; its self-description: "organizes the whole product lifecycle (Define, Plan, Build, Verify, Review, Ship) with review personas, anti-rationalization guards, and an in-repo eval framework" (author's own comparison, so self-promotional) — [docs/comparison.md](https://github.com/addyosmani/agent-skills/blob/main/docs/comparison.md)

### Inferences
- Goal is a portable, tool-agnostic process library (discipline enforcement), not an environment/config bundle like ECC. No rules, MCP configs, or settings management.

### Gaps

## Repo layout, component counts, size

### Takeaway
Tiny compared with ECC: 208 tracked files, ~2.1 MB on disk (incl. .git-less working tree), ~14.1k lines of markdown. Core payload = 25 skills (~7.5k SKILL.md lines, ~400 KB with support files), 4 agents, 9 commands (3 duplicated adapter copies), 7 references, 3 hook scripts (2 features + a session-start hook), plus heavy contributor tooling (13 scripts, 84 eval files).

### Cited Findings
Counts computed locally (`find`, `wc`) at commit 2686b62:
- Tracked files: 208 (`git ls-files`); working tree ~2.1 MB excluding nothing else large.
- `skills/`: 25 dirs, 31 files, 8,451 lines, ~400 KB. 25 SKILL.md total 7,494 lines (avg about 300; range 178 idea-refine to 496 performance-optimization; lint cap is 500). Extra supporting files: idea-refine (examples.md, frameworks.md, refinement-criteria.md, scripts/idea-refine.sh 15 lines), security-and-hardening/references/hardening-patterns.md (324 lines), constraint-driven-development/references/floor-guard.md (168 lines). Only 3 of 25 skills have supporting files.
- `agents/`: 4 files (code-reviewer, security-auditor, test-engineer, web-performance-auditor), 493 lines, ~24.5 KB.
- Commands: 9 per adapter. `.claude/commands/*.md` (9 files, 272 lines) is the Claude Code copy; `commands/*.toml` (Antigravity/legacy, 266 lines) and `.gemini/commands/*.toml` (266 lines) are parallel copies. Names: spec, plan (planning.toml in toml dirs), build, test, constraints, review, webperf, code-simplify, ship. So about 27 command files for 9 logical commands. `validate-commands.js` enforces parity across dirs. — [README Project Structure](https://github.com/addyosmani/agent-skills/blob/main/README.md), [.claude/commands](https://github.com/addyosmani/agent-skills/tree/main/.claude/commands)
- `references/`: 7 checklists (accessibility, definition-of-done, observability, orchestration-patterns, performance, security, testing-patterns), 1,408 lines, ~67 KB.
- `hooks/`: 9 files, 1,653 lines: session-start.sh (+test), sdd-cache-pre.sh / sdd-cache-post.sh (+test, SDD-CACHE.md), simplify-ignore.sh (+test, SIMPLIFY-IGNORE.md). Only 4 production scripts (~1.6 KB + 4.3 KB + 4.6 KB + 14.5 KB); the rest are tests/docs. No hooks.json exists anywhere in the repo, so hooks are not auto-registered by the plugin (see install section).
- `docs/`: 16 files, 2,100 lines: per-tool setup guides (antigravity, codex, commandcode, copilot, copilot-cli, cursor, gemini-cli, opencode, windsurf), adoption-guide, advanced-per-agent-configuration, agents, comparison, developer-onboarding, getting-started, skill-anatomy.
- `scripts/`: 14 files, 3,802 lines (validate-skills, validate-commands, validate-versions, validate-reference-links, validate-artifact-paths, run-evals, floor-guard-reference-test, lib/skill-lint + tests). README says "13 scripts".
- `evals/`: 84 files, 2,110 lines: 25 case JSON files (one per skill), fixtures per skill, plugin evals (code-review-fires / stays-quiet with graders), evals/README.md, skill-impact.md.
- `.github/workflows/test-plugin-install.yml`: single CI file (see below). Also ISSUE_TEMPLATE dir.
- Adapter dirs: `.claude/` (commands, rules/skills-contributing.md), `.claude-plugin/` (plugin.json, marketplace.json), `.codex-plugin/plugin.json`, `.agents/plugins/marketplace.json` (Codex marketplace), `.gemini/commands/`, `.opencode/skills` (symlink to ../skills/), root `plugin.json` (Antigravity/Copilot CLI).
- Inconsistency: `.codex-plugin/plugin.json` longDescription says "24 production engineering workflows"; README/AGENTS/comparison say 25 skills (24 lifecycle + `using-agent-skills` meta-skill). Actual `skills/` dir count is 25. — [.codex-plugin/plugin.json](https://github.com/addyosmani/agent-skills/blob/main/.codex-plugin/plugin.json)

### Skill inventory (25), grouped by phase as in AGENTS.md
- Meta: using-agent-skills (192 lines; skill-discovery flowchart)
- Define: interview-me, idea-refine, spec-driven-development, constraint-driven-development
- Plan: planning-and-task-breakdown
- Build: incremental-implementation, test-driven-development, context-engineering, source-driven-development, doubt-driven-development, frontend-ui-engineering, api-and-interface-design
- Verify: browser-testing-with-devtools, debugging-and-error-recovery
- Review: code-review-and-quality, code-simplification, security-and-hardening, performance-optimization
- Ship: git-workflow-and-versioning, ci-cd-and-automation, deprecation-and-migration, documentation-and-adrs, observability-and-instrumentation, shipping-and-launch
— [AGENTS.md / CLAUDE.md](https://github.com/addyosmani/agent-skills/blob/main/CLAUDE.md), [skills/](https://github.com/addyosmani/agent-skills/tree/main/skills)

### Agents (4) and commands (9)
- Agents are personas with frontmatter `name` and `description` only (e.g., `code-reviewer`: "Senior code reviewer... five dimensions — correctness, readability, architecture, security, and performance"). Others: security-auditor (OWASP/threat model), test-engineer (Prove-It pattern), web-performance-auditor (CWV, Quick/Deep modes, run via /webperf) — [agents/](https://github.com/addyosmani/agent-skills/tree/main/agents)
- Command frontmatter: only `description`; body begins "Invoke the agent-skills:<skill> skill". `/build` = one task per run (RED, GREEN, regression, build, commit, stop); `/build auto` = require SPEC.md, clean git baseline, plan, one approval gate, then autonomous per-task commits, stop on failure/risky step. `/ship` = parallel fan-out of code-reviewer + security-auditor + test-engineer, then merged go/no-go and rollback plan. `/spec` writes SPEC.md with six areas (objective, commands, project structure, code style, testing, boundaries) — [build.md](https://github.com/addyosmani/agent-skills/blob/main/.claude/commands/build.md), [ship.md](https://github.com/addyosmani/agent-skills/blob/main/.claude/commands/ship.md), [spec.toml](https://github.com/addyosmani/agent-skills/blob/main/commands/spec.toml)

### Hooks (opt-in, not wired by plugin)
- session-start.sh: injects the using-agent-skills meta-skill as SessionStart `additionalContext` via jq. Header states it is "Not wired by the plugin" because Claude Code/Codex already route skills from descriptions; use only on hosts without native routing — [hooks/session-start.sh](https://github.com/addyosmani/agent-skills/blob/main/hooks/session-start.sh)
- sdd-cache (PreToolUse + async PostToolUse on WebFetch): disk cache under `.claude/sdd-cache/` of fetched docs for source-driven-development, revalidated with ETag/If-Modified-Since on every reuse (no TTL) — [hooks/SDD-CACHE.md](https://github.com/addyosmani/agent-skills/blob/main/hooks/SDD-CACHE.md)
- simplify-ignore (PreToolUse Read, PostToolUse Edit|Write, Stop): protects `simplify-ignore-start/end` blocks by replacing them with placeholders before the model reads them; backups in `.claude/.simplify-ignore-cache/` — [hooks/SIMPLIFY-IGNORE.md](https://github.com/addyosmani/agent-skills/blob/main/hooks/SIMPLIFY-IGNORE.md)

### Inferences
- Comparable-scale view vs ECC (68 agents/293 skills/94 commands): agent-skills is roughly 1/12 the skills, 1/17 the agents, 1/10 the logical commands, ~0 rules. The heavy component is CI/eval tooling, not content.
- Skills are long-form (avg ~300 lines each, hard-capped at 500) and few, versus many short skills.

### Gaps
- Did not compute token counts.

## Install methods and what they write to the machine

### Takeaway
No project-owned installer. Claude Code: two `/plugin` commands (marketplace add + install). Other tools use their own native installers or copy `skills/`. Nothing in the repo edits ~/.claude/settings.json; hooks require the user to hand-paste JSON into `.claude/settings.json`.

### Cited Findings (exact commands from README)
- Universal (open skills CLI by vercel-labs, 70+ agents): `npx skills add addyosmani/agent-skills` (all 25), `npx skills add addyosmani/agent-skills --list`, `npx skills add addyosmani/agent-skills --skill code-review-and-quality` — [README Quick Start](https://github.com/addyosmani/agent-skills/blob/main/README.md). Caveat in README: per-skill npx install copies only `skills/<name>/`, not repo-level `references/`, so shared checklist links break (tracked issue #361).
- Claude Code marketplace: `/plugin marketplace add addyosmani/agent-skills` then `/plugin install agent-skills@addy-agent-skills`. HTTPS fallback: `/plugin marketplace add https://github.com/addyosmani/agent-skills.git`; SSH workaround `git config --global url."https://github.com/".insteadOf git@github.com:` — README
- Claude Code local dev: `git clone https://github.com/addyosmani/agent-skills.git` then `claude --plugin-dir /path/to/agent-skills` — README
- Plugin manifest `.claude-plugin/plugin.json`: `"commands": ["./.claude/commands", "./commands"]`, `"skills": "./skills"`, `"experimental": {"evals": "evals/plugin"}`. It does not declare agents or hooks paths (agents/ is described in AGENTS.md as auto-discovered from the plugin's agents/ dir by Claude Code) — [.claude-plugin/plugin.json](https://github.com/addyosmani/agent-skills/blob/main/.claude-plugin/plugin.json), [AGENTS.md](https://github.com/addyosmani/agent-skills/blob/main/AGENTS.md)
- Codex (CLI v0.122+): `codex plugin marketplace add addyosmani/agent-skills` then `codex plugin add agent-skills@agent-skills`; skills invoked with `@spec-driven-development` — README
- Gemini CLI: `gemini skills install https://github.com/addyosmani/agent-skills.git --path skills` (or local `gemini skills install ./agent-skills/skills/`) — README
- Antigravity CLI: `agy plugin install https://github.com/addyosmani/agent-skills.git`; README notes that in affected releases legacy command TOMLs are not discoverable — README
- Command Code: `cmd skills add addyosmani/agent-skills` (`--global`, `-s <skill>`) — README
- Cursor: copy/sync skills into `.cursor/skills/` and short policies in `.cursor/rules/*.mdc`; Windsurf: paste into rules; OpenCode: copy skills to `.opencode/skills/` or `~/.config/opencode/skills/` plus AGENTS.md; Copilot: agents as personas + `.github/copilot-instructions.md`, or copilot CLI plugin; Kiro: `.kiro/skills/`; "Other": any agent that accepts system prompts — README, docs/*-setup.md
- Supported tools named in README: Claude Code, Cursor, Codex, GitHub Copilot (IDE + CLI), Gemini CLI, Antigravity, Windsurf, OpenCode, Kiro, Command Code, Cline and others via `npx skills`. Comparison doc lists "Claude Code, Cursor, Gemini CLI, Antigravity, OpenCode, Windsurf, Copilot, Kiro, Codex, Command Code" — [docs/comparison.md](https://github.com/addyosmani/agent-skills/blob/main/docs/comparison.md)
- Machine writes: plugin install writes into Claude Code's plugin cache/marketplace state (managed by Claude Code, not by repo scripts). The repo ships no install script and no code that touches `~/.claude/settings.json`. Hooks are manual: docs give JSON snippets for the user to add to `.claude/settings.json` (project) or `.claude/settings.local.json`, using `${CLAUDE_PROJECT_DIR}/hooks/...` paths — [SDD-CACHE.md](https://github.com/addyosmani/agent-skills/blob/main/hooks/SDD-CACHE.md), [SIMPLIFY-IGNORE.md](https://github.com/addyosmani/agent-skills/blob/main/hooks/SIMPLIFY-IGNORE.md). session-start.sh requires `jq`.
- CI verifies install: `claude plugin validate .`, `claude plugin marketplace add ./`, `claude plugin install agent-skills@addy-agent-skills --scope user` — [workflow](https://github.com/addyosmani/agent-skills/blob/main/.github/workflows/test-plugin-install.yml)

### Inferences
- Installation is non-invasive (no global config mutation), which contrasts with ECC's settings.json-writing installer. Hook features are opt-in and manual, so a plugin install gives skills + commands (+ agents) only.

### Gaps
- Did not run the install or inspect the resulting ~/.claude plugin cache; the exact on-disk location for plugin install is Claude Code behavior, not verified here.
- Not verified that `agents/` is actually auto-loaded by the manifest (manifest has no agents key; claim comes from AGENTS.md).

## Dependencies between components; guidance for agents editing the repo

### Takeaway
Dependencies are implicit (prose references), not declared in frontmatter/manifest; a linter warns on dangling skill references. Contributor guidance and validation are unusually strong: CLAUDE.md, AGENTS.md, a path-scoped rule, CONTRIBUTING pre-flight, lint scripts, unit tests and 3-tier evals in CI.

### Cited Findings
- Frontmatter is only `name` and `description` (checked across all 25; no tools/allowed-tools/requires/dependencies fields). Agents also only `name`/`description`; commands only `description` — local grep of files.
- Cross-references are prose: 15 of 25 skills mention other skills by backticked name (e.g., using-agent-skills routes to 9 skills; interview-me to 5; spec-driven-development to 5; constraint-driven-development to 6). Commands invoke skills by namespaced name ("Invoke the agent-skills:incremental-implementation skill alongside agent-skills:test-driven-development"); /ship names personas via `subagent_type`. Skills point to shared references via relative paths like `../../references/security-checklist.md`; 8 skills reference `references/` (code-review, frontend-ui, observability, incremental-impl, performance, shipping, doubt-driven, constraint-driven's own) — local grep
- Composition rules: skills = "how", personas = "who", commands = "when"; user/slash command is orchestrator; personas do not invoke other personas (may invoke skills); only endorsed multi-persona pattern is parallel fan-out + merge (/ship); doubt-driven-development must not be added to a persona's `skills:` frontmatter — [AGENTS.md](https://github.com/addyosmani/agent-skills/blob/main/AGENTS.md), skills/doubt-driven-development/SKILL.md line 46
- Rules for agents editing the repo: CLAUDE.md (4.1 KB) and AGENTS.md (5.4 KB) with explicit scope disclaimers; `.claude/rules/skills-contributing.md` (path-scoped to `skills/**`: anti-duplication guardrail); CONTRIBUTING.md (8.7 KB) with "before proposing a new skill" pre-flight (search catalog, check open PRs, justify gap); CLAUDE.md "Never: Add skills that are vague advice instead of actionable processes / Never: Duplicate content between skills — reference other skills instead" — [CLAUDE.md](https://github.com/addyosmani/agent-skills/blob/main/CLAUDE.md), [.claude/rules/skills-contributing.md](https://github.com/addyosmani/agent-skills/blob/main/.claude/rules/skills-contributing.md)
- Validation: `scripts/lib/skill-lint.js` enforces: SKILL.md exists, frontmatter name/description, name equals directory name, lowercase-hyphen, description <= 1024 chars and contains a "when to use" trigger, required sections present, non-empty subdirs; warnings for dangling cross-skill refs and SKILL.md > 500 lines. Other validators: command parity across `.claude/commands`, `commands`, `.gemini/commands` with description sync; manifest version sync; references/ link check; spec/plan/todo artifact paths — [skill-lint.js](https://github.com/addyosmani/agent-skills/blob/main/scripts/lib/skill-lint.js), [workflow](https://github.com/addyosmani/agent-skills/blob/main/.github/workflows/test-plugin-install.yml)
- Tests/CI: single workflow runs on push/PR on ubuntu + macOS + Windows: validate-skills, node unit tests for each validator, `run-evals.js --min-rank1 95` (routing floor), hook regression tests (`sdd-cache-test.sh` with stubbed curl, `simplify-ignore-test.sh`, `session-start-test.sh`), `claude plugin validate .` and an actual plugin install — same workflow file.
- Evals: 3 tiers — (1) structural, (2) trigger/routing via stemmed TF-IDF over descriptions (deterministic, CI), (3) behavioral via headless `claude` runs graded against `expectations[]` (on demand, spends tokens); includes pressure cases (time pressure, sunk cost, authority). Adopts Anthropic skill-creator's evals.json schema — [evals/README.md](https://github.com/addyosmani/agent-skills/blob/main/evals/README.md)
- AGENTS.md's own "Commands" note in CLAUDE.md: "`npm test` — Not applicable (this is a documentation project)"; no package.json in repo (root listing has none) — [CLAUDE.md](https://github.com/addyosmani/agent-skills/blob/main/CLAUDE.md)

### Inferences
- Because dependencies are implicit, installing a subset (e.g., `npx skills add --skill X`) can break relative `../../references/` links and references to sibling skills; the repo acknowledges this (issue #361) and is moving toward per-skill `references/` (constraint-driven, security-and-hardening already do).
- Repo has no `package.json`; Node scripts have no dependencies (only fs/path), so contributors run them with plain `node`.

### Gaps
- Did not verify whether the marketplace listing depends on the `agent-skills:` namespace prefix used inside commands.

## Notable design ideas

### Takeaway
Process-not-prose skills with a fixed anatomy (Overview, When to Use, Process, Common Rationalizations, Red Flags, Verification), an explicit meta-skill router, human-gated lifecycle commands, adversarial/verification-heavy skills, and CI evals that test routing.

### Cited Findings
- Fixed anatomy enforced by linter: Overview / When to Use / Process / Common Rationalizations / Red Flags / Verification — [CLAUDE.md Conventions](https://github.com/addyosmani/agent-skills/blob/main/CLAUDE.md), [README How Skills Work](https://github.com/addyosmani/agent-skills/blob/main/README.md)
- Stated key design choices: "Process, not prose"; anti-rationalization tables; "Verification is non-negotiable" (evidence: tests, build output, runtime data); progressive disclosure (SKILL.md entry point, supporting files only when content > 100 lines) — README, CLAUDE.md
- Anti-rationalization example (test-driven-development, 7 rows): "I'll write tests after the code works" -> "You won't. And tests written after the fact test implementation, not behavior."; "Let me run the tests again just to be extra sure" -> repeated runs add nothing unless code changed — [skills/test-driven-development/SKILL.md](https://github.com/addyosmani/agent-skills/blob/main/skills/test-driven-development/SKILL.md)
- AGENTS.md's OpenCode section adds an anti-rationalization rule for routing: check for skill "even 1% chance"; ignore thoughts like "This is too small for a skill" — [AGENTS.md](https://github.com/addyosmani/agent-skills/blob/main/AGENTS.md)
- Lifecycle: DEFINE (/spec) -> PLAN (/plan) -> BUILD (/build) -> VERIFY (/test) -> REVIEW (/review) -> SHIP (/ship); extras /constraints, /code-simplify, /webperf; `/build auto` single approval gate then per-task RED/GREEN/commit loop — README
- Notable skills: interview-me (one question at a time until ~95% confidence); doubt-driven-development (CLAIM -> EXTRACT -> DOUBT -> RECONCILE -> STOP, fresh-context adversarial review, optional cross-model escalation); constraint-driven-development (writes CONSTRAINTS.md, detects agents weakening the bar: new @ts-ignore/eslint-disable, skipped/deleted tests, thresholds lowered; ships floor-guard reference); source-driven-development (cite official docs, flag unverified) with sdd-cache hook; test-driven-development (test pyramid 80/15/5, Prove-It pattern for bugs, DAMP over DRY); code-review-and-quality (five axes, ~100-line change sizing, Nit/Optional/FYI labels) — README skill tables
- Meta-skill `using-agent-skills` provides a discovery flowchart mapping task type to skill; the session-start hook to inject it is deliberately not wired by default to avoid a duplicate router — [skills/using-agent-skills/SKILL.md](https://github.com/addyosmani/agent-skills/blob/main/skills/using-agent-skills/SKILL.md), [hooks/session-start.sh](https://github.com/addyosmani/agent-skills/blob/main/hooks/session-start.sh)
- Orchestration philosophy: no router persona; user/commands orchestrate; parallel fan-out + merge only — [references/orchestration-patterns.md](https://github.com/addyosmani/agent-skills/blob/main/references/orchestration-patterns.md), AGENTS.md
- Platform-adapter design: shared portable core (`skills/`, `agents/`, `references/`) plus thin host-specific adapter dirs; README explicitly says host-specific paths are native discovery conventions and cannot be merged — README Project Structure

### Inferences
- Philosophy is "small, opinionated, verified catalog" (quality gates + evals) versus ECC's breadth-first catalog plus environment installer; likely cost is coverage (no language-specific skills, no rules layer, no MCP configs).
- Command/skill/persona split is a clean 3-layer model; ECC comparison points: ECC commands/agents/skills counts are much larger and it writes settings, while agent-skills leaves configuration to the host.

### Gaps
- Did not read every SKILL.md in full; assessments of individual skill quality are based on frontmatter, headings and README summaries, not full review.

## How a typical skill, agent and command is written (with real excerpts)

### Takeaway
Skill = frontmatter (name, description with "Use when...") + fixed sections; body is imperative, stepwise, with code blocks, ending in rationalization table, red flags and a checkbox verification list. Agent = persona system prompt (frontmatter name/description, review framework, output template, rules, composition note). Command = short prompt that says "Invoke the agent-skills:<skill> skill" then lists numbered steps.

### Cited Findings
- Skill (skills/test-driven-development/SKILL.md, 398 lines). Section order: Overview, When to Use, (Discover the Stack First), The TDD Cycle, Prove-It Pattern, Test Pyramid, Writing Good Tests, Anti-Patterns, Browser Testing, When to Use Subagents, See Also, Common Rationalizations, Red Flags, Verification. Excerpts:
  - Frontmatter: `name: test-driven-development` / `description: Drives development with tests using the red-green-refactor loop. Use when implementing any logic, fixing any bug, or changing any behavior. ...`
  - Step style: "### Step 1: RED — Write a Failing Test. Write the test first. It must fail. A test that passes immediately proves nothing." followed by a TypeScript example.
  - Rationalization table: `| "I'll write tests after the code works" | You won't. And tests written after the fact test implementation, not behavior. |`
  - Red flag: "Running the same test command twice in a row without any intervening code change"
  - Verification: "- [ ] Bug fixes include a reproduction test that failed before the fix"; "- [ ] No tests were skipped or disabled"
  — [SKILL.md](https://github.com/addyosmani/agent-skills/blob/main/skills/test-driven-development/SKILL.md)
- Agent (agents/code-reviewer.md; frontmatter only `name`, `description`). Body: "You are an experienced Staff Engineer conducting a thorough code review." then "Review Framework" with five numbered dimensions as question lists (e.g. "### 4. Security - Is user input validated and sanitized at system boundaries? - Are secrets kept out of code, logs, and version control?"), "Output Format" with severity labels (Critical / Required / Optional / Nit), a review output template (including "What's Done Well" and "Verification Story"), numbered Rules ("Review the tests first", "Don't approve code with Critical issues"), and a Composition section: "Do not invoke from another persona... orchestration belongs to slash commands, not personas." — [code-reviewer.md](https://github.com/addyosmani/agent-skills/blob/main/agents/code-reviewer.md)
- Command (.claude/commands/review.md, 846 bytes; frontmatter only `description`): "Invoke the agent-skills:code-review-and-quality skill. Review the current changes (staged or recent commits) across all five axes: 1. Correctness ... 4. Security ... (Use security-and-hardening skill) ... Categorize findings as Critical, Important, or Suggestion." Note the command's severity names (Critical/Important/Suggestion) differ from the agent's (Critical/Required/Optional/Nit): a small inconsistency. Larger commands (build.md 3.9 KB, ship.md 4.6 KB) use `$ARGUMENTS` modes and phased steps, e.g. `/build auto`: "Require a spec... Establish a clean baseline (git status --porcelain)... Single checkpoint... one commit per task". TOML copies wrap the same text: `description = "..."` and `prompt = """ ... """` (commands/spec.toml). — [review.md](https://github.com/addyosmani/agent-skills/blob/main/.claude/commands/review.md), [build.md](https://github.com/addyosmani/agent-skills/blob/main/.claude/commands/build.md), [spec.toml](https://github.com/addyosmani/agent-skills/blob/main/commands/spec.toml)

### Inferences
- Content is natural-language process guidance only; no tool restrictions, model settings or hook declarations in any frontmatter.

### Gaps
- Only sampled a few bodies; other skills follow the same anatomy per the linter but were not all read.

## What the manifests and hook scripts actually do

### Takeaway
Manifests are thin pointers to directories plus metadata. Hooks are three optional bash scripts (need jq) that are never registered automatically.

### Cited Findings
- `.claude-plugin/plugin.json`: name, version 0.6.11, description, author, license MIT, `"commands": ["./.claude/commands", "./commands"]`, `"skills": "./skills"`, `"experimental": {"evals": "evals/plugin"}`. No hooks, agents, MCP or settings keys.
- `.claude-plugin/marketplace.json`: marketplace name `addy-agent-skills`, one plugin `agent-skills` with `source: {source: github, repo: addyosmani/agent-skills}` (hence install id `agent-skills@addy-agent-skills`).
- `.codex-plugin/plugin.json`: `"skills": "./skills/"` plus an `interface` block (displayName, shortDescription, defaultPrompt, capabilities Interactive/Read/Write). `.agents/plugins/marketplace.json`: local source `./`, policy `installation: AVAILABLE`, `authentication: ON_INSTALL`. Root `plugin.json`: only name/version/description (Antigravity/Copilot CLI).
- `hooks/session-start.sh`: reads `skills/using-agent-skills/SKILL.md`, emits via jq `{hookSpecificOutput:{hookEventName:"SessionStart",additionalContext:"agent-skills loaded... <SKILL.md content>"}}`; if jq is missing emits a JSON notice and exits 0. Not wired by the plugin.
- `hooks/sdd-cache-pre.sh` (PreToolUse, matcher WebFetch): takes `.tool_input.url` from stdin JSON, key = sha256(URL) truncated to 128 bits, looks up `${CLAUDE_PROJECT_DIR}/.claude/sdd-cache/<hash>.json`, reads stored fetched_at/prompt/etag/last_modified; per SDD-CACHE.md it revalidates with the origin (If-None-Match / If-Modified-Since) and serves cache only on 304. `sdd-cache-post.sh` (PostToolUse, async) stores the fetched result. Debug via `SDD_CACHE_DEBUG=1`. — [sdd-cache-pre.sh](https://github.com/addyosmani/agent-skills/blob/main/hooks/sdd-cache-pre.sh), [SDD-CACHE.md](https://github.com/addyosmani/agent-skills/blob/main/hooks/SDD-CACHE.md)
- `hooks/simplify-ignore.sh` (one script, three events): PreToolUse Read backs the file up to `.claude/.simplify-ignore-cache/` and rewrites `simplify-ignore-start/end` blocks to `BLOCK_<hash>` placeholders on disk; PostToolUse Edit|Write expands placeholders then re-filters; Stop restores real content. Requires jq and shasum/sha1sum; handles bash 5.2 `patsub_replacement`. — [simplify-ignore.sh](https://github.com/addyosmani/agent-skills/blob/main/hooks/simplify-ignore.sh)
- Registration is manual JSON in `.claude/settings.json`, using `bash "${CLAUDE_PROJECT_DIR}/hooks/<script>"` (see the two hook docs).

### Inferences
- simplify-ignore mutates working files in place while a session is active (recovery depends on the Stop hook or the backup), which is a riskier pattern than the read-only skills.

### Gaps
- Did not read the full body of sdd-cache-pre/post or the response shape returned on a cache hit.

## How components reference each other

### Takeaway
References are prose or relative-path links; a linter only checks that name-shaped references resolve.

### Cited Findings
- Command to skill: "Invoke the agent-skills:incremental-implementation skill alongside agent-skills:test-driven-development" (build.md); review.md mentions security-and-hardening and performance-optimization in parentheses.
- Command to agent: ship.md tells the runner to spawn subagents concurrently with `subagent_type` equal to persona `name` (code-reviewer, security-auditor, test-engineer).
- Agent to skill: code-reviewer says it uses "the same severity labels as the `code-review-and-quality` skill"; agents link `../docs/agents.md`.
- Skill to skill: backticked or named skill in prose (e.g. interview-me mentions idea-refine, spec-driven-development, planning-and-task-breakdown, doubt-driven-development, source-driven-development; using-agent-skills mentions 9 skills in a flowchart).
- Skill to shared reference: relative paths such as `../../references/security-checklist.md` (code-review-and-quality lines 351-352); skill-local `references/floor-guard.md` in constraint-driven-development.
- No manifest or frontmatter field expresses any of these links.

### Inferences
- Deleting or renaming a skill breaks references silently unless the lint warning is noticed.

### Gaps
- None beyond earlier notes.

## How a typical skill, agent and command is written (with real excerpts)

### Takeaway
Skill = frontmatter (name, description with "Use when...") + fixed sections; body is imperative and stepwise with code blocks, ending in a rationalization table, red flags and a checkbox verification list. Agent = persona system prompt (frontmatter name/description, review framework, output template, rules, composition note). Command = short prompt that says "Invoke the agent-skills:<skill> skill" then lists numbered steps.

### Cited Findings
- Skill (skills/test-driven-development/SKILL.md, 398 lines). Section order: Overview, When to Use, Discover the Stack First, The TDD Cycle, Prove-It Pattern, Test Pyramid, Writing Good Tests, Anti-Patterns, Browser Testing, When to Use Subagents, See Also, Common Rationalizations, Red Flags, Verification. Excerpts:
  - Frontmatter: `name: test-driven-development` / `description: Drives development with tests using the red-green-refactor loop. Use when implementing any logic, fixing any bug, or changing any behavior. ...`
  - Step style: "### Step 1: RED — Write a Failing Test. Write the test first. It must fail. A test that passes immediately proves nothing." followed by a TypeScript example.
  - Rationalization table row: `| "I'll write tests after the code works" | You won't. And tests written after the fact test implementation, not behavior. |`
  - Red flag: "Running the same test command twice in a row without any intervening code change"
  - Verification: "- [ ] Bug fixes include a reproduction test that failed before the fix"; "- [ ] No tests were skipped or disabled"
  — [SKILL.md](https://github.com/addyosmani/agent-skills/blob/main/skills/test-driven-development/SKILL.md)
- Agent (agents/code-reviewer.md; frontmatter only `name`, `description`). Body: "You are an experienced Staff Engineer conducting a thorough code review." then "Review Framework" with five numbered dimensions as question lists (e.g. "### 4. Security - Is user input validated and sanitized at system boundaries? - Are secrets kept out of code, logs, and version control?"), "Output Format" with severity labels (Critical / Required / Optional / Nit), a review output template (incl. "What's Done Well", "Verification Story"), numbered Rules ("Review the tests first", "Don't approve code with Critical issues"), and a Composition section: "Do not invoke from another persona... orchestration belongs to slash commands, not personas." — [code-reviewer.md](https://github.com/addyosmani/agent-skills/blob/main/agents/code-reviewer.md)
- Command (.claude/commands/review.md, 846 bytes; frontmatter only `description`): "Invoke the agent-skills:code-review-and-quality skill. Review the current changes (staged or recent commits) across all five axes: 1. Correctness ... 4. Security ... (Use security-and-hardening skill) ... Categorize findings as Critical, Important, or Suggestion." The command's severity names (Critical/Important/Suggestion) differ from the agent's (Critical/Required/Optional/Nit), a small inconsistency. Larger commands (build.md 3.9 KB, ship.md 4.6 KB) use `$ARGUMENTS` modes and phased steps, e.g. `/build auto`: require a spec, clean git baseline via `git status --porcelain`, single approval checkpoint, one commit per task. TOML copies wrap the same text as `description = "..."` and `prompt = """ ... """` (commands/spec.toml). — [review.md](https://github.com/addyosmani/agent-skills/blob/main/.claude/commands/review.md), [build.md](https://github.com/addyosmani/agent-skills/blob/main/.claude/commands/build.md), [spec.toml](https://github.com/addyosmani/agent-skills/blob/main/commands/spec.toml)

### Inferences
- Content is natural-language process guidance only; no tool restrictions, model settings or hook declarations in any frontmatter.

### Gaps
- Only sampled a few bodies; other skills follow the same anatomy per the linter but were not all read.

## What the manifests and hook scripts actually do

### Takeaway
Manifests are thin pointers to directories plus metadata. Hooks are three optional bash scripts (need jq) that are never registered automatically.

### Cited Findings
- `.claude-plugin/plugin.json`: name, version 0.6.11, description, author, license MIT, `"commands": ["./.claude/commands", "./commands"]`, `"skills": "./skills"`, `"experimental": {"evals": "evals/plugin"}`. No hooks, agents, MCP or settings keys.
- `.claude-plugin/marketplace.json`: marketplace name `addy-agent-skills`, one plugin `agent-skills` with `source: {source: github, repo: addyosmani/agent-skills}` (hence install id `agent-skills@addy-agent-skills`).
- `.codex-plugin/plugin.json`: `"skills": "./skills/"` plus an `interface` block (displayName, shortDescription, defaultPrompt, capabilities). `.agents/plugins/marketplace.json`: local source `./`, policy `installation: AVAILABLE`, `authentication: ON_INSTALL`. Root `plugin.json`: only name/version/description.
- `hooks/session-start.sh`: reads `skills/using-agent-skills/SKILL.md`, emits via jq `{hookSpecificOutput:{hookEventName:"SessionStart",additionalContext:"agent-skills loaded... <SKILL.md content>"}}`; if jq is missing it emits a JSON notice and exits 0. Not wired by the plugin.
- `hooks/sdd-cache-pre.sh` (PreToolUse, matcher WebFetch): reads `.tool_input.url` from stdin JSON, key = sha256(URL) truncated to 128 bits, looks up `${CLAUDE_PROJECT_DIR}/.claude/sdd-cache/<hash>.json` (fields fetched_at, prompt, etag, last_modified); per SDD-CACHE.md it revalidates with the origin (If-None-Match / If-Modified-Since) and serves cache only on 304. `sdd-cache-post.sh` (PostToolUse, async) stores the fetched result. Debug via `SDD_CACHE_DEBUG=1`. — [sdd-cache-pre.sh](https://github.com/addyosmani/agent-skills/blob/main/hooks/sdd-cache-pre.sh), [SDD-CACHE.md](https://github.com/addyosmani/agent-skills/blob/main/hooks/SDD-CACHE.md)
- `hooks/simplify-ignore.sh` (one script, three events): PreToolUse Read backs the file up to `.claude/.simplify-ignore-cache/` and rewrites `simplify-ignore-start/end` blocks to `BLOCK_<hash>` placeholders on disk; PostToolUse Edit|Write expands placeholders then re-filters; Stop restores real content. Requires jq and shasum/sha1sum. — [simplify-ignore.sh](https://github.com/addyosmani/agent-skills/blob/main/hooks/simplify-ignore.sh)
- Registration is manual JSON in `.claude/settings.json` using `bash "${CLAUDE_PROJECT_DIR}/hooks/<script>"` (see the two hook docs).

### Inferences
- simplify-ignore mutates working files in place during a session (recovery relies on the Stop hook or the backup), a riskier pattern than the read-only skills.

### Gaps
- Did not read the full body of sdd-cache-pre/post or the response shape returned on a cache hit.

## How components reference each other

### Takeaway
References are prose or relative-path links; a linter only checks that name-shaped references resolve.

### Cited Findings
- Command to skill: "Invoke the agent-skills:incremental-implementation skill alongside agent-skills:test-driven-development" (.claude/commands/build.md); review.md mentions security-and-hardening and performance-optimization in parentheses.
- Command to agent: .claude/commands/ship.md tells the runner to spawn subagents concurrently with `subagent_type` equal to the persona `name` (code-reviewer, security-auditor, test-engineer).
- Agent to skill: code-reviewer uses "the same severity labels as the `code-review-and-quality` skill" and links `../docs/agents.md`.
- Skill to skill: named in prose (interview-me mentions idea-refine, spec-driven-development, planning-and-task-breakdown, doubt-driven-development, source-driven-development; using-agent-skills routes to 9 skills via a flowchart).
- Skill to shared reference: relative paths such as `../../references/security-checklist.md` (skills/code-review-and-quality/SKILL.md lines 351-352); skill-local `references/floor-guard.md` in constraint-driven-development.
- No manifest or frontmatter field expresses any of these links.

### Inferences
- Renaming or deleting a skill breaks references silently unless the lint warning is noticed.

### Gaps
- None beyond those above.
