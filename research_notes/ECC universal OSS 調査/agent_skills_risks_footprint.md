# addyosmani/agent-skills: risks and context footprint (as of 2026-09-30)

Method: shallow clone of https://github.com/addyosmani/agent-skills at commit 2686b620 (2026-09-25, plugin version 0.6.11, 1 commit visible in the depth-1 clone, so contributor count/history not assessed). Files read directly; sizes measured with Python (chars of frontmatter `description`, chars/lines of files). Token estimate = chars / 4 (rough English heuristic, not a tokenizer; real counts may be +-25%). Claude Code behavior verified against https://code.claude.com/docs/en/skills and https://code.claude.com/docs/en/plugins-reference (fetched today). Repo is 207 files, about 3.4 MB on disk with .git (du of clone).

## Q1. What does installing write/modify; does it ship hooks that run shell; permission changes; supply chain; conflicts with ~/.claude

### Takeaway
Plugin install path is low-risk: the plugin manifest registers skills/commands only, no hooks.json, no MCP, no settings/permission changes, no installer script. Shell hooks exist in hooks/ but are opt-in, manual copy-paste into settings.json, and one of them (simplify-ignore) rewrites files in place and is the only real operational hazard. Far smaller attack surface than ECC's settings.json-writing installer.

### Cited Findings
- Install methods documented: `/plugin marketplace add addyosmani/agent-skills` + `/plugin install agent-skills@addy-agent-skills`; also `npx skills add addyosmani/agent-skills` (third-party `skills` CLI, not in this repo; no package.json in repo) — README.md lines ~49-83 in the [repo](https://github.com/addyosmani/agent-skills)
- Repo contains no installer script of its own; no package.json, no install.sh. Only Node scripts are validators/eval runner in scripts/ (validate-skills.js, run-evals.js, etc.); grep found the only child_process use is `execFileSync` in scripts/run-evals.js; no network fetch or npm install in scripts — measured by grep in the clone
- Plugin manifest .claude-plugin/plugin.json declares only `commands: ["./.claude/commands","./commands"]`, `skills: "./skills"`, and `experimental.evals`. No `hooks`, `mcpServers`, `settings`, or `bin` keys. There is no hooks/hooks.json (`find -name hooks.json` returned nothing). Per docs, hooks are auto-loaded from `hooks/hooks.json` or manifest key, so the plugin registers no hooks — [plugins-reference](https://code.claude.com/docs/en/plugins-reference) (Standard layout / hooks) plus repo files
- hooks/session-start.sh states in its header: "Not wired by the plugin"; it would inject using-agent-skills meta-skill via SessionStart JSON envelope, requires jq — hooks/session-start.sh
- hooks/sdd-cache-pre.sh / sdd-cache-post.sh (WebFetch Pre/PostToolUse) are documented as manual setup: user pastes JSON into `.claude/settings.json`, command `bash "${CLAUDE_PROJECT_DIR}/hooks/sdd-cache-pre.sh"` — hooks/SDD-CACHE.md. Behavior: PostToolUse stores WebFetch result in `<project>/.claude/sdd-cache/<sha>.json`, issues extra `curl -sI` HEAD to the same URL; PreToolUse does HEAD with If-None-Match and, on 304, blocks WebFetch (exit 2) and feeds cached body back to agent via stderr. Network egress limited to the URL the agent already fetched; local write limited to project .claude/sdd-cache (gitignored). Risk: a cached model-summarized page is replayed as tool output (stale/injected content persists across sessions; cache poisoning only if the origin serves it) — hooks/sdd-cache-pre.sh, sdd-cache-post.sh
- hooks/simplify-ignore.sh (PreToolUse Read, PostToolUse Edit|Write, Stop) is manual opt-in (hooks/SIMPLIFY-IGNORE.md). It replaces `simplify-ignore-start/end` blocks in the user's source files in place with BLOCK_<hash> placeholders while the session is active, backs up to `.claude/.simplify-ignore-cache/`, and restores on Stop. Header states "The file on disk ALWAYS has placeholders while the session is active." Failure mode (crash/kill before Stop) leaves working-tree source with placeholders; the script has a Stop-time expand path and a backup fallback, and a 17.8 KB test file, but I did not run it — hooks/simplify-ignore.sh (14.5 KB)
- Neither hook configuration changes `permissions`; snippets only add hooks entries. No `allow`/`deny`/`defaultMode`/bypass content in any SKILL/agent/command frontmatter (grep for allowed-tools returned no hits in skills/agents/commands). Agents have no `tools:` or `model:` frontmatter (grep), so they inherit all tools of the parent by default per general Claude Code behavior (agent-tool inheritance not re-verified today)
- Skill/command text with risky shell: only mentions of `git push` (tag, rollback example) and an explicit prohibition of `npm audit fix --force` in security-and-hardening; no `curl | sh`, `sudo`, `rm -rf` hits — grep over skills/agents/commands
- source-driven-development skill instructs the agent to treat fetched docs as untrusted input (line ~99) — skills/source-driven-development/SKILL.md
- Supply chain: single-author repo (all visible commits by Addy Osmani in the shallow view; the clone had 1 commit so the full history was not inspected), MIT license, marketplace entry sources `github: addyosmani/agent-skills` — .claude-plugin/marketplace.json. Plugin `version: 0.6.11` is set, which per docs pins users to that version until it changes — [plugins-reference `version`](https://code.claude.com/docs/en/plugins-reference). Tags exist up to at least v0.6.9 in listing tail (git tag | tail)
- CI (.github/workflows/test-plugin-install.yml) runs skill lint, version validation, evals with `--min-rank1 95`, hook regression tests with stubbed curl, `claude plugin validate .`, and an actual `claude plugin install ... --scope user` — workflow file. Actions are pinned by major tag (`actions/checkout@v6`), not commit SHA
- Conflicts with an existing ~/.claude: plugin skills/commands/agents are namespaced under the plugin name (docs: "Claude Code namespaces every component under it") — [plugins-reference `name`](https://code.claude.com/docs/en/plugins-reference); commands here refer to `agent-skills:<skill>` (.claude/commands/spec.md diff). So no clobbering of user's own CLAUDE.md/settings. A root CLAUDE.md in a plugin "isn't loaded as context" and `claude plugin validate` warns — same doc; the repo's own CLAUDE.md is explicitly scoped "for this repo, don't copy it" — CLAUDE.md
- Repo also ships parallel adapters (.codex-plugin, .gemini, .opencode, .agents, commands/*.toml) irrelevant to Claude Code; extra 9 .toml files in commands/ are declared in plugin `commands` path, but the docs say commands are `.md` files; whether .toml is ignored silently was not tested

### Inferences
- With plugin-only install nothing executes shell code from this repo at session time; the runtime risk is essentially prompt-level (skills are instructions the model follows), i.e., a malicious future commit would be prompt injection via skill text, mitigated by version pinning and review.
- Hooks would be the risky part only if the user copies them; I would advise skipping simplify-ignore.sh unless the block-protection feature is needed, and sdd-cache only if WebFetch cost matters.
- Compared with ECC (installer writing ~/.claude/settings.json, 94 commands, hooks), risk surface here is about one order smaller: no global config mutation, no auto-registered hooks.

### Gaps
- Did not execute install or any hook script; hook safety judged by reading only.
- Did not audit every one of 340 KB of skill text for embedded injection-like instructions; only greps for shell commands/permission keywords.
- Full git history/contributor list and open PR/issue security posture not checked (shallow clone).
- Whether `npx skills add` (third-party CLI) writes to ~/.claude/skills or other paths not inspected (not in this repo).
- Behavior of .toml files under a `commands` path in Claude Code not tested.

## Q2. Context footprint: always-loaded vs on-demand, measured

### Takeaway
Always-on cost under plugin install is about 2.5-3k tokens (25 skill descriptions ~2.2k tokens, plus 9 command and 4 agent one-liners); full bodies are on demand (avg 13.6 KB/3.4k tokens per skill, largest 21.5 KB/5.4k). Nothing like rules/CLAUDE.md/MCP definitions is injected.

### Cited Findings
- Claude Code behavior: "In a regular session, skill descriptions are loaded into context so Claude knows what's available, but full skill content only loads when invoked"; description + when_to_use capped at 1,536 chars per skill in the listing; listing budget scales at 1% of context window; `/skill-doctor` and `/doctor` show costs; `disable-model-invocation: true` removes a skill's description from context; `skillOverrides: "name-only"` lists a skill without description — [Claude Code skills docs](https://code.claude.com/docs/en/skills)
- Auto-compaction re-attaches invoked skills, first 5,000 tokens each, 25,000 tokens combined budget — same doc. So the largest skills (21.5 KB ~5.4k tokens) would be truncated on re-attachment
- Measured skill count: 25 skills (skills/*/SKILL.md). Sum of `description` chars = 8,691 (mean 348, max 880 constraint-driven-development, min 198). All under the 1,536 cap; the repo's own lint caps at 1,024 chars — scripts/lib/skill-lint.js
- Sum of frontmatter chars = 9,766. Estimated skill-listing cost ~2.2k tokens for descriptions (8,691/4) plus name/format overhead; ~2.4-2.7k tokens total (estimate)
- Commands: 9 markdown commands in .claude/commands (build, code-simplify, constraints, plan, review, ship, spec, test, webperf); description lengths 67-117 chars, ~850 chars total (~0.2k tokens). Per docs commands are merged with skills so descriptions likely appear in the listing, but I did not verify slash-command listing budget behavior for plugin commands
- Agents: 4 files (code-reviewer 3,977 chars; security-auditor 4,984; test-engineer 3,259; web-performance-auditor 12,244); descriptions 170-222 chars, ~756 chars (~0.2k tokens). Agent bodies load only when a subagent is spawned (general knowledge; docs page for agents not fetched today)
- On-demand sizes: total SKILL.md chars = 340,298 (~85k tokens if all were loaded, which does not happen). Largest: performance-optimization 21,526 chars/496 lines (~5.4k tokens); constraint-driven-development 20,956/311; code-review-and-quality 20,565/396; security-and-hardening 17,530/216; doubt-driven-development 16,557/243; test-driven-development 16,289/398; interview-me 15,034; smallest idea-refine 8,083 (plus idea-refine/examples.md 20,284 chars, frameworks.md 5,404, refinement-criteria.md 5,738 as supporting files). Median ~13.5 KB (~3.4k tokens)
- Typical workflow chain (/spec -> /plan -> /build -> /test -> /review) would load roughly 4-6 skills = ~15-25k tokens over a session (my estimate from the size list; not measured in a real session)
- Shared references/ (7 files, 67.3 KB total; orchestration-patterns 18.2 KB, security-checklist 14.4 KB, performance-checklist 13.1 KB) are read only if a skill points to them — file sizes measured
- using-agent-skills meta-skill is 9,782 chars (~2.4k tokens); only injected if user wires session-start.sh, which the plugin does not — hooks/session-start.sh header
- Not loaded as always-on context: root CLAUDE.md (4,094 chars), AGENTS.md (5,386), README (25,146), .claude/rules/skills-contributing.md — plugin root CLAUDE.md not loaded per [plugins-reference](https://code.claude.com/docs/en/plugins-reference); the rules file is a contributor guard (`paths: skills/**`) inside the repo only. No MCP servers defined (no .mcp.json)
- Skill authors follow a 500-line soft budget (lint warning) — scripts/lib/skill-lint.js; all measured SKILL.md are <=496 lines

### Inferences
- Always-on footprint ~3k tokens is <0.5% of a 1M window and ~1.5% of 200k. ECC's 293 skills would hit the 1%-of-context listing budget and get descriptions dropped; this bundle will not.
- Real risk is not idle cost but skill-body bloat when a skill fires (up to ~5k tokens each) and possible over-triggering because 25 descriptions are broad ("Use when ...").
- I did not recompute ECC numbers; comparison is on the counts given in the brief (68 agents/293 skills/94 commands).

### Gaps
- No real `/context` or `/doctor` reading from an installed session; all figures are chars/4 estimates.
- Token overhead of listing format per entry (name, plugin prefix) not measured.
- Agent-listing (Task tool description) context cost not verified against docs.

## Q3. Can Claude Code understand and safely extend this repo

### Takeaway
Yes, comfortably: small (25 skills, 4 agents, 9 commands, ~30 scripts/tests), uniform structure, explicit contributor rules, and CI validators that enforce them. Main friction is multi-tool duplication (Claude/Gemini/Codex/OpenCode adapters) that must stay in sync.

### Cited Findings
- Contributor docs: CONTRIBUTING.md (8.7 KB) with pre-flight checks before adding a skill; docs/skill-anatomy.md; CLAUDE.md / AGENTS.md define structure, conventions (frontmatter name+description, description starts third-person with "Use when...", required sections Overview/When to Use/Process/Common Rationalizations/Red Flags/Verification), and Boundaries — CLAUDE.md, .claude/rules/skills-contributing.md
- Validators in scripts/: validate-skills.js (thin wrapper over lib/skill-lint.js: name matches dir, description <=1024, has trigger phrase, required sections, supporting files rules), validate-commands.js (parity/description sync across `.claude/commands`, `commands/*.toml`, `.gemini/commands`), validate-reference-links.js, validate-artifact-paths.js, validate-versions.js (manifest version sync), run-evals.js (trigger/routing evals, CI gate `--min-rank1 95`), plus unit tests for each — scripts/ listing and workflow
- Evals: evals/cases (25 case files) and fixtures per skill; `--behavioral <skill>` for graded runs — CLAUDE.md commands section; evals/ dir listing
- Dependencies between components are light and enforced: skills cross-reference by name (lint warns on unknown cross-skill refs), commands map to skills 1:1, agents are standalone personas, docs/agents.md describes orchestration; hooks are standalone with own tests (sdd-cache-test.sh 11 KB, simplify-ignore-test.sh 17.8 KB, session-start-test.sh) — file listing
- CLAUDE.md states `npm test` is "Not applicable"; no package.json, so the tooling relies on plain `node scripts/...` and Node 24 in CI — CLAUDE.md, workflow
- CI is broad: ubuntu + macOS + Windows validators, hook regression tests, `claude plugin validate .`, real plugin install — workflow file
- Maintenance overhead: one change to a command touches 3 copies (.claude/commands/*.md, commands/*.toml, .gemini/commands/*.toml) with a parity validator; version must be synced across .claude-plugin/plugin.json, marketplace.json, root plugin.json (validate-versions.js) — measured from file contents (versions all 0.6.11)

### Inferences
- The validation harness gives Claude Code fast feedback for extensions (run `node scripts/validate-skills.js` etc.), which the ECC bundle at 293 skills would need to match; here the small size makes whole-repo comprehension in a single context feasible (~340 KB skill text total is too large to read at once, but each skill is self-contained).
- Safe forks for local additions should go to a separate plugin or ~/.claude/skills, not edits to this repo, to avoid merge churn against a fast-moving upstream (version 0.6.11 at Sep 2026).

### Gaps
- Did not run the validators or evals (no run-time check that CI passes at HEAD).
- Open issue/PR count, release cadence, bus-factor beyond single author not assessed.
- Quality/efficacy of the skills (whether they improve outcomes) not evaluated; evals only test trigger routing per repo docs.
