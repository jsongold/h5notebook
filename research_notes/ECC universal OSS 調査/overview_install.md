# ECC (affaan-m/ECC, npm: ecc-universal) - purpose, repo structure, installation

Sources fetched 2026-09-30 via `gh api` (raw files on main) and `npm view`. Repo main is ahead of npm: README/VERSION/plugin.json say 2.2.2, npm latest is 2.2.1.

## 1. What is ECC (name, author, goal, target tools)

### Takeaway
ECC = "Everything Claude Code", a MIT-licensed "agent harness performance optimization system" by Affaan Mustafa (bundle of agents, skills, hooks, rules, memory, security scanning). Claude Code is the primary/stable target; Codex has native plugin support; Cursor, OpenCode, Gemini, Zed etc. get capability-limited adapters.

### Cited Findings
- Name expansion: AGENTS.md title is "Everything Claude Code (ECC) — Agent Instructions"; describes it as a "production-ready AI coding plugin" with 68 agents, 293 skills, 94 commands, hook workflows. — [AGENTS.md](https://github.com/affaan-m/ECC/blob/main/AGENTS.md)
- GitHub description: "The agent harness performance optimization system. Skills, instincts, memory, security, and research-first development for Claude Code, Codex, Opencode, Cursor and beyond." Homepage https://ecc.tools. — [GitHub API repos/affaan-m/ECC](https://api.github.com/repos/affaan-m/ECC)
- Stats at fetch time: about 269,981 stars, primary language JavaScript, license MIT, last push 2026-09-30. — [GitHub API](https://api.github.com/repos/affaan-m/ECC)
- Author: Affaan Mustafa (x.com/affaan); npm maintainer listed as "cogsec <me@affaanmustafa.com>". — [npm registry](https://www.npmjs.com/package/ecc-universal), [plugin.json](https://github.com/affaan-m/ECC/blob/main/.claude-plugin/plugin.json)
- Goal (README): "Your agent can write code, but ECC gives it a coordinated engineering system and toolbox: it plans before it builds, verifies changes with tests, reviews its own work from a fresh context, remembers what matters, and turns repeated wins into reusable skills and workflows." Loop: `plan -> test -> implement -> review -> verify -> remember -> improve`. Tagline: "Optimize the context window. Persist everything else." — [README](https://github.com/affaan-m/ECC/blob/main/README.md)
- Contents table: 68 agents, 293 skills, 94 legacy command shims (commands moving to a skills-first surface), hooks/memory runtime (session summaries, continuous learning, instincts), selective rules, AgentShield scanning (prompts, hooks, MCP config, permissions, secrets). — [README](https://github.com/affaan-m/ECC/blob/main/README.md)
- Target tools: "works best with Claude Code today, has a supported Codex sync path, and provides capability-limited adapters for Cursor, OpenCode, Gemini, Zed, GitHub Copilot, Antigravity, Qwen, and other harnesses." Support matrix: Claude Code = Stable primary; Codex = Supported native plugin; Cursor, OpenCode = Beta; Copilot = instruction-only; Gemini/Zed/Antigravity/Qwen/Hermes/OpenClaw/Kimi/CodeBuddy/JoyCode = experimental/minimal. — [README Platform Support](https://github.com/affaan-m/ECC/blob/main/README.md)
- Background (README): author says he has used Claude Code since the experimental rollout, won the Anthropic x Forum Ventures hackathon (Sep 2025), configs "battle-tested across multiple production applications". — [README](https://github.com/affaan-m/ECC/blob/main/README.md)
- Business model: OSS "MIT forever"; ECC Pro = hosted GitHub App for private repos ("from $19/seat/mo"); sponsors listed (CodeRabbit, Greptile, Moonshot Kimi, Itô Markets, SerpApi). — [README](https://github.com/affaan-m/ECC/blob/main/README.md)
- Security notice: install only from official channels (repo, npm `ecc-universal` and `ecc-agentshield`, GitHub App, plugin slug `ecc@ecc`, ecc.tools); third-party mirrors may contain malware. — [README](https://github.com/affaan-m/ECC/blob/main/README.md)
- Three identifiers not interchangeable: GitHub `affaan-m/ECC`, Claude marketplace/plugin `ecc@ecc`, npm `ecc-universal`. npm releases are cut per version tag, not per commit. — [README naming note](https://github.com/affaan-m/ECC/blob/main/README.md)

### Inferences
- The repo was renamed/consolidated to short "ECC" (older name likely "everything-claude-code"; README says older posts show "the former long marketplace identifier"). Not verified beyond that README sentence.
- npm name "universal" reflects multi-harness intent; npm package first published 2026-02-11.

### Gaps
- Exact original repo name/rename date not checked.
- Star count is a moving number.

## 2. Repository layout

### Takeaway
Root is the single source of truth (agents/, skills/, commands/, rules/, hooks/, scripts/); per-harness dot-directories are adapters/packaging of the same content. Node.js scripts do install/repair/orchestration; a Rust `ecc2/` and Python dashboard also exist.

### Cited Findings
- README "What's Inside" tree: `agents/` (68 subagents), `skills/` (293 workflows loaded on demand), `commands/` (94 slash-command shims), `rules/` (opt-in common + language standards), `hooks/` (runtime automation), `scripts/` (install, repair, sync, orchestration, checks), `.claude-plugin/` (marketplace manifest: plugin.json, marketplace.json), `.codex/` (Codex reference config, agent roles), `.opencode/`, `.cursor/`, `docs/`. "The root is the source of truth. Platform adapters package or map these same workflows." — [README](https://github.com/affaan-m/ECC/blob/main/README.md)
- Actual top-level (GitHub contents API, counts of entries): `agents/` (68 files), `commands/` (94), `skills/` (293 dirs), `rules/` (23 entries incl. common, typescript-type language/framework dirs, README.md), `hooks/` (README.md, hooks.json, codex-hooks.json, hooks.metadata.json, memory-persistence/), `scripts/` (63 entries: ecc.js, install-apply.js, install-guided.js, setup.js, doctor.js, repair.js, uninstall.js, list-installed.js, sync-ecc-to-codex.sh, memory-mcp.mjs, claw.js, orchestrate-*, harness-audit.js etc.; lib/, ci/, codex/), `manifests/` (install-modules.json, install-profiles.json, install-components.json, context-packs, context-profiles, install-assets, pi-core.json), `docs/` (53 entries, translations, guides), `ecc2/` (Rust: Cargo.toml, src), `src/llm`, `mcp-configs/`, `contexts/`, `schemas/`, `plugins/`, `integrations/`, `legacy-command-shims/`, `scaffolds/`, `examples/`, `workflows/`, `docker/`, `tests/`, `config/`, `pi/`. — [GitHub contents API](https://api.github.com/repos/affaan-m/ECC/contents)
- Harness adapter dot-dirs: `.claude-plugin`, `.claude`, `.codex`, `.codex-plugin`, `.cursor`, `.opencode`, `.gemini`, `.zed`, `.kimi`, `.kiro`, `.qwen`, `.trae`, `.adal`, `.hermes`, `.openclaw`, `.codebuddy`, `.pi`, `.agents`. Also `.mcp.json`, `install.sh`, `install.ps1`, `agent.yaml`, `AGENTS.md`, `CLAUDE.md`, `SOUL.md`, `COMMANDS-QUICK-REF.md`, `TROUBLESHOOTING.md`, `ecc_dashboard.py`, `pyproject.toml`, `the-shortform/longform/security-guide.md`. — [GitHub contents API](https://api.github.com/repos/affaan-m/ECC/contents)
- `.claude-plugin/plugin.json`: name `ecc`, version 2.2.2, author Affaan Mustafa, license MIT, `userConfig` with `hooks_enabled` (bool, default true) and `hook_profile` (minimal/standard/strict, default standard). README warns contributors not to add a `"hooks"` field to plugin.json because Claude Code 2.1+ auto-loads `hooks/hooks.json` (duplicate-detection error; regression test enforces). — [plugin.json](https://github.com/affaan-m/ECC/blob/main/.claude-plugin/plugin.json), [README Requirements](https://github.com/affaan-m/ECC/blob/main/README.md)
- Rules: Claude Code plugins cannot distribute `rules`, so users copy them manually to `~/.claude/rules/ecc/`. — [README](https://github.com/affaan-m/ECC/blob/main/README.md)
- Key concepts documented: Agents, Skills, Hooks, Rules; hook controls via env vars; package-manager detection (`CLAUDE_PACKAGE_MANAGER`, `.claude/package-manager.json`, etc.). — [README](https://github.com/affaan-m/ECC/blob/main/README.md)

### Inferences
- The catalog is large (293 skills), so the plugin path loads a big catalog into context; README itself says use selective/manual profile when context footprint matters.

### Gaps
- Per-directory contents beyond names (e.g. what each of the 293 skills does) not enumerated; another researcher may cover.

## 3. Installation methods and exact commands

### Takeaway
Recommended: `npx ecc-universal setup` (guided wizard that drives Claude Code's own `claude plugin ...` CLI to install plugin `ecc@ecc`). Alternatives: native `/plugin` commands, Codex plugin commands, or `install.sh --profile ... --target ...` for manual/other harnesses. Pick one method per harness; don't stack.

### Cited Findings
- Prereqs: Node.js >= 18; Claude plugin setup also needs Git and Claude Code >= 2.1 on PATH. — [README Install ECC](https://github.com/affaan-m/ECC/blob/main/README.md)
- Recommended guided setup (README pins to repo version 2.2.2, which is NOT yet on npm; npm latest is 2.2.1): `npx ecc-universal@2.2.2 setup`. Equivalents: `pnpm dlx ecc-universal@2.2.2 setup`, `yarn dlx ...` (Yarn 2+), `bunx ecc-universal@2.2.2 setup`. Unpinned `npx ecc-universal setup` resolves to latest 2.2.1. — [README](https://github.com/affaan-m/ECC/blob/main/README.md), [npm](https://www.npmjs.com/package/ecc-universal)
- Wizard: inventories official marketplace and every native Claude install scope, then installs/updates/moves `ecc@ecc` to chosen scope; rerun to update, change scope, or change hook profile. Interactive choices (README Windows walkthrough): "Global user" scope, "Standard" hooks, confirm; then in Claude Code run `/plugin list` to verify `ecc@ecc`. — [README](https://github.com/affaan-m/ECC/blob/main/README.md)
- Non-interactive: `ecc setup --mode claude-plugin --scope user|project|local --hooks off|minimal|standard|strict --yes` (also `--move-scope`, `--dry-run`, `--json`). Scopes: user = global; project = shared project config (committed); local = private project config. — [scripts/setup.js](https://github.com/affaan-m/ECC/blob/main/scripts/setup.js)
- Multi-harness wizard: `npx ecc-universal@2.2.2 install --guided` (Claude Code, Codex, Kimi Code; one final confirmation). Automation: `npx ecc-universal@2.2.2 install --guided --harness claude --harness codex --harness kimi --claude-scope local --claude-hooks standard --profile core --yes`. Dry-run: `... install --guided --harness codex --dry-run`, `... install --profile core --target kimi --dry-run`. Kimi = managed project files under `./.kimi-code`. — [README](https://github.com/affaan-m/ECC/blob/main/README.md)
- Native Claude Code plugin (alternative, run inside Claude Code): `/plugin marketplace add https://github.com/affaan-m/ECC` then `/plugin install ecc@ecc`. Installs skills, agents, commands, plugin-managed hooks. After install `/ecc:configure-ecc` reconfigures. — [README Claude Code details](https://github.com/affaan-m/ECC/blob/main/README.md)
- Declarative alternative in `~/.claude/settings.json`: `"extraKnownMarketplaces": {"ecc": {"source": {"source": "github", "repo": "affaan-m/ECC"}}}` and `"enabledPlugins": {"ecc@ecc": true}`. — [README](https://github.com/affaan-m/ECC/blob/main/README.md)
- Rules (manual, since plugins can't ship them): `git clone https://github.com/affaan-m/ECC.git; cd ECC; mkdir -p ~/.claude/rules/ecc; cp -R rules/common ~/.claude/rules/ecc/; cp -R rules/typescript ~/.claude/rules/ecc/`. — [README](https://github.com/affaan-m/ECC/blob/main/README.md)
- Codex: `codex plugin marketplace add affaan-m/ECC`, `codex plugin add ecc@ecc`, `codex plugin list --json`, `node scripts/codex/check-plugin-cache.js`; refresh with `codex plugin marketplace upgrade ecc` + `codex plugin add ecc@ecc`; inside Codex `$configure-ecc`. Legacy (deprecated) `bash scripts/sync-ecc-to-codex.sh` copies/merges config into `~/.codex`; removal via `node scripts/ecc.js uninstall --legacy-codex-sync`. — [README Codex](https://github.com/affaan-m/ECC/blob/main/README.md)
- Manual/other targets (from a git clone): `./install.sh --profile minimal --target cursor|gemini|zed|antigravity|qwen|hermes|openclaw|kimi|codebuddy|joycode`; OpenCode: `npm install && npm run build:opencode && ./install.sh --profile full --target opencode --enable-hooks`. Windows: `.\install.ps1 --profile minimal --target claude`. — [README Other agents](https://github.com/affaan-m/ECC/blob/main/README.md)
- Low-context/no-hooks: `npx ecc-universal@2.2.2 install --profile minimal --target claude` (excludes `hooks-runtime`); `./install.sh --profile core --no-hooks --target claude`; add hooks later `./install.sh --target claude --modules hooks-runtime --enable-hooks`. Any install materializing the hook runtime requires explicit `--enable-hooks` or `--no-hooks`, else installer prints what hooks can do and stops before writing. — [README Advanced Install](https://github.com/affaan-m/ECC/blob/main/README.md)
- Warning: `npx ecc-install ...` is wrong; `ecc-install` is a bin inside `ecc-universal`, not a separate package. Extra CLI: `consult`, `doctor`, `list-installed`, `repair`, `uninstall [--dry-run]`, `feedback`. — [README](https://github.com/affaan-m/ECC/blob/main/README.md)
- Do-not-stack: OK = Claude plugin + Codex native plugin; AVOID = Claude plugin + full Claude manual install; AVOID = Codex sync + Codex marketplace plugin. — [README](https://github.com/affaan-m/ECC/blob/main/README.md)
- Uninstall/reset: `npx ecc-universal@2.2.2 list-installed | doctor | repair | uninstall --dry-run | uninstall` from the same project dir; ECC only removes files recorded in its install-state. Plugin users remove plugin in Claude Code, and delete manually copied rule folders. — [README Reset/Uninstall](https://github.com/affaan-m/ECC/blob/main/README.md)

### What `setup` does to the machine (from source, scripts/setup.js and scripts/lib/claude-plugin-setup.js)
- Requires `git` and `claude` on PATH; shells out to Claude Code CLI (`claude plugin marketplace list --json`, `claude plugin list --json`). If marketplace `ecc` exists it must be the official affaan-m/ecc source, else it refuses (collision protection); it runs `claude plugin marketplace update ecc`, otherwise `claude plugin marketplace add <official URL> --scope <scope>`, verifies with `marketplace list --json`. Then `claude plugin update ecc@ecc --scope <scope>` if installed, else `claude plugin install ecc@ecc --scope <scope>`. Scope move = install+verify at destination before removing the source. — [claude-plugin-setup.js](https://github.com/affaan-m/ECC/blob/main/scripts/lib/claude-plugin-setup.js)
- Writes to Claude user settings `<configDir>/settings.json` (i.e. `~/.claude/settings.json`) atomically: sets `pluginConfigs["ecc@ecc"].options` = `{hooks_enabled: hooks !== 'off', hook_profile: hooks==='off' ? 'standard' : hooks}` when `--hooks` given. Invalid existing settings JSON aborts with error. — [claude-plugin-setup.js](https://github.com/affaan-m/ECC/blob/main/scripts/lib/claude-plugin-setup.js)
- Side effect worth flagging: unless the user already has `includeCoAuthoredBy` or `attribution.commit/pr` set, setup writes `"includeCoAuthoredBy": false` to `~/.claude/settings.json` (disables Claude's Co-Authored-By trailer on commits/PRs; "ECC-managed installs default that off"). — [claude-commit-attribution.js](https://github.com/affaan-m/ECC/blob/main/scripts/lib/claude-commit-attribution.js), [claude-plugin-setup.js](https://github.com/affaan-m/ECC/blob/main/scripts/lib/claude-plugin-setup.js)
- Plugin files themselves are placed by Claude Code's plugin manager (plugin cache under ~/.claude/plugins/), not by ECC; the `user`/`project`/`local` scope determines which Claude settings file records enablement. Manual (non-plugin) Claude installs (`install.sh --target claude`) place skills at `~/.claude/skills/<skill-name>/` (or `.claude/skills/...` for claude-project) and track files in an install-state. — [README Advanced Install](https://github.com/affaan-m/ECC/blob/main/README.md)
- `--dry-run` reports without changing anything. — [setup.js](https://github.com/affaan-m/ECC/blob/main/scripts/setup.js)

### Inferences
- Exact plugin cache path (~/.claude/plugins/...) is inferred from general Claude Code behavior, not read from ECC source.
- The `includeCoAuthoredBy:false` write is a behavior change users may not expect from "setup"; worth surfacing in the report.

### Gaps
- Did not read install-apply.js / install.sh to enumerate every file written for `--target claude` manual installs (hooks written into settings.json, install-state file location).
- Did not run `npx ecc-universal setup` (would modify machine); behavior taken from source at main (2.2.2), which may differ slightly from published 2.2.1.
- Value of OFFICIAL_MARKETPLACE_URL constant not read (README uses https://github.com/affaan-m/ECC).

## 4. License, language, npm package contents

### Takeaway
MIT; JavaScript (Node >= 18) core with Shell/Python/Rust/TypeScript pieces; npm package ships ~2,615 files / 17.7 MB unpacked with 6 bins and 4 runtime dependencies.

### Cited Findings
- License MIT (LICENSE; README badge; package.json). GitHub primary language: JavaScript. README badges list Shell, TypeScript, Python, Go, Java, Perl, Markdown (much are skills/examples). Repo also has Rust crate `ecc2/` and Python `ecc_dashboard.py`. — [GitHub API](https://api.github.com/repos/affaan-m/ECC), [README](https://github.com/affaan-m/ECC/blob/main/README.md)
- npm `ecc-universal@2.2.1`: dist-tags latest=2.2.1, staged=2.2.1, next=2.0.0-rc.1; versions 1.0.0, 1.6.0, 1.7.0, 1.8.0, 1.9.0, 1.10.0, 2.0.0-rc.1, 2.0.0, 2.1.0, 2.2.0, 2.2.1; created 2026-02-11; 2.2.1 published 2026-09-08. — `npm view ecc-universal --json`
- npm description: "Harness-native agent operating system for Codex, OpenCode, Cursor, Gemini, Claude Code, and terminal workflows - skills, hooks, rules, MCP conventions, and operator control-plane patterns". Engines node >=18. Dependencies: ajv 8.20.0, sql.js 1.14.2, js-yaml 4.3.1, @iarna/toml 2.2.5. Unpacked size 17,658,942 bytes, 2,615 files. — `npm view ecc-universal@2.2.1`
- bin entries: `ecc` and `ecc-universal` -> scripts/ecc.js; `ecc-install` -> scripts/install-apply.js; `ecc-memory-mcp` -> scripts/memory-mcp.mjs; `ecc-plan-canvas` -> scripts/plan-canvas.js; `ecc-control-pane` -> scripts/control-pane.js. — [package.json](https://github.com/affaan-m/ECC/blob/main/package.json)
- package.json `files` whitelist includes adapter dirs (.claude-plugin/, .codex/, .codex-plugin/, .cursor/, .gemini/, .opencode/ and dist, .kimi/, .pi/, .hermes/, .openclaw/, .qwen/, .zed/, .adal/, .agents/), agents/, commands/, hooks/, manifests/, mcp-configs/, plugins/ecc/, rules/, schemas/, install.sh, install.ps1, AGENTS.md, agent.yaml, .mcp.json, selected scripts/ and translated docs (de-DE, ja-JP, ko-KR, pt-BR, pl, ru, tr, uk-UA, vi-VN, zh-CN, zh-TW). `prepack` runs `npm run build:opencode`. `skills/` was not visible in the portion of `files` read (list was truncated at ~120 lines). — [package.json](https://github.com/affaan-m/ECC/blob/main/package.json)
- `welcome` script message on install mentions `ecc typescript`, Itô compute sponsorship. — [package.json](https://github.com/affaan-m/ECC/blob/main/package.json)

### Inferences
- Version drift (repo 2.2.2 vs npm 2.2.1) means README commands pinned to `@2.2.2` will fail with ETARGET until 2.2.2 is published; use unpinned or `@2.2.1` on 2026-09-30 (unverified by running).

### Gaps
- Did not confirm `skills/` appears in the npm tarball (files list truncated in my read); did not check weekly download counts.
