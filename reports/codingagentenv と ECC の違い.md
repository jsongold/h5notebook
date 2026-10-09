# CAE の安定感は事故面の小ささであり、成熟ではない

目的：自分の repo codingagentenv (CAE) が「affaan-m/ECC より安定」という印象がどの意味で正しいかを、addyosmani/agent-skills を第三の基準にして、コードレベルの事実で軸ごとに判定する。

**結論：印象は「壊れたときの被害範囲（blast radius）」「副作用」「installer の書き込み範囲」「常時コンテキスト量」の 4 軸では正しい。「成熟・運用実績」「強制力」「skill 内容の検証」の 3 軸では正しくない。** CAE の安定感の主因は、部品が少なく、壊れうる強制機構をそもそも持たないことにある（機能量の差）。同じ機能を安定して実装している証拠ではない。hook は 2 本で書き込み・ネットワークが無く、ECC の fail-closed な Bash dispatcher や Stop 時のソース書き換えに相当する構造が無い。一方で CAE は稼働約 10 日・293 commit の若い repo で、skill の中身を検証する仕組みが 0 件、CI は最新の guardrail（scope-check）のテストを回しておらず、installer には部分適用で止まる欠陥がある。agent-skills は「実行コードをほぼ持たず、代わりに検証を厚くする」第三の型で、CAE に足りないのはこちらの検証層である。トークン数はすべて文字数/4 などの概算で、`/context` による実測はしていない。

## 判定は 10 軸に分けて行う：事故面は CAE、検証は agent-skills が強い

「安定」は 1 つの性質ではないので、軸ごとに分けて判定する。CAE 側の事実は `/Users/yasumasa_takemura/projects/codingagentenv`（HEAD 17fcab9 時点）の実ファイルと一時ディレクトリでの追加実験、ECC は commit c70874fa、agent-skills は commit 2686b620（0.6.11）の読解に基づく。ECC・agent-skills の詳細出典は前回調査 ([ECC](https://github.com/affaan-m/ECC), [agent-skills](https://github.com/addyosmani/agent-skills)) を再利用している。

| 軸 | CAE | ECC | agent-skills | 判定（CAE の印象は） |
|---|---|---|---|---|
| 事故面（blast radius） | hook 2 本。ブロックするのは `gh issue/pr create` の本文形式のみ | hook 24 個・7 イベント、`.*` matcher で全ツール呼び出しに介入 | hook は 4 スクリプトで自動登録なし（手貼り） | 正しい（ただし agent-skills には劣後） |
| 失敗時の挙動 | 想定外入力は exit 0（fail-open）。環境不備（node 不在）も実質 fail-open | `pre-bash-dispatcher.js` は例外・入力超過で exit 2（fail-closed）。ECC 側のバグが全 Bash 停止になる | session-start は jq 無しで通知して exit 0。sdd-cache は 304 で意図的に exit 2 | 正しい |
| 副作用 | hook はファイル書き込み・ネットワークとも 0 | Stop 時に Biome/Prettier `--write` でソースを書き換え、`claude -p` で transcript 要約、`~/.gateguard` 等に状態保存 | simplify-ignore が Read 時にソースをプレースホルダ化し Stop で復元（異常終了で残りうる） | 正しい |
| installer の書き込み範囲 | symlink、自 hook のみの追記、CLAUDE.md のマーカー区間 | ファイルをコピー（full で約 1,100 ファイル）、`includeCoAuthoredBy: false` を既定で書く | installer 無し（`/plugin` のみ） | ECC 比では正しい。agent-skills 比では劣後 |
| installer の安全性（冪等・復旧） | 冪等、uninstall は自分の分だけ除去、status で drift 検出。dry-run 無し、部分適用に rollback 無し、bak 無制限 | ID 単位マージ、ロック、install-state、hook-consent、dry-run。ただし版数ずれ・tarball 欠落の報告あり | 該当なし | 部分的に正しい（設計は良いが、失敗系が弱い） |
| 強制力 | 機械的に止めるのは 1 点のみ。規範の大半は CLAUDE.md の文章 | GateGuard、config-protection、`--no-verify` 阻止、commit 前の内容走査など | 強制なし。「基準を弱める変更」は skill が検知を促すのみ | 「壊れにくい」のは強制が無いから。安定性としては割り引く |
| skill 内容の検証 | 0 件。`status` は symlink の有無しか見ない | validate-skills/commands/agents/hooks など約 10 種 + catalog drift 検査。ただし skill 間参照 51 件中 14 が dangling | skill-lint、routing eval（rank1 95% 以上を CI 門番）、3 OS の CI、実 plugin 導入 | 正しくない |
| テスト・CI | installer と SessionStart に 86 check、今回すべて通過。ただし CI は 2 本のみ実行 | `npm test` 連鎖 + coverage 閾値（80%/79%）+ workflow 検査 | 単一 workflow で validator・eval・hook 回帰・plugin install | 部分的に正しい（テストの質は高いが、CI の穴とスコープが狭い） |
| 常時コンテキスト（概算） | description 9 本で約 330〜1,300 token、グローバル CLAUDE.md が約 1〜2k token | minimal 約 17k、full 約 37k token（skill 説明が 1% 予算を超過する見込み） | 約 2.5〜3k token | 正しい（最小クラス） |
| 成熟・変更速度 | 稼働約 10 日、293 commit、9/29 は 106 commit、hook 範囲を数回作り直し、利用者は本人のみ | 約 3,040 commit（約 8.4 か月）、直近は 1 日約 25 commit、リリース 2〜4 週、事実上 1 人が保守 | 590 commit、リリース 1〜2 週、貢献の約 40〜50% が他者 | 正しくない |

## 「壊れにくい」の正体：強制機構を持たないことと、書き込み範囲を絞ったこと

CAE の 2 本の hook は、実行時の失敗モードが構造的に小さい。SessionStart（`hooks/harness-session-start.sh`）は `.claude/handoff/*.md` を読んで先頭 60 行を注入するだけで、`set -u` のみ・常に `exit 0`、書き込みもネットワークも無い。PreToolUse（`hooks/harness-scope-check.ts`）は installer が付ける `if: Bash(gh *)` により gh コマンド以外では node を起動せず、`gh (issue|pr) create` の本文に変更一覧の行が無いときだけ exit 2 で止める。JSON が壊れていても対象外でも exit 0 である。これに対し ECC は、`pre-bash-dispatcher.js` が入力サイズ超過でも dispatcher 例外でも exit 2 で止めるため、ECC 側のコード不具合が「全 Bash 拒否」の障害に直結する ([pre-bash-dispatcher.js](https://github.com/affaan-m/ECC/blob/main/scripts/hooks/pre-bash-dispatcher.js))。`stop-format-typecheck.js` は Edit ツールを経ずにソースを書き換え、別の formatter 設定を持つ repo で意図しない差分を出しうる ([stop-format-typecheck.js](https://github.com/affaan-m/ECC/blob/main/scripts/hooks/stop-format-typecheck.js))。fail-closed と副作用の 2 点は、ここを直さない限り CAE に対する ECC の構造的な劣勢である。

installer も同じ向きに差がある。CAE の `bin/codingenv`（`set -eu`、依存は jq のみ）が書くのは、skill・hook・harness.json・tools の symlink、settings.json への自 hook の追記、`~/.claude/CLAUDE.md` のマーカー区間だけで、`includeCoAuthoredBy` など無関係なキーは書かない（grep で確認）。uninstall は自分の symlink と自分の hook だけを除き、status は drift があれば exit 1 を返す。ECC の installer は `apply.js` が `copy-file` でルール・agent・command・script・skill をコピーし、既定で `"includeCoAuthoredBy": false` も書く ([claude-commit-attribution.js](https://github.com/affaan-m/ECC/blob/main/scripts/lib/claude-commit-attribution.js))。ただし ECC も ID 単位のマージ、ロック、install-state、hook-consent（実行前に能力を列挙）を持ち、設計の水準は低くない。CAE が優位なのは「触る範囲の狭さ」であり、失敗時の扱いの丁寧さではない。agent-skills は installer 自体が無く settings.json を書き換えるコードが無いので、この軸では最も安全である。

ただし、この安定は強制力の低さと表裏である。CAE が機械的に止めるのは PR/Issue 本文の形式だけで、編集前確認、設定改変の阻止、`--no-verify` 阻止に相当する hook は無い。CLAUDE.md の規範（main は実装しない、1 PR 1 変更など）は LLM への依頼文に留まる。ECC の GateGuard や config-protection は、止めるぶん誤検知（heredoc を命令と誤認する #3152 など）も出すので、「壊れる」ことと「守る」ことは同じ根から出ている ([#3152](https://github.com/affaan-m/ECC/issues/3152))。したがって「CAE の方が壊れにくい」は正しいが、「CAE の方が安全に守れている」は成り立たない。

## 検証の層では CAE が最も薄く、agent-skills が最も厚い

CAE のテストは、installer と SessionStart に対しては実用的な質である。`test/harness.test.sh`（ok 55）と `test/hooks.test.sh`（ok 31）、`test/scope-check.test.sh`（ok 13）は、実機に触れず一時ディレクトリで動き、今回の実行で失敗は 0 だった。手書きの旧セクション、退役 hook、非 symlink 衝突など、過去に起きた不具合が test に落ちている。しかし、skill の中身を対象にしたテスト・lint は 0 件で（`SKILL` / `frontmatter` の grep が 0 件）、`codingenv status` は symlink の有無しか見ない。agent-skills は `skill-lint.js` が「name とディレクトリ名の一致」「description に when-to-use の trigger を含む」「必須セクションの存在」「dangling な skill 参照の警告」を検査し、routing eval を `--min-rank1 95` の CI 門番にし、3 OS で実 plugin 導入まで回す ([skill-lint.js](https://github.com/addyosmani/agent-skills/blob/main/scripts/lib/skill-lint.js), [workflow](https://github.com/addyosmani/agent-skills/blob/main/.github/workflows/test-plugin-install.yml))。ECC も `npm test` が validate-skills / validate-hooks / catalog drift などを連鎖させる。それでも ECC の skill 間参照は 26 skill・51 件のうち 14 件が dangling で、検証の存在と結果の健全さは別であることも分かる（前回調査の正規表現による概算）。

CAE の CI（`.github/workflows/test.yml`）が実行するのは `hooks.test.sh`、`harness.test.sh`、cad と orchd の `go test` だけである。最も新しい guardrail である scope-check（commit f14f23b、9/29）のテストは CI に無く、project の CLAUDE.md の「テスト」記述にも挙がっていない。文書・CI・実体が少しずつずれている状態である。`.ts` hook を Claude Code が実際に呼ぶ経路、`if: Bash(gh *)` が効くこと、壊れた settings.json、並行実行、バックアップ管理も未カバーである。加えて worker-image テストは今回 exit 1 だった（docker 不在の環境依存の可能性が高いが、ログは読んでおらず未検証）。

依存の明示度は 3 者とも frontmatter 宣言が無く、名前参照の暗黙依存が中心である。CAE には機械可読な契約が 4 つある（harness.json、`waves.py` が読む changegraph YAML、handoff の `Issue: #N` 行、`codingenv` の SKILLS/TOOLS の走査と HOOKS の手書き一覧）。skill が 9 本で参照が taskman → design → changegraph → dispatch の直線的な流れなので、人間が全体を把握できる。しかし dangling 参照を機械では検出できない。実例として `.claude/changegraph/dev-env-upgrade.yaml` が参照する `docs/decisions/0008-*.md` は repo に存在しない。規模が小さいうちは問題にならないが、skill が増えたときに最初に効いてくる弱点である（推論）。

## 若さと単一障害点：最大のリスクは hook 個体ではなく配布の仕組みにある

「10 日・293 commit」は、安定の証拠ではなく変動の証拠である。9/28 が 87、9/29 が 106 commit で、`fix|revert` を含む subject が 41 件あり、大半は cad / orchd / deploy 側の修正である。SessionStart hook は当初 `gh` で doc:spec Issue を列挙していたが後に縮小され、TaskCompleted hook は 9/29 に丸ごと廃止された（commit 3d65706）。ECC の 3,040 commit と直近 1 日約 25 commit も速いが、こちらは第三者の issue 報告と公開リリース（2〜4 週）という外部の検証を通っている。ただし ECC 自体も、repo の 2.2.2 が npm に未公開で README の pin が 404 になる報告（[#3222](https://github.com/affaan-m/ECC/issues/3222)）や、tarball の欠落ファイル（[#3270](https://github.com/affaan-m/ECC/issues/3270)）を抱えており、成熟しているとは言えない。CAE の利用者は本人のみで、hook が発火・誤ブロックしたログも無い（hook がログを書かない設計のため取得できない）。

単一障害点は 2 つある。第一は symlink である。`~/.claude/skills/*` と `~/.claude/hooks/*` は repo への symlink なので、repo のブランチ切替や worktree での編集、削除が、全 project の全セッションに即座に効く。feature ブランチ上の未レビューの hook が有効になりうる（この調査の repo も `feature/ecc-cc` 上にあった）。ECC と agent-skills はコピーまたは plugin のバージョン固定で配るためリリース境界があり、CAE にはそれが無い。第二は `~/.claude/CLAUDE.md` のマーカー区間の自動書き換えで、LLM への指示文（`global/CLAUDE.harness.md`）が事実上の実行系になっている。この文書が参照する `testdb` は `tools/` に実体が見つからず（未確認）、LLM を誤導しうる（推定）。

CAE 固有の具体的な弱点は次のとおりで、いずれも今回の調査で確認したものである。

| 弱点 | 事実 | 影響 |
|---|---|---|
| CI が scope-check テストを回さない | `test.yml` は hooks・harness・go test のみ | 最新の guardrail が回帰しても CI は通る |
| `.bak` が無制限に積み上がる | `backup()` が無条件 `cp`。実機で CLAUDE.md 21 個、settings.json 22 個、計約 1.7 MB（10 日）。内容が同一のものも多い | 70 KB 級の settings.json の平文コピーが増え続ける（機密性は未確認） |
| 部分適用で終わり rollback が無い | 壊れた settings.json で install すると symlink は作成済み、settings.json は未変更、空の `settings.json.tmp` が残り、CLAUDE.md は未更新 | 途中状態が残り、settings.json を直すまで同じ場所で失敗する |
| 固定 `.tmp` 名・dry-run 無し | `settings.json.tmp` / `CLAUDE.md.tmp` は固定名で mktemp・lock 無し | 並行実行や Stop 時の書き戻し hook と競合しうる（推定・未実験） |
| マーカー文字列に旧名 | `managed by codingagentenv bin/harness` が残る（コメントは「rename するな」） | 実害は無いが名称ドリフト |
| 参照先が存在しない | `docs/decisions/0008-*.md` | 古い参照が残る例 |
| `.env` が未追跡かつ未 ignore | `git status` に `?? .env`（166 bytes）、`.gitignore` に `.env` 無し。中身は読んでいない | 誤 commit のリスク |
| skill の内容検証が無い | skill 対象のテスト・lint は 0 件 | dangling 参照・description の肥大・harness.json 読み手の不整合を検出できない（推論） |
| 環境前提が無検査 | `.ts` hook は node の型ストリップ依存（古い Node では機能停止と推定）、`--body-file -` は誤ブロック側 | 環境不備は fail-open か誤ブロック |

なお、調査ノード間で数値の不一致がある。テスト行数は 705 と 765、installer の行数は 224 と約 250 と記録されており、どちらが正確かは再確認していない（判定への影響は小さい）。

## CAE が取り込むべきもの、取り込まないもの

優先度は、CAE の実害が大きい順、かつ 1 PR で足せる順に並べる。

| 優先 | 取り込む | 出どころ | コードレベルの理由 |
|---|---|---|---|
| 1 | `scope-check.test.sh` を CI に追加、project の CLAUDE.md のテスト記述も更新 | CAE 内 | `test.yml` に 1 行足すだけで、唯一のブロック hook が CI 対象になる |
| 2 | skill lint（frontmatter の name==dir、description の存在、`/<name>` 参照先の実在、行数上限） | agent-skills の `skill-lint.js` | CAE は skill を走査で自動登録しており（共有一覧を触らない）、lint も走査型なら並列開発と衝突しない。`docs/decisions/0008-*.md` のような dangling を機械で拾える |
| 3 | installer の堅牢化：JSON 妥当性の事前検査（symlink 作成より前に `jq empty`）、bak の同一内容スキップと世代上限、`mktemp` | CAE 内（ECC の lock・atomic write の考え方） | 部分適用（symlink 済み・settings 未変更）と bak 43 ファイル / 1.7 MB は、いずれも `bin/codingenv` の実行順と `backup()` の無条件 `cp` に由来する |
| 4 | `status` に「repo が main 以外のブランチ／未コミット変更あり」の警告 | CAE 内 | symlink の target がチェックアウトに追従するため、未レビューの hook が全 project に効く経路を検出できる。リリース境界の代用になる |
| 5 | `.env` を `.gitignore` へ、bak の権限（`chmod 600`）確認 | CAE 内 | 未追跡 `.env` と settings.json の平文コピーが増える経路がある |
| 6 | description のトリガー判定 eval（決定的な TF-IDF 方式） | agent-skills の `run-evals.js` | 9 skill で今すぐ必要とは言えない。日本語 description に対する有効性は未検証なので、skill が増えてからでよい |
| 7 | 編集阻止系（config-protection、`--no-verify` 阻止）を限定的に | ECC | 現状の強制力の空白を埋めるが、入れるなら例外時に許可へ倒す fail-open 形に直す。誤検知の報告（#3048 等）があるため opt-in で試す |

取り込まないほうがよいものも明確である。fail-closed の dispatcher（ECC 側のバグが全 Bash 停止になる構造）、Stop 時の formatter / typecheck による無断書き換え、`.*` matcher の observe（全ツール呼び出しで node を起動する）、`claude -p` による transcript 要約（追加のモデル利用と送信）、`includeCoAuthoredBy` の暗黙書き込み、manifests・registry・hook の sidecar ハッシュ（skill 9 本・hook 2 本では管理コストが利益を上回る。status が既に drift を見ている）、command 3 コピーの parity（CAE は command 層を持たない）、Rationalizations 表・Red Flags の全 skill への一律導入（invocation ごとの本文トークンが増え、「少し足りないくらいから始める」という CAE の方針と逆行する）である。hook-consent の能力列挙も、利用者が本人のみで install を本人が実行する現状では不要である。

## Conclusion

「CAE は ECC より安定」は、正確には「CAE は壊れたときの被害範囲が小さい」であり、その理由は品質の高さより、持っている機能と強制機構の少なさにある。この 2 つを混同すると、CAE に guardrail を足すたびに ECC と同じ failure mode（fail-closed、副作用、常時コンテキスト増）を招く。逆に、ECC の欠陥と同型のものが CAE の installer にも存在する（部分適用、バックアップ管理、CI の穴）ので、「構造的に存在しない」のは fail-closed と Stop 時の書き換えのみと限定して読むのが正確である。

次にやる価値が高いのは、機能の追加ではなく検証の追加である。agent-skills が示すように、小さい repo でも lint・eval・CI 門番があれば skill の中身に対する信頼を機械的に積み上げられる。CAE の弱点は「強制が弱い」ことより「skill と CI の穴を機械が見ていない」ことで、そこを塞ぐ優先 1〜4 は、いずれも既存の走査型・小ファイル方針を崩さずに足せる。未検証の点は、CI の他 workflow の中身、worker-image テスト失敗の原因、`testdb` の所在、`.env` の中身、実際の発火精度と context 消費（`/context` 未実行）である。
