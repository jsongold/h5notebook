# ECC (affaan-m/ECC, npm ecc-universal): maintenance status and community (as of 2026-09-30)

Note: all "gh api" figures were pulled live on 2026-09-30 from https://api.github.com/repos/affaan-m/ECC (cited as [GitHub API]). npm figures from registry https://registry.npmjs.org/ecc-universal and https://api.npmjs.org/downloads (cited as [npm]).

## GitHub metrics (stars, forks, contributors, commits, issues/PRs, releases, first commit)

### Takeaway
Very large (about 270k stars, 40k forks), created 2026-01-18, extremely active (about 3,040 commits in about 8.4 months, pushed the day of the check), roughly 2-4 week release cadence, but effectively one maintainer with a large open PR/issue backlog.

### Cited Findings
- Stars 269,982; forks 40,348; watchers (subscribers) 1,351; MIT license; not archived; homepage https://ecc.tools; size ~54 MB — [GitHub API](https://api.github.com/repos/affaan-m/ECC)
- Repo created 2026-01-18T00:51:51Z; first commit 2026-01-18T01:49:33Z, "Initial release: Complete Claude Code configuration collection" (9 agents, 9 commands, 8 rules, 7 skills, 15 MCP configs; message cites "10+ months of daily usage" and the hackathon win) — [GitHub API commits, last page](https://api.github.com/repos/affaan-m/ECC/commits?per_page=1&page=3040)
- Total commits about 3,040 (pagination last page, per_page=1). Latest commit 2026-09-30T01:24:05Z, "feat(pi): curated pi/core skills+prompts profile ... 2.2.2 release sync (#3264)", noting 293 skills and 94 commands at root and "a two-week cadence in CONTRIBUTING.md" — [GitHub API](https://api.github.com/repos/affaan-m/ECC/commits?per_page=1)
- Commit frequency: about 407 commits since 2026-09-01 and about 172 since 2026-09-23 (page counts with since= filter, per_page=1); implies roughly 13/day in Sept, about 25/day in the last week. Includes bot/merged PR commits — [GitHub API](https://api.github.com/repos/affaan-m/ECC/commits?since=2026-09-01T00:00:00Z&per_page=1)
- Contributors: about 399 (incl. anon; last page of contributors per_page=1). Top: affaan-m 1,749 commits; haelyra 200; dependabot[bot] 65; pangerlkr 47; gaurav0107 28 — [GitHub API](https://api.github.com/repos/affaan-m/ECC/contributors?per_page=5)
- Issues: 83 open / 734 closed. PRs: 133 open, 2,175 closed (1,133 merged, so about 1,040 closed unmerged). (GitHub's open_issues_count field says 216 = open issues + open PRs) — [GitHub search API](https://api.github.com/search/issues?q=repo:affaan-m/ECC+is:pr+is:merged)
- Most-commented items are contributor feature PRs (e.g. #730 C# support, 99 comments; #2766 docs governance system, 122; #3097 antigravity hooks, 110 comments, open) — [GitHub API issues](https://api.github.com/repos/affaan-m/ECC/issues)
- Releases (GitHub): v1.8.0 2026-03-05; v1.9.0 2026-03-21; v1.10.0 2026-04-05; v2.0.0-rc.1 2026-05-25; v2.0.0 2026-06-10; v2.1.0 2026-07-27; v2.2.0 2026-08-28; v2.2.1 2026-09-08 (latest release). 18 tags total. Commit of 2026-09-30 says 2.2.2 is synced in-repo (no release yet at check time) — [GitHub API releases](https://api.github.com/repos/affaan-m/ECC/releases)
- Sponsors listed in README: CodeRabbit, Greptile, Moonshot AI (Kimi), Itô Markets, SerpApi; past: Atlas Cloud. README states "a single maintainer ships weekly across 7 harnesses" and funds work via GitHub Sponsors and "ECC Pro" (hosted GitHub App for private repos) — [README](https://github.com/affaan-m/ECC#readme)

### Inferences
- Star count appears to have grown from about 82k (Medium article, date unknown, probably Feb-Mar 2026) to 153k-214k (36kr / Italian blogs) to 270k now; sustained rather than a spike-and-decay. Some non-trivial share of the 1,040 closed-unmerged PRs are likely rejected or stale contributions, suggesting heavy inbound volume (may include AI-generated PRs; not verified).
- Open PR backlog (133) with 100+ comment threads suggests maintainer review bandwidth is the bottleneck.

### Gaps
- Star-growth timeline (dates per milestone) not pulled; would need stargazers pagination. Whether stars are organic vs inflated is not evidenced anywhere I found.
- Exact contributor count excludes verification of how many are one-time contributors.

## Maintainer and organization backing

### Takeaway
Single-maintainer, personal-account repo (owner type User) by Affaan Mustafa; no org backing of the repo itself, but monetized via sponsors and ECC Pro; the maintainer now runs a separate startup (Itô Markets) that is also a listed sponsor.

### Cited Findings
- Owner is a User account (affaan-m), not an org; profile: Affaan Mustafa, NYC, 11.6k followers, account created 2023-02-04, bio "The Agentic Exchange for Compute @Ito-Markets ... OSS meta-harness for AI agents @ECC-Tools"; site affaanmustafa.com — [GitHub API user](https://api.github.com/users/affaan-m)
- Orgs: ECC-Tools (org "ECC Tools", created 2026-01-25) and Ito-Markets — [GitHub API](https://api.github.com/users/affaan-m/orgs)
- Affaan Mustafa and David Rodriguez won the Anthropic x Forum Ventures hackathon (NYC, Sept 2025) with zenith.chat / "PMF Probe", built with Claude Code; README says the same — [README](https://github.com/affaan-m/ECC#readme); [36kr](https://eu.36kr.com/en/p/3823966768943239)
- Prize reported as about $15,000 platform/API credits (not cash) — [36kr](https://eu.36kr.com/en/p/3823966768943239); [Medium/tentenco snippet via search](https://medium.com/@tentenco/everything-claude-code-inside-the-82k-star-agent-harness-thats-dividing-the-developer-community-4fe54feccbc1)
- README also cites a Claude Code Hackathon (Cerebral Valley x Anthropic, Feb 2026) for a sub-project (1282 tests) — [README](https://github.com/affaan-m/ECC#readme)
- npm package author field: "Affaan Mustafa (https://x.com/affaan)"; git email me@affaanmustafa.com — [npm](https://registry.npmjs.org/ecc-universal)

### Inferences
- Bus factor is about 1 (affaan-m has 1,749 of about 3,000 commits; next contributor 200). The maintainer's attention is split with Itô Markets (a compute exchange) and ECC Tools; the README promotes Itô as "preferred compute sponsor", a mild conflict-of-interest/commercialization signal.

### Gaps
- Detailed personal background (education, prior employers) not verified from primary sources; only the profile and hackathon claims above.

## npm downloads and publish history

### Takeaway
npm downloads are small relative to GitHub stars: about 38.6k/month and 12k/week; the package is one of several distribution channels (plugin marketplace `ecc@ecc`, git clone/install script).

### Cited Findings
- npm package created 2026-02-11; 13 versions listed. dist-tags latest/staged 2.2.1, next 2.0.0-rc.1. Publish times: 1.0.0 2026-02-11; 1.6.0 02-27; 1.7.0 02-27; 1.8.0 03-05; 1.9.0 03-21; 1.10.0 04-15; 2.0.0-rc.1 05-26; 2.0.0 06-10; 2.1.0 07-27; 2.2.0 08-27; 2.2.1 09-08 (last modified) — [npm](https://registry.npmjs.org/ecc-universal)
- Downloads: 38,600 for 2026-08-30 to 2026-09-28; 12,041 for 2026-09-22 to 09-28; 112,884 total for 2025-03-30 to 2026-09-30 — [npm downloads API](https://api.npmjs.org/downloads/point/last-month/ecc-universal)
- npm publish lags GitHub tags by days to weeks (e.g., v1.10.0 GitHub 04-05 vs npm 04-15; 2.2.0 GitHub 08-28 vs npm 08-27 UTC) — compare GitHub releases above with npm.

### Inferences
- About 113k cumulative npm downloads vs 270k stars/40k forks shows most users install via plugin marketplace or clone, so npm is not a good adoption proxy.

### Gaps
- Plugin marketplace install counts unavailable. Dependents count not checked.

## Rename history

### Takeaway
Repo was originally affaan-m/everything-claude-code and is now affaan-m/ECC; old URL still resolves to the same repo (redirect). Exact rename date not determined.

### Cited Findings
- `gh api repos/affaan-m/everything-claude-code` returns full_name affaan-m/ECC (redirect) with the same star count — [GitHub API](https://api.github.com/repos/affaan-m/everything-claude-code)
- Three public identifiers now: repo affaan-m/ECC, plugin id `ecc@ecc` (short so slash-command namespaces fit strict validators; the older long marketplace id is a "legacy alias"), npm `ecc-universal` — [search summary of README/docs](https://github.com/affaan-m/ECC)
- Third-party posts still refer to "Everything Claude Code (ECC)", e.g. [Pasquale Pillitteri](https://pasqualepillitteri.it/news/4877/everything-claude-code-ecc-agent-harness), [DataCamp tutorial](https://www.datacamp.com/vi/tutorial/everything-claude-code)
- Repo description has evolved from "configuration collection" to "The agent harness performance optimization system ... for Claude Code, Codex, Opencode, Cursor and beyond" — [GitHub API](https://api.github.com/repos/affaan-m/ECC)

### Gaps
- Rename date not found (would require the repo renamed events / commit history of README; not checked).

## Press, adoption, and hackathon coverage

### Takeaway
Wide, mostly promotional/tutorial coverage (36kr, DataCamp, Medium, Italian/Portuguese/Japanese blogs) calling it the most-starred/fastest-growing Claude Code config repo.

### Cited Findings
- 36kr (Chinese tech media, EU edition) profile: hackathon win, open-sourced in January under MIT, "150,000 stars"; article contains no criticism — [36kr](https://eu.36kr.com/en/p/3823966768943239)
- Medium (tentenco): "Inside the 82K-Star Agent Harness That's Dividing the Developer Community" — calls it most-starred Claude Code config repo, 10,700 forks then — [Medium](https://medium.com/@tentenco/everything-claude-code-inside-the-82k-star-agent-harness-thats-dividing-the-developer-community-4fe54feccbc1) (page returned 403 to fetch; only search snippet seen)
- Other coverage: DataCamp tutorial; Pasquale Pillitteri "214,000 stars and the open-source OS"; Zenn (Japanese) article; claudeskills.info hackathon/eval-driven post; redreamality.com "ECC: A Verifiable Harness Layer" — [DataCamp](https://www.datacamp.com/vi/tutorial/everything-claude-code); [Zenn](https://zenn.dev/hikaru_tell/articles/b62e715c0e41d2); [redreamality](https://redreamality.com/blog/ecc-agent-harness-optimization/)

### Gaps
- No Anthropic endorsement or official mention found. No enterprise adopters named.

## Criticism and controversy

### Takeaway
I did not find substantive, citable criticism from HN/Reddit/X of ECC specifically; the only signal is the Medium headline ("dividing the developer community") and generic context-bloat concerns that plausibly apply to a 293-skill/94-command bundle.

### Cited Findings
- Medium headline says ECC is "dividing the developer community", but body not retrievable (403) — [Medium](https://medium.com/@tentenco/everything-claude-code-inside-the-82k-star-agent-harness-thats-dividing-the-developer-community-4fe54feccbc1)
- A third-party guide suggests capping SessionStart context and setting `ECC_SESSION_START_CONTEXT=off` for small/local models rather than installing all 292 skills (paraphrased from search summary, unverified against page) — [redreamality](https://redreamality.com/blog/ecc-agent-harness-optimization/)
- The maintainer's own repo is moving to curated subsets (pi/core: 123 of 293 skills, 24 of 94 commands, "35,006 characters of skill description text"), an implicit acknowledgment of size/context cost — [commit c70874f](https://github.com/affaan-m/ECC/commit/c70874fae9eb0e5ad0365beb7e2955899fd1d30f)
- General context-rot/bloat literature: [MindStudio](https://www.mindstudio.ai/blog/context-rot-claude-code-skills-bloated-files) (not about ECC specifically)

### Inferences
- Likely critique vectors (unverified): breadth vs depth, context/token overhead, star-driven hype, single-maintainer sustainability, commercial ties (ECC Pro, Itô).

### Gaps
- HN, Reddit, X threads on ECC were not found by search (search tool returned generic results); should be checked directly (e.g. hn.algolia.com search for "everything-claude-code", r/ClaudeAI). Star authenticity not analysed.
