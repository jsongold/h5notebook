# addyosmani/agent-skills: maintenance status and community (as of 2026-09-30)

## 1. GitHub health metrics (compared with affaan-m/ECC)

### Takeaway
agent-skills (created 2026-02-15) has about 100k stars and 10.5k forks, roughly 37% of ECC's stars and 26% of its forks. It is much younger and smaller in commits (590 vs ~3,040) and contributors (~84 vs ~399). It is actively maintained and ships releases roughly every 1-2 weeks lately, which is faster than ECC's 2-4 weeks. The maintainer is effectively one person, as with ECC, but with a somewhat healthier contributor tail.

### Cited Findings
All figures below were pulled live via `gh api` on 2026-09-30 from [repo API](https://api.github.com/repos/addyosmani/agent-skills) unless noted.
- Stars 100,062; forks 10,512; watchers (subscribers) 518; MIT licence; not archived; owner type User (personal account, not an org); topics: agent-skills, antigravity, claude-code, codex, cursor, skills — [GitHub](https://github.com/addyosmani/agent-skills)
- Created 2026-02-15T20:20Z; first commit 2026-02-15T20:30Z; last push 2026-09-26T04:19Z; repo updated_at 2026-09-30 — GitHub API
- Commit count: 590 (from pagination of the commits endpoint, per_page=1, last page = 590) — GitHub API
- Contributors: 84 (contributors endpoint with anon=true, last page = 84 at per_page=1). Top: addyosmani 300 commits, nucliweb 44, federicobartoli 44, ayobamiseun 20, HMAKT99 13, mavericksea-ai 12 — GitHub API
- Owner's share: 300 of 590 commits is about 51% (contributors endpoint may exclude some commits, so treat it as approximate). Since 2026-08-01, addyosmani authored 82 of 203 commits (~40%); the rest came from ayobamiseun (17), mavericksea-ai (12), nucliweb (11) and others; 12 commits had no linked login — GitHub API
- Recent frequency: 203 commits since 2026-08-01, 105 since 2026-09-01, 25 in the 7 days since 2026-09-23 — GitHub API (paginated commits count)
- Most recent commits (2026-09-26) are merges of contributor PRs (#498, #571, #605, #607) plus a release bump commit by Addy Osmani, i.e. he acts as merger and release manager for community PRs — GitHub commits API
- Releases/tags (13 total): 0.5.0 2026-04-10; 0.6.0 2026-04-28; 0.6.1 2026-05-23; 0.6.2 2026-06-11; 0.6.3 2026-07-03; 0.6.4 2026-07-12; 0.6.5 2026-07-26; 0.6.6 2026-08-04; 0.6.7 2026-08-14; 0.6.8 2026-08-28; 0.6.9 2026-09-05; 0.6.10 2026-09-18; 0.6.11 2026-09-26. Gaps: about 3 weeks in May-June, then 1-2 weeks from July onward — GitHub releases API
- Issues: 58 open / 108 closed (about 166 total). PRs: 61 open / 377 closed (192 merged, so ~185 closed unmerged). The repo API `open_issues` field (119) equals open issues plus open PRs — GitHub search API
- ECC comparison (numbers supplied by the requester): 269,982 stars, 40,348 forks, ~3,040 commits, ~399 contributors, 83 open/734 closed issues, releases every 2-4 weeks. Ratios: stars 0.37x, forks 0.26x, commits 0.19x, contributors ~0.21x, closed issues 0.15x. Fork/star ratio 10.5% vs ECC 14.9%.
- Repo content: 25 skills per README (earlier press counted 19-24), slash commands (/spec, /plan, /build, /test, /review, /code-simplify, /ship), agent personas, hooks, evals, references, and manifests for Claude, Codex, Gemini and OpenCode — [README](https://github.com/addyosmani/agent-skills), directory listing via GitHub API

### Star growth (from dated third-party sources; each is a snapshot)
- ~14k stars "in days", June 2026 — [noqta](https://noqta.tn/en/blog/production-grade-agent-skills-authoring-guide-2026)
- 25k stars — search-result snippet, undated
- 27k stars on 2026-05-27 — [O'Reilly Radar](https://www.oreilly.com/radar/agent-skills/)
- 72.6k stars on 2026-07-08 — [productize.life](https://productize.life/blog/addy-osmani-skills/en)
- 100k on 2026-09-30 — GitHub API
- The sources conflict on timing (the noqta "14k in days, June" claim versus O'Reilly's 27k on May 27), so the early growth curve is uncertain. The July 72.6k to September 100k stretch is ~27k in about 12 weeks.

### Inferences
- Growth appears to be decelerating but still strong. Maintenance cadence is high and driven by community PR merges.
- Bus factor is around one, but 40-50% of commits come from other people, and there are repeat contributors (nucliweb, federicobartoli, ayobamiseun, mavericksea-ai). This looks more distributed than ECC's "effectively single maintainer" pattern, though I did not measure ECC's own commit distribution.
- Open PR backlog (61 open vs 192 merged) suggests PR review is a bottleneck.

### Gaps
- Exact ECC commit distribution was not gathered (not in scope).
- Star history timeline was not pulled from the stargazers API, so the dates above rely on third-party snapshots.

## 2. Maintainer and organisation

### Takeaway
Owned by Addy Osmani's personal account. He is a well-known Google engineer and author with a very large following, which explains the fast star growth. Effectively one lead maintainer with a community of PR contributors.

### Cited Findings
- GitHub profile: name Addy Osmani, company "Google", bio "Former Director at Google working on Gemini and Google Cloud", 53,742 followers, account created 2009-08-01 — [GitHub API users/addyosmani](https://api.github.com/users/addyosmani)
- Press describes him as a "Google Chrome senior engineer" — [search snippets, e.g. tproger/webreactiva](https://tproger.ru/news/addy-osmani-zawil-senior-inzhenernuyu-disciplinu-v-skilly-dlya-ai-a). This conflicts with his GitHub bio (former Director, Gemini/Cloud); the press descriptions are likely stale or imprecise.
- Author publishes on his blog and Substack, and co-published the announcement in O'Reilly Radar (2026-05-27) — [O'Reilly Radar](https://www.oreilly.com/radar/agent-skills/), [addyosmani.com](https://addyosmani.com/blog/agent-skills/)
- The repo is a personal project, not under a company org. The README frames the skills as encoding Google-style engineering practices — [O'Reilly Radar](https://www.oreilly.com/radar/agent-skills/)

### Inferences
- Personal brand plus O'Reilly and Substack distribution likely accounts for the reach, more than any organisation support.

### Gaps
- Current employer status is ambiguous (GitHub "Google" company field vs "Former Director" bio). Not verified against a primary source such as LinkedIn.

## 3. Distribution: npm, marketplace, adoption numbers

### Takeaway
Distribution is via the `npx skills` CLI (Vercel Labs open skills tool) and a Claude Code plugin marketplace entry. It does not publish its own npm package that I could verify, and no install counts were found.

### Cited Findings
- README install: `npx skills add addyosmani/agent-skills` (skills CLI at github.com/vercel-labs/skills, described as installing into 70+ agents); per-skill installs supported; Claude Code marketplace: `/plugin marketplace add addyosmani/agent-skills` then `/plugin install agent-skills@addy-agent-skills` — [README](https://github.com/addyosmani/agent-skills)
- Repo contains `.claude-plugin/marketplace.json`, `.codex-plugin`, `.gemini`, `.opencode`, `plugin.json` (name agent-skills, version 0.6.11, matching the latest tag) — GitHub API contents
- npm package named `agent-skills` exists (v1.0.4) but is maintained by an unrelated account (qwpm590@gmail.com); no evidence it is this project — `npm view agent-skills`
- Third-party directories list it: agenticskills.io, vibeindex.ai, landscape.jimmysong.io, skillrank.it.com — search results

### Inferences
- Because installs go through `npx skills` from GitHub and the plugin marketplace, no download counter is public. Stars and forks are the only adoption proxies; the 10.5k forks are a notable signal.

### Gaps
- No npm download or marketplace install numbers exist or were found.

## 4. Press, blog and community reception

### Takeaway
Reception is broadly positive in blogs and aggregators. Criticism found is mild: size and the risk of skills bloat. No HN or Reddit threads could be located via search, and no X data was reachable.

### Cited Findings
- Praise: "the most comprehensive of the set, 24 skills covering from spec all the way to ship"; strong on web-specific skills (browser testing with DevTools, performance); anti-rationalization tables — [productize.life review, 2026-07-08](https://productize.life/blog/addy-osmani-skills/en)
- Caution: "it is large, so do not enable every skill at once" — [productize.life](https://productize.life/blog/addy-osmani-skills/en)
- Contrast with typical skills: "most agent skills floating around GitHub are someone's half-finished experiment" (implicit praise) — [noqta](https://noqta.tn/en/blog/production-grade-agent-skills-authoring-guide-2026)
- Value described as exit criteria rather than the markdown itself — [Developers Digest](https://www.developersdigest.tech/blog/agent-skills-production-checklist)
- General skills-bloat concern (not specific to this repo): large skill packs can be skimmed or ignored by agents — same Developers Digest article
- Author's own framing: "Process over prose. Workflows over reference. Steps with exit criteria over essays" — [O'Reilly Radar](https://www.oreilly.com/radar/agent-skills/)
- Coverage in Russian (tproger), Spanish (webreactiva) and LinkedIn posts ("Google's ..." style hype posts) — search results

### Inferences
- Much coverage is promotional or aggregator content built on star counts, so the reception signal is weaker than the star count suggests. The lack of critical primary discussion may reflect limited search reach, not real absence.

### Gaps
- No Hacker News, Reddit or X discussion found; search returned none. Recommend manual check of HN Algolia and Reddit.
- No independent evidence about actual effectiveness (no benchmark data found; the repo has an `evals` directory that was not inspected).
