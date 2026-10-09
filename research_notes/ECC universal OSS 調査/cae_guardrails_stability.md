# codingagentenv (CAE) の guardrails / installer / 安定性（ECC・agent-skills との比較用）

調査日 2026-09-30。対象 repo は /Users/yasumasa_takemura/projects/codingagentenv、HEAD 17fcab9（main、git status は untracked のみ）。出典はすべてローカルの実ファイルと実行結果（Source 欄は絶対パス、または実行したコマンド）。推定・未検証は明記した。

## Q1. hook（guardrails）は何個あり、何をし、失敗時にどう振る舞うか

### Takeaway
CAE が登録する hook は 2 本だけ（SessionStart の文脈注入、PreToolUse の `gh issue/pr create` 本文チェック）。副作用はほぼ無く、ネットワーク呼び出しも無いので、ECC の 24 個の `node -e` bootstrap や fail-closed dispatcher と比べて事故面が小さい。一方で「ガードレール」としての強制力は極めて限定的で、しかも fail-open である。

### Cited Findings
- 登録される hook は `HOOKS='SessionStart:harness-session-start.sh PreToolUse:harness-scope-check.ts'` の 2 本だけ。timeout は 10 秒固定で、`$filter.group + {hooks:[{type:"command",command:$command,timeout:10} + $filter.handler]}` で書く — /Users/yasumasa_takemura/projects/codingagentenv/bin/codingenv（L47, L133-L140 付近）
- **Hook 1: SessionStart（`hooks/harness-session-start.sh`、bash）**。matcher なし。
  - 動作：stdin の `source` が startup / clear / compact のときだけ動く。resume / fork は即 `exit 0`（L15-L19）。`$CLAUDE_PROJECT_DIR`（無ければ `.cwd`）配下の `.claude/handoff/*.md` を `ls -t` で列挙し、未記入テンプレート（`YYYY-MM-DD` を含む）は除外する。有効な handoff が 1 件なら先頭 60 行を stdout に出す（= Claude の context に注入される）、2 件以上なら 1 行ずつの一覧、0 件なら `PROGRESS.md` の先頭 60 行にフォールバックする。最後に「/pickup の手順に従え」という指示行を出す（L60-L64）
  - 失敗挙動：`set -u` のみで `set -e` なし。常に `exit 0` で終わる。jq が無い・JSON が壊れている場合は `jq ... 2>/dev/null` が空を返し `startup` 扱いで進む。fail-open（コンテキストが注入されないだけ）
  - 副作用：ファイル書き込みなし、ネットワークなし。読むだけ。git / gh も呼ばない（test が gh スタブで「呼ばれないこと」を検証している）
  - 遅延リスク：`ls`/`grep`/`awk` を handoff 数だけ回す程度で ms オーダ。ただし handoff 内容（人間が書いた markdown）を無検証で context に流し込む点はプロンプト注入面（自分の repo 内ファイルなので信頼境界は自分自身）
  - 出典：/Users/yasumasa_takemura/projects/codingagentenv/hooks/harness-session-start.sh
- **Hook 2: PreToolUse（`hooks/harness-scope-check.ts`、node）**。matcher `Bash`、さらに handler に `"if": "Bash(gh *)"` を付けるので、`gh` で始まるコマンドの時だけ起動する（installer の `hook_filter`）。
  - 動作：コマンド文字列が `/\bgh\s+(issue|pr)\s+create\b/` に一致するときだけ、本文に `- 追加|変更|削除: <語> \`...\`` 形式の行があるかを正規表現で検査する。本文は `--body-file/-F` のファイル、無ければコマンド全体（`--body`/heredoc を含む）。無ければ stderr に 1 行出して `exit 2`（= ブロック）
  - 失敗挙動：stdin が JSON でない → `exit 0`。`command` が文字列でない、対象でない → `exit 0`。`--body-file -`（stdin）は空文字扱いで **ブロックされる**（誤ブロック側）。`--body-file` の読み込み失敗も空扱いでブロック。node が無い / 古い / ファイルが消えている場合は node 自身のエラーで exit 1 か 127 になる。exit 2 以外は非ブロックというのが Claude Code の hook 契約だが、これはこの調査では公式 docs を再確認していない（**未検証**。CAE 自身の冒頭コメントは「Exit 2 blocks」とだけ書く）。したがって環境不備は fail-open
  - 副作用：ファイル書き込みなし、ネットワークなし。`--body-file` の読み取りのみ
  - 遅延：`if` で絞るため gh 以外の Bash では node を起動しない。gh コマンドでは node 起動 1 回（数十 ms 台と推定、未計測）
  - 実行前提：`.ts` を `node` で直接実行する（型注釈 `let input: {...}` を含む）。Node の TypeScript 型ストリップに依存し、この環境は node v26.0.0 で動く。古い Node（型ストリップ非対応）では構文エラーで exit 1 = 機能停止（これは推定。実際に古い Node では試していない）。`if` フィールドが効かない古い Claude Code でも、ブロック条件は hook 側で判定するので誤ブロックはしない
  - 出典：/Users/yasumasa_takemura/projects/codingagentenv/hooks/harness-scope-check.ts（L1-L6 コメント、L9-L11 正規表現、L34 の `exit(2)`）
- 保護対象の狭さ：ブロックするのは「PR/Issue 本文に変更一覧行があるか」だけ。ECC の GateGuard（最初の Edit/Write を止める）、config-protection、`--no-verify` 阻止のような、コード変更・git 操作そのものを止める hook は CAE に無い — 同上 hooks/ の 2 ファイルのみ（`git ls-files hooks/`）
- 実際に稼働中の環境（~/.claude/settings.json）では、CAE の 2 hook は symlink（~/.claude/hooks/harness-*.{sh,ts} → repo）として存在し、`hooks.SessionStart` の 3 番目、`hooks.PreToolUse` の最後に登録されている（`jq '.hooks.SessionStart'` で確認）。同じ settings.json には CAE 由来でない hook が多数ある（Orca / Superset の通知、gsutil cat の検査、`auto-branch.sh`、`save-prompt.sh`、Stop 時に `.claude/settings.local.json` の allow を global settings.json へ書き戻す python3 -c）— /Users/yasumasa_takemura/.claude/settings.json（74,065 bytes）
- 過去に退役した hook：TaskCompleted の `harness-task-completed.sh` と `env.CLAUDE_CODE_ENABLE_TODO_TOOLS` は #103 で廃止され、installer の `drop_retired` が掃除する — bin/codingenv L44-L45, L86-L90。commit 3d65706 "remove built-in Task list and TaskCompleted hook (ADR-0016)"
- hook 内の Claude 呼び出し（`claude -p`）、Stop 時のソース書き換え、format/typecheck 実行、コスト記録、観察ログは **CAE の hook には無い**（hooks/ の 2 ファイルを全読して確認）。書き込むファイルはゼロ

### Inferences
- ECC 比：CAE の hook は「context を足す」「不備のある PR 本文を止める」の 2 機能で、実行コード量・副作用・遅延・fail-closed による全 Bash 停止リスクのいずれも小さい。「安定している」のは主に機能が少ないため（機能量の差）であり、同じ機能を安定して実装している証拠ではない
- CAE の唯一のブロック hook は fail-open 寄りの設計（想定外入力は exit 0）。一方 `--body-file -` や `-F` 相対パスの解決失敗は誤ブロックになりうる。ブロックされた場合のメッセージは 1 行で、回避（本文に 1 行足す）も容易
- 「ガードレール」としては、規範の大半は `global/CLAUDE.harness.md` の文章（LLM への依頼）に依存し、機械的な強制は上記 2 点のみ。ECC が hook で強制している領域（編集前確認、設定改変の阻止、`--no-verify` 阻止）は空白。安全性＝強制力が高い、ではない
- 別の危険は、hook 本体が repo への symlink であること。repo で編集・ブランチ切替・削除をすると、全 project の全セッションの hook 挙動が即座に変わる（CLAUDE.md にも「編集は全 project に即座に効く」と明記）。ECC の「versioned な npm/plugin 配布」と比べると、リリース境界が無い

### Gaps
- Claude Code の hook 終了コード契約（exit 2 のみブロック）と `if` フィールドの正確な仕様は、今回公式 docs で再確認していない
- `harness-scope-check.ts` を古い Node で実行した場合の実挙動、gh コマンド時の node 起動レイテンシは未計測
- 実運用でブロックがどれだけ発火したか、誤ブロック率のログは無い（hook 側にログ出力が無い）

## Q2. installer（`bin/codingenv install`）は何を書き換え、冪等・rollback・衝突処理はどうなっているか

### Takeaway
`~/.claude` への書き込みは「symlink 作成」「settings.json の hook 追記」「CLAUDE.md のマーカー区間置換」に限定され、ECC のように includeCoAuthoredBy 等の無関係設定を書かない。冪等で uninstall / status もあるが、dry-run が無く、途中失敗の rollback が無く、毎回タイムスタンプ付きバックアップを積み上げる。

### Cited Findings
- 出典は特記なき限り /Users/yasumasa_takemura/projects/codingagentenv/bin/codingenv（約 250 行の bash、`set -eu`、依存は jq のみ）。サブコマンドは `install | uninstall | status` の 3 つだけ。`CLAUDE_DIR` と `BIN_DIR` を環境変数で上書きでき、テストはこれで一時ディレクトリに対して実行する
- **symlink 方式**（コピーではない）：`ln -sfn` で、`skills/*`（現在 9 個）→ `~/.claude/skills/<name>`、`hooks/*` 2 本 → `~/.claude/hooks/`、`global/harness.json` → `~/.claude/harness.json`、`bin/codingenv` と `tools/*`（ai-review, cad, codex-localreview, codex-probe, orchd）→ `~/.local/bin/`。実機で symlink を確認済み（`ls -la ~/.claude/skills ~/.claude/hooks`、harness.json も repo への symlink）
- **衝突処理**：`$BIN_DIR/codingenv` が非 symlink ファイルなら install 全体を中止（exit 1）。個別 tool と `harness.json` が非 symlink なら警告して skip。skill は `ln -sfn` で、同名の既存ディレクトリがあると symlink をディレクトリ内に作る挙動になりうる（`ln -sfn` は既存の実ディレクトリに対しては中に作る。これは POSIX/BSD ln の一般挙動で、CAE 上では未再現。既存 skill との同名衝突チェックは無い、が確認できた事実）。実機では ~/.claude/skills に CAE 外の skill（commit, pr, report 等）が共存しており、同名衝突は現状無い
- **settings.json**：無ければ `{}` を作り、`backup` してから jq で追記。(1) 退役 env と TaskCompleted を除去、(2) 各 hook について「同じ command を持つ group が無ければ append」。既存の hook や他キーは保持（test で検証済み）。`includeCoAuthoredBy` / `attribution` / permissions / model など無関係な設定は書かない。実機 settings.json の `includeCoAuthoredBy` は false だが、installer のコードに該当書き込みは無い（grep で確認）ので CAE 由来ではない（値の出どころは不明）
- **CLAUDE.md**：`backup` 後、`strip_section`（マーカー区間、またはマーカー導入前の手書き `## ハーネス（全プロジェクト共通）` 見出しから次の `## ` まで）を除いた本文の末尾に、`global/CLAUDE.harness.md` を begin/end マーカーで挟んで追記し、`.tmp` から `mv`。ユーザー自身の他セクションは維持（test 済み）。実機の diff では、deployed のマーカー区間と repo 正本は一致
- **マーカー文字列の残骸**：マーカーに `managed by codingagentenv bin/harness` と旧名が残る（L32）。コメントに「rename するな」とある通り互換のため。README/help は `bin/codingenv`。実害は無いが名称ドリフト（コマンドが harness→codingenv に改名された名残）
- **冪等**：install 2 回で status が 0 を返すことを test（"install repairs it" 等）。追加実験（一時 dir、実 ~/.claude には触れず）：3 回連続 install で hook 登録は `["PreToolUse","SessionStart"]` のまま重複せず、settings.json / CLAUDE.md 用の `.bak-*` が **各実行ごとに 1 つずつ増えた**（3 回で計 6 ファイル）
- **バックアップの累積**：実機 ~/.claude に `CLAUDE.md.bak-*-harness` が 21 個、`settings.json.bak-*-harness` が 22 個、合計約 1.7 MB（`ls`/`du`）。最初は 2026-09-20 11:52、最新は 2026-09-30 13:29。CLAUDE.md の bak は内容が同一のものも多い（例：`20260927-161328` と `161330`、5340 bytes）。バックアップは内容が変わらなくても取る（`backup()` は無条件 `cp`、L60-L63）。削除・世代管理・上限は無い。settings.json の bak は 70KB 級で permissions や MCP 設定を含むため、平文コピーが増え続ける点は衛生上の懸念（含まれる中身の機密性は未確認）
- **uninstall**：symlink が自分のもの（`readlink` が repo を指す）の場合のみ削除、settings.json から自分の hook のみ jq で除去（group は hooks が空になったときだけ削除）、CLAUDE.md からセクションを除去。無関係な hook は残る（test 済み）。ただし backup を復元する機能ではなく、「自分が足した分を引く」方式。ECC の install-state（ID 単位のマージと記録）と似た目的だが、状態ファイルは持たず、command 文字列の一致で自分のものを判別する（同じ command を他人が登録していても、区別できない）
- **status**：読み取り専用の drift 検査（symlink 先、hook 登録、退役物の残存、CLAUDE.md セクションの一致）で、drift があれば exit 1。ECC には見当たらない軽量な整合確認として有用
- **dry-run**：無い。`grep -n dry bin/codingenv` は 0 件。`status` が事後確認の代替
- **失敗時の途中状態（実験）**：壊れた settings.json（`{bad json`）を置いた一時 CLAUDE_DIR で install すると、jq が exit 5 で失敗し、その時点で symlink（skills 9 本、hooks 2 本、harness.json）は作られ済み、settings.json は未変更のまま残り、空の `settings.json.tmp` が残骸として残り、CLAUDE.md は未更新（CLAUDE.md 自体が作られない）。つまり **部分適用状態で終わり、rollback は無い**。再実行で回復するかは、settings.json が直るまで同じ場所で失敗するので不可
- **並行実行**：`settings.json.tmp` / `CLAUDE.md.tmp` は固定名で、`mktemp` でも lock でもない。2 プロセスの同時 install は競合しうる（推定。実験していない）。さらに実機の settings.json には、Stop 時に `~/.claude/settings.json` を `json.dump(..., open(global_path,'w'))` で直接書き戻す既存 hook がある（CAE 由来ではない）。jq → `mv` の間隔は短いが、install 中のセッション終了と重なれば最後に書いた側が勝つ（推定）
- **再起動要件**：install は最後に「Restart Claude Code to pick up the hooks」を表示する。稼働中セッションの hook は入れ替わらない
- **CLAUDE.md への副作用**：install は `~/.claude/CLAUDE.md`（全 project に効くユーザーグローバル指示）を書き換える。README と CLAUDE.md には「install はユーザーが実行する」とあり、Claude の自動実行は想定していない（/Users/yasumasa_takemura/projects/codingagentenv/CLAUDE.md の「コマンド」節）

### Inferences
- ECC との比較で確かに優位な点：(1) 無関係な設定（includeCoAuthoredBy 等）を書かない、(2) symlink なので repo が唯一の正本、更新は `git pull` だけで反映、(3) uninstall と status がある、(4) 書き換え範囲がマーカーと command 一致で限定されている。設計上のスコープは狭く、透明性が高い
- 弱点：(1) dry-run と事前 consent 表示が無い（ECC の hook-consent に相当するものが無い）、(2) 失敗時に部分適用で止まり戻せない（backup は取るが復元コマンドが無い）、(3) バックアップが無制限に積み上がる（実測 43 ファイル / 1.7MB / 10 日）、(4) 固定 `.tmp` 名で並行実行に弱い、(5) 壊れた settings.json の事前検査が無い。install の実行頻度が高い（10 日で 22 回 = ほぼ 1 日 2 回以上）ことがバックアップ累積と、正本を頻繁に変えている実態の両方を示す
- symlink 方式の副作用：repo のブランチを切り替える／worktree で編集すると、そのブランチの skills と hooks が即座に全 project に効く。feature ブランチ上の未レビューの hook が有効になるリスクがある。実機は `feature/ecc-cc` 以外の main 上 HEAD で調査したが、これは日常的に起こりうる
- ~~ECC の 2.2.2 vs npm 2.2.1 のような版数ドリフト~~ に相当する問題は、CAE には配布物が無い（npm 等に出さない、repo 直下の bin を symlink）ため起きない。代わりに「ローカル checkout と `~/.claude` の同期ずれ」が drift で、`status` がそれを検出する

### Gaps
- 並行 install、jq 途中失敗時の `.tmp` 残骸による副作用（次回 install で上書きされるはず）は実験していない
- backup の中身に含まれる機密情報の有無は確認していない
- `ln -sfn` の同名ディレクトリ衝突は未再現

## Q3. テストと CI は何をカバーし、今通るか

### Takeaway
installer と SessionStart hook には、実機に触れない一時ディレクトリ方式の網羅的な shell テストがあり、今すべて通る。CI が回すのはその一部だけで、scope-check hook のテストは CI に載っていない。

### Cited Findings
- test/ に 8 本の shell テスト（合計 765 行）。CI 対象は `test/hooks.test.sh` と `test/harness.test.sh`、および `cad/` と `orchd/` の `go test ./...` — /Users/yasumasa_takemura/projects/codingagentenv/.github/workflows/test.yml（Go 1.27、`actions/checkout` は SHA ピン）。CI に載っていないのは `scope-check.test.sh`, `ai-review.test.sh`, `fetch-auth.test.sh`, `opencode-worker.test.sh`, `worker-image.test.sh`, `worker-startup.test.sh`
- 今回の実行結果（この環境、非破壊。各テストは mktemp の一時 dir を使う）：
  - hooks.test.sh：exit 0、ok 31、FAIL 0
  - harness.test.sh：exit 0、ok 55、FAIL 0
  - scope-check.test.sh：exit 0、ok 13、FAIL 0
  - ai-review.test.sh：exit 0、ok 36、FAIL 0
  - fetch-auth.test.sh：exit 0（出力形式が `PASS:` 行で、ok カウントは 0 だが失敗なし）
  - opencode-worker.test.sh / worker-startup.test.sh：exit 0（出力形式違い、失敗なし）
  - worker-image.test.sh：**exit 1**、`FAIL - docker build (see .worker-image-build.log)`。docker が使えない環境依存の可能性が高いが、原因ログは読んでいない（**未検証**）。CI の `image.yml` は別で、直近の run は success
  - `go test ./...`：cad、orchd とも ok
- harness.test.sh がカバーする内容：手書き旧セクションを含む既存 settings/CLAUDE.md に対する install、2 回 install の冪等性、既存 hook 保持、退役 hook/env の除去、status の drift 検出（退役物、harness.json の欠落、手編集）、uninstall が他人の設定を残すこと、空ディレクトリへの新規 install、settings 不在での uninstall、手書き harness.json の保護、非 symlink ファイルの上書き拒否、不明サブコマンドの失敗 — /Users/yasumasa_takemura/projects/codingagentenv/test/harness.test.sh
- **カバーされていないもの**：壊れた settings.json、jq 失敗時の部分適用、並行実行、バックアップの世代管理、symlink 先ディレクトリの同名衝突、`.ts` hook を実際に Claude Code が呼ぶ経路（テストは `node hook` へ直接 stdin を流す）、hook の `if: Bash(gh *)` が実際に効くこと
- CI の直近 run：`test` は main と PR ブランチで success（2026-09-30 04:26-04:27）、`image` も success、`Claude Code Review` success、`Claude Code`（@claude 起動）は多数 skipped（設計上、@claude を含むコメント時のみ動く）— `gh run list --limit 12`
- lint / 型検査：無し（CLAUDE.md に「lint / 起動：なし」）。`.ts` hook の型検査は CI に無い。リポジトリに package.json は無く、`node_modules/@typesafe-ai/sdk`（未追跡、package-lock のみ）が残っているが hook は依存しない
- 覆っていない別種の検証：ECC が持つ validate-hooks / hooks.metadata のハッシュ検査、catalog drift 検査に相当するものは無い（CAE は catalog を持たないので不要とも言える）

### Inferences
- installer と SessionStart には、ECC 相当以上に「実際の失敗ケースを test に落とした」痕跡がある（退役 hook、手書き旧セクション、非 symlink 衝突）。過去の障害を受けて test を足す運用が回っている
- ただし、最も新しく追加された guardrail（scope-check、commit f14f23b、2026-09-29）は CI に載っておらず、ローカル手動実行に依存する。CLAUDE.md の記述も「テスト：hooks.test.sh、harness.test.sh」の 2 本で scope-check を挙げない。ドキュメント・CI・実体の 3 者が少しずつずれている
- 失敗系（壊れた入力）のテストが薄いのは、単一ユーザーが自分の環境に入れる前提の設計と整合する

### Gaps
- worker-image テスト失敗の根本原因（docker 不在か実バグか）は未確認
- CI の `image.yml`、`claude.yml`、`claude-code-review.yml` の中身は今回精査していない（`image.yml` は success を確認しただけ）
- カバレッジ測定は無い

## Q4. 安定性の証拠：git 履歴、既知の問題、失敗しうる箇所

### Takeaway
repo は 10 日で 293 commit と、変更が非常に速い若い repo であり、「安定している」と言える期間の実績は無い。guardrail/installer 自体の fix は少ないが、周辺（cad/orchd/デプロイ）は fix が多く、設計が頻繁に置き換わっている。

### Cited Findings
- 履歴：総 commit 293。日付別は 2026-09-20 が 27、09-27 が 59、09-28 が 87、09-29 が 106、09-30 が 2（`git log --format=%ad --date=short | sort | uniq -c`）。最古は 2026-09-20 で、稼働期間は約 10 日
- `fix|revert` を含む subject は 41 件。うち Revert は 1 件（b54677f "Revert orchd: add mode awake"、2026-09-29）。fix の大半は cad / orchd / deploy / worker（例 acfd130 wall-clock timers、cea9161 retry a task that never started、abb392c ai-review の stale head 拒否）で、guardrail/installer 系は次の通り少数：92ecd5b "fix(hooks): skip the doc:spec gh call on compact and outside github.com repos"、386c241、9d16328 "hooks: drop backslashes from scope-check stderr message"、b… "hook: stop listing spec Issues"（3b45a86）、19b8fd0 "Drop the hook change: skills read harness.json tickets themselves"
- **hook 設計の書き直し**：SessionStart hook は当初 doc:spec Issue を `gh` で列挙していたが、後に「hook は spec を一覧しない」へ縮小した（3b45a86、70ed506、ae70cce 周辺。全体の変遷は `git log -- hooks/`）。TaskCompleted hook と Task list は 09-29 に丸ごと廃止（3d65706）。つまり hook の機能範囲は短期間に何度も変更されている
- **レビュー起因の修正が多い**：`address Codex review`（9083e78、dc9dde7、d7b8180 等）、`address review` が複数。実装者と別の AI レビューを通す運用（README/CLAUDE.harness.md）が回っている証拠だが、同時にレビューで初めて欠陥が見つかる頻度が高いことも示す
- **issue 状況（`gh issue list --state all`）**：約 45 件、その大半が `spec:` で始まる OPEN（設計決定 Issue、#92-#108 など）。バグ報告ラベルの Issue は見当たらず、closed は #59,#60（sleep-check の疑似タスク）、#64,#67,#73-#75 のみ。#83「place の runner レコードのパラメーターを適切な値にする仕組み」、#37「セキュリティ: main の保護と認証情報の置き場所」、#38「ハーネスを cloud セッションにも届ける」などが OPEN で、未解決の課題として残る。TODO/FIXME コメントは README/CLAUDE.md/docs に無い
- `~/.claude/tasks/codingagentenv/`：19 個の JSON（status 集計：completed 14、in_progress 4、pending 1）。最終更新 2026-09-29 22:29。内蔵 Task list は #103 / ADR-0016 で廃止済みなので、これらは廃止機能の残骸（`~/.claude/tasks/` の下に他の project 分も残る）。in_progress のまま止まっている 4 件は現在の運用には無関係だが、片付けの仕組みが無い。中身の精査はしていない
- **README とコードのずれ・名前ドリフト**：`bin/harness` の旧名がマーカー文字列に残存（上記）。`global/CLAUDE.harness.md` が `orchd`、`cad`、`ai-review`、`codex-probe`、`testdb`、`cad` の「Issue #10」などを前提にしており、`testdb` は `tools/` に存在しない（`ls tools` は ai-review, cad, codex-localreview, codex-probe, orchd のみ。**`testdb` の実体は repo 内で未確認**）。ハーネス文書が参照するコマンドの一部が repo 外か未実装の可能性がある（ただし deployed CLAUDE.md の指示として LLM を誤導しうる点は推定）
- **ハードコード・環境依存**：`.claude/settings.json`（repo 内）は enabledPlugins と skillOverrides のみ。bin/codingenv は `$HOME/.claude/hooks/...` をコマンド文字列に埋め込み（環境変数 `CLAUDE_DIR` で上書きしても登録される command は `$HOME/.claude` 固定）、`CLAUDE_DIR` 上書きは実質テスト専用。repo 内に `.env`（166 bytes）が未追跡であり（`git status`: `?? .env`）、内容は読んでいない。`.gitignore` に `.env` は無く、誤 commit のリスクがある（**要確認事項**）。CI は Go 1.27 を要求（go.mod も 1.27）
- **依存**：install は `jq` 必須、hook は `bash`、`jq`、`node`（型ストリップ対応版）に依存。Claude Code 側の仕様（`hooks[].if`、SessionStart の stdout が context になる、`source` 値の種類）に依存し、これらは Claude Code の版によって変わりうる。CAE は版を固定・検査しない（推定）
- **ECC の失敗モードとの対比**：ECC で報告されている (a) fail-closed dispatcher による全 Bash 停止、(b) Stop 時のソース書き換え、(c) npm 版と repo 版の不一致、(d) install で無関係設定（includeCoAuthoredBy）を書く、はいずれも CAE には構造上存在しない。CAE の hook は副作用がなく、install も無関係キーを書かない

### Inferences
- 「ECC より安定」という主張は、**事故面（blast radius）が小さい**という意味では支持される（hook 2 本、書き込み無し、fail-open、無関係設定に触れない、status/uninstall あり、installer と SessionStart に実テストあり）。ただし **成熟・運用実績という意味では支持されない**：稼働 10 日、1 日 100 commit 級の変更、hook 範囲が数度作り直し、単一ユーザー・単一マシン前提で第三者による検証が無い。ECC は大規模だが公開・第三者検証・issue 報告がある一方、CAE は自分以外の利用実績が無い（利用者は本人のみと CLAUDE.md に明記）
- 強制力の低さと安定性は表裏の関係にある。CAE が壊れにくいのは、壊れうる強制機構を持たないため
- 実際の最大リスクは hook 個体ではなく (1) symlink による即時全域反映（未レビュー変更が全 project に効く）、(2) `~/.claude/CLAUDE.md` の自動書き換えで、LLM への指示文（`global/CLAUDE.harness.md`）が事実上の実行系になっている点、(3) 文書が参照する周辺ツール（cad / orchd / testdb / 起動中の VM）が未成熟な点。周辺ツールの fix 密度（41 件中大半）が高い
- installer の堅牢化候補（優先度順、推定）：バックアップの重複排除と世代上限、jq 失敗時の事前検証（settings.json が有効な JSON か）、固定 `.tmp` を `mktemp` に、dry-run（または `status` 相当の diff 表示）、scope-check テストの CI 追加

### Gaps
- `testdb` の所在、`.env` の中身と扱い、worker-image 失敗の原因は未確認
- hook の実発火ログ・誤ブロック実績のデータは無い（hook がログを書かない設計のため取得不能）
- ECC 側の現行 HEAD との比較（例：ECC の hook の実際の失敗率）は今回のスコープ外
- 他人の環境（別 OS、別 Node、別ユーザー）での install 成否は未検証（bash 3.2 互換性も未確認。macOS 標準 bash での実行は今回の実行環境で通った）
