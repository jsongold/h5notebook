# codingagentenv（自分の harness repo）の中身と部品の書き方

方法：/Users/yasumasa_takemura/projects/codingagentenv を read-only で調査（ls / wc / cat / grep、`codingenv status` のみ実行）。出典はすべてローカルファイルのパス。ECC / agent-skills との比較用。行数は `wc -l`、サイズは `wc -c`（bytes）と `wc -m`（chars）。repo の最初の commit は 2026-09-20、最新は 2026-09-30（`git log` の日付）、commit 293。

## 1. Repo の構成と規模（何を含むか）

### Takeaway
harness の中核は skill 9 本（合計 268 行 / 24,176 bytes）・hook 2 本・global の CLAUDE 節 1 つ・installer 1 本（`bin/codingenv` 224 行）と小さい。agents / commands / rules ディレクトリは無い。repo の体積の大半は、24/7 運用のための周辺ツール（Go 製 `cad` と `orchd`、GCP デプロイ、worker image）である。

### Cited Findings
- 追跡ファイル 109（`git ls-files`。`.claude/worktrees/` の作業コピーは未追跡なので除外）。ディレクトリ別: cad 31、orchd 16、deploy 11、skills 10、test 8、tools 5、.github 5、docs 4、agent 4、hooks 2、global 2 ほか。
- skill 9 本（`skills/*/SKILL.md`）: taskman 29 行、design 41、changegraph 49（+ `waves.py` 77 行）、dispatch 28、pickup 22、handoff 25、setupca 24、igosleep 23、iwokeup 27。合計 268 行、24,176 bytes。最大は iwokeup の 4,328 bytes。
  - 出典: `/Users/yasumasa_takemura/projects/codingagentenv/skills/*/SKILL.md`
- agent 定義・command 定義・rules: repo 内に `agents/`、`commands/`、`rules/` は存在しない（`ls` で No such file）。`~/.claude/agents/` と `~/.claude/commands/` には別物（plotrip-data-collector.md、rule-md-generator.md、agmsg.md、test.md）があるが、この repo の管理外。Subagent は skill（dispatch）から Agent tool の `model` を指定して呼ぶだけで、agent 定義ファイルは持たない。
- hook 2 本: `hooks/harness-session-start.sh` 73 行（SessionStart、handoff を文脈に注入）、`hooks/harness-scope-check.ts` 39 行（PreToolUse / Bash、`gh issue|pr create` の本文に「追加・変更するもの」の箇条書きが無ければ exit 2 で拒否）。`bin/codingenv` の `HOOKS` 変数に列挙され、`matcher: Bash` と `if: Bash(gh *)` と timeout 10 で登録される。
- installer: `bin/codingenv` 224 行（install / uninstall / status）。symlink 方式（skills、hooks、harness.json、tools）、settings.json は jq で自分の hook だけ追記・削除、`~/.claude/CLAUDE.md` はマーカー区間（`<!-- harness:begin ... -->`）だけ置換、実行前にバックアップ（`.bak-<日時>-harness`）。`status` は drift を検出し exit 1。冪等、退役した hook / env の掃除つき。
- 周辺ツール: `cad/`（Go、非テスト・テスト合計 3,762 行のうちテスト 1,444 行）、`orchd/`（Go、1,460 行のうちテスト 713 行）、`tools/`（薄い 5 本のラッパー: ai-review 4 行、cad 14、codex-localreview 37、codex-probe 25、orchd 11）、`agent/ai-review.ts` 153 行、`agent/bootstrap.sh` 107 行、`deploy/`（GCP VM・worker 起動 shell）。Go 全体で追跡 5,222 行。
- テスト: `test/*.test.sh` 8 本（合計 705 行: ai-review 163、fetch-auth 91、harness 143、hooks 106、opencode-worker 54、scope-check 50、worker-image 94、worker-startup 64）+ Go test。CI（`.github/workflows/test.yml`）が実行するのは `test/hooks.test.sh`、`test/harness.test.sh`、`go test`（cad と orchd）のみ。`test/scope-check.test.sh` は workflow に出てこない（grep で確認、CI 未実行の可能性。他の workflow は未精査）。
- skill 自体を検証するテスト・lint: 無い。`test/*.sh` を `SKILL`/`frontmatter` で grep して 0 件。skills を対象にする検証は `codingenv status` の「symlink が張られているか」だけ。
- docs: `docs/mission.md` 39 行、`docs/context/{business 33, glossary 13, token-saving-spec 136}`。README 112 行（10,278 bytes）。設計判断は GitHub Issue（label `doc:spec`）に置く方針で、`.github/ISSUE_TEMPLATE/spec.md`（26 行）がテンプレート。
- 未追跡の `.claude/design/*.md`（3 本、計 257 行）、`.claude/changegraph/*.yaml`（2 本）、`.claude/handoff/dev-env-upgrade.md` は作業成果物で、changegraph の出力先そのもの。
- `.claude/changegraph/dev-env-upgrade.yaml` が参照する `docs/decisions/0008-*.md` は repo に存在しない（`docs/decisions` ディレクトリ無し）。歴史的な記録で、古い参照が残る例。

### Inferences
- 「skill 数 9・hook 2」は agent-skills（25 skill / hook 4）より小さく、ECC（293 / 24）とは 2 桁違う。ただしこの repo は「skill 集」より「運用基盤」寄りで、行数の大半は Go と shell。
- installer が小さく（224 行）、settings.json を触る範囲を自 hook の追記・削除と CLAUDE.md のマーカー区間に限定している点は、ECC の広い書き込みとの対比になる。

### Gaps
- CI で `scope-check.test.sh` が動くか（他の workflow を含めて）は未確認。
- `test/*.test.sh` を実際に実行してはいない（通過は未確認。`codingenv status` のみ実行し全項目 ok を確認）。

## 2. skill / command / agent の書き方（実例）

### Takeaway
skill はどれも、短い frontmatter（name / description / 任意の argument-hint）＋日本語の番号付き手順＋「やらないこと」節という一貫した型で、20〜50 行。command と agent は別ファイルとして存在せず、skill がスラッシュコマンドを兼ね、Subagent は dispatch の手順文中でモデル指定つきで呼ばれる。

### Cited Findings
- frontmatter のキーは 4 種だけ: `name`、`description`（全 9 本）、`argument-hint`（taskman・design・changegraph・dispatch・setupca の 5 本）、`disable-model-invocation: true`（handoff・igosleep・iwokeup の 3 本、owner が明示的に打ったときだけ動く）。`allowed-tools`、`model`、`metadata` 等は使っていない。pickup だけ `argument-hint` が無い（本文で `<name>` は任意と説明）。
- 典型的な構造（taskman、29 行）: `# 名前 — 一言`、責任の 1 文、「流れ」（矢印で `依頼 → classify → /design → 設計 → /changegraph → Task 一覧 → /dispatch`）、「## 手順」の番号リスト、「## やらないこと」（他 skill の責任を名指しで除外）。
  - 抜粋（`skills/taskman/SKILL.md`）: 「責任は Task を作り出すところまで。配送（`/dispatch`）以降は扱わない。」「`auto=false`（既定）：Task 一覧を見せ、OK が出るまで次に進まない。」
- design（41 行）: 手順 6 つ + Markdown テンプレート（流れ / 境界ごとの概要・Scope・Input・Output / Mermaid Node Graph）。手順 4 に「隣り合う境界の Output と Input が一致しているかを確認する。ずれていたら 2 に戻る」という検査ループを書く。
- changegraph（49 行）: Input / Output を冒頭に明記（「Input：設計」「Output：Task 一覧」）。手順 3 で YAML スキーマ（`nodes: id / pr_title / files / depends_on / done`）を示し、手順 4 で `uv run --with pyyaml python ~/.claude/skills/changegraph/waves.py <yaml>` を実行して並列 wave を計算する。ルール1「files が 1 つでも重なるノードは同じ wave に入らない」、ルール2「depends_on が全部 done か前の wave のノードだけ」。`conflicts_with` を手で書く項目を持たないと明言。`waves.py`（77 行）が決定的に計算する、唯一の skill 付属コード。
- dispatch（28 行）: Input / Output 明記、「何も記録しない」。Subagent への指示の型（文脈ゼロで始まる前提で Task 本文をそのまま渡す、報告は 15 行以内で変更ファイル / 検証コマンドと結果 / 指示と違えた点 / 未解決事項）と、モデル選択表（haiku=転記・整形、sonnet=調査・手順明確な実装、指定なし=設計判断・デバッグ）。`CLAUDE_CODE_SUBAGENT_MODEL` で一律に下げないと書く。agent 定義ファイルの代わりに、この手順文が「Subagent の仕様」になっている。
- pickup（22 行）/ handoff（25 行）: ファイルベースの状態受け渡し。handoff は `.claude/handoff/<name>.md`（30 行以内、見出し固定: 最終更新 / 目的 / 完了条件 / 決定事項 / 却下した案 / 現在の状態 / 次の一手 / 注意）を書いてコミット、pickup が読み直して 3 行で復唱。分岐（名前あり / なし / 複数 / 0 件）を箇条書きの入れ子で網羅している。handoff の `Issue: #<番号>` の行は pickup が読む機械可読な契約で、「この形でない ID は無視し gh に渡さない」と書いてある。
- setupca（24 行）: 対話で `harness.json` を書く。キーの表（`tickets.system` / `tickets.spec.label` / `tickets.project` と「読む側」の列）と 2 つの JSON 例を持つ。「読む側は検証しないので、正しい形はここで書くことで担保する」と、検証しない設計を明記。
- igosleep / iwokeup（23 / 27 行）: 運用系。本番操作（`gcloud compute ssh cad-2 ... systemctl ...`）は Claude が実行せず、`!` 行の形で owner にコピペさせる。
- 文体: 全部日本語、指示文は「〜する」「〜しない」で言い切り、原則と理由を括弧書きで添える。ECC / agent-skills にある Rationalizations 表や Red Flags 節、Verification チェックリストは無い。例は最小限（design のテンプレートと 2 行のサンプル程度）。
- command: skill 名 = スラッシュコマンド名。`~/.claude/commands/` へは何も置かない（`codingenv install` は skills / hooks / harness.json / tools のみ配備）。
- agent: 定義ファイル無し。`~/.claude/agents/` の 2 ファイルはこの repo と無関係。

### Inferences
- 1 skill = 1 責任で、責任の境界を「やらないこと」節で他 skill 名を挙げて閉じる書き方が全 skill に共通する。agent-skills の固定構造（Overview〜Verification）とは違い、こちらは「Input / Output と手順と非責務」の型。
- command 層と agent 層を持たないため、agent-skills の「skill=手順、agent=人格、command=起動」の 3 層に対し、こちらは skill 1 層に畳んでいる。

### Gaps
- skill の本文が実際にどれだけ想定どおり発火するか（routing の質）は未検証。eval も無い。
- `disable-model-invocation` と description の関係の挙動は Claude Code docs では確認していない（repo 内の記述のみ）。

## 3. 部品間の依存関係（明示 / 暗黙）

### Takeaway
依存はほぼすべて本文中の名前参照（`/design` など）で、宣言する欄（frontmatter・manifest）も検証もない。明示的な機械可読の契約は、`harness.json` のスキーマ、`waves.py` が読む changegraph YAML、handoff の `Issue: #N` 行、`codingenv` の HOOKS/SKILLS リスト、の 4 つに限られる。

### Cited Findings
- スキル間の参照（grep で `/<名前>` を数えた、自己参照を含む）:
  - taskman → design、changegraph、dispatch（本文の手順 1〜5 と description）。
  - design → changegraph、taskman（「〜から呼ばれ、出力は /changegraph に渡る」）。
  - changegraph → taskman、design、dispatch。
  - dispatch → taskman（分解・設計は taskman の仕事）。
  - pickup ↔ handoff（互いに名指し）。igosleep → handoff、iwokeup → pickup、igosleep ↔ iwokeup（対）。
  - setupca は他 skill から `/setupca` で呼ばれない（grep 0 件）。`harness.json` を介して、pickup・design・handoff・orchd/wake.md と間接に結びつく。
- チェーンの向きは、各 skill の description に「/taskman から呼ばれ、出力は /changegraph に渡る」と両側で書いて二重に示す（例: design の description、changegraph の description）。frontmatter の `requires` / `depends-on` のような欄は無い。
- `harness.json`（`global/harness.json` は 1 行 60 bytes: `{"tickets":{"system":"github","spec":{"label":"doc:spec"}}}`）: 優先順位は project の `.claude/harness.json` > `~/.claude/harness.json`（global の symlink）> 未設定。README は「読むのは skill（pickup・design・handoff）と orchd/wake.md」と書き、setupca の表の「読む側」列が同じ内容を再掲する。スキーマは README と setupca の表・JSON 例に prose として存在するのみ（JSON Schema なし、validator なし）。「知らないキーは無視する」「読む側は検証しない」と明記。読む側が 4 箇所（3 skill と wake.md）に分散し、それぞれ本文で読み方を書いている（例: pickup は「github 以外・label 無し・gh 不可のときは何も言わずに飛ばす」）。
- changegraph YAML: `.claude/changegraph/<名前>.yaml` は `waves.py` が読む唯一の構造化契約（`nodes[].id/pr_title/files/depends_on/done`）。skill 内に書式例あり。実ファイル `.claude/changegraph/dev-env-upgrade.yaml`（63 行）はこの書式で実運用されている（`done: true` が付く）。
- 配備の宣言: `bin/codingenv` の `SKILLS=$(ls -d */ ...)`（skills/ 直下を走査して自動登録）、`TOOLS=$(ls tools)`（同様）、`HOOKS='SessionStart:harness-session-start.sh PreToolUse:harness-scope-check.ts'`（hook だけ手書きの一覧）。新しい skill は走査で自動的に配備される（共有の一覧ファイルを触らない設計、CLAUDE.md の並列開発ルールに合致）。
- 絶対パス参照: changegraph が `~/.claude/skills/changegraph/waves.py` を、skill 群が `~/.claude/harness.json` を前提にする。`codingenv install` 済みの環境でのみ成立する暗黙の依存。
- 検証: `codingenv status` は配備の drift（symlink・hook 登録・CLAUDE.md 節の一致）を見る。skill 間の dangling 参照（例: 存在しない `/foo` への参照）を検出するものは見つからない（skill を対象にするテスト 0 件）。
- 外部ツールへの暗黙の依存: `gh`、`jq`、`uv`（`--with pyyaml`）、`node`、`cad`/`orchd`/`ai-review` 等（README、wake.md、CLAUDE.harness.md の prose）。skill 内で前提チェックするのは handoff / pickup の「gh が使えないとき飛ばす」程度。
- global の CLAUDE.harness.md は skill 名（`/taskman`、`/dispatch`、`/handoff`、`/pickup`、`/design`、`/setupca`）を本文で直接参照し、skill 側は CLAUDE.harness.md の「コード設計」節を「harness の『コード設計』節」と参照する（design 冒頭）。CLAUDE の節と skill が相互に名前で結ばれる。

### Inferences
- 依存宣言の有無では、ECC・agent-skills と同じく frontmatter 宣言なしで、名前参照の暗黙依存。違いは、参照先が 9 本しかなく全体を人が把握できる点と、レイヤ（taskman → design → changegraph → dispatch）が直線的で循環が無い点（grep 上、taskman↔design のような「呼ぶ / 呼ばれる」の双方向記述はあるが呼び出しの循環ではない）。
- 「スキーマは書く側（setupca）で担保し、読む側は検証しない」は意図的な設計で、検証の仕組みを持たないぶん軽い。一方で dangling 参照や、読む側 4 箇所の読み方の食い違いを機械では検出できない（未検証の推論）。

### Gaps
- ECC のような依存グラフ / registry に相当するものが本当に無いかは、repo 全体ではなく skills/hooks/bin/test/CI の範囲を見て判断した。`.claude/design/*.md`（3 本）は精読していない。

## 4. 常時コンテキスト負荷（推定）

### Takeaway
常時ロードされるのは、`~/.claude/CLAUDE.md` 内の harness 節（3,360 chars / 6,824 bytes）と 9 skill の description（計 1,309 chars / 2,679 bytes）が主。ECC（full で約 37k token）や agent-skills（約 3k）の推定と比べ、description だけなら 1k token 未満で最小クラス。ただし日本語なので chars/4 は過小になる。

### Cited Findings
- 方法: description 行を frontmatter から正規表現で抜き、Python で文字数（chars）と UTF-8 バイト数を計測。トークン数は換算式を 2 通り置く（下限 = chars/4、上限側 = bytes/4）。日本語は 1 文字 1〜2 token 程度になりうるという一般的な見立てを置くが、この repo のトークナイザでの実測はしていない（`/context` 未実行）。
- skill description（chars / bytes）: changegraph 133/289、design 160/338、dispatch 115/211、handoff 95/199、igosleep 219/415、iwokeup 197/373、pickup 104/210、setupca 151/343、taskman 135/301。合計 1,309 chars / 2,679 bytes。
  - 換算: chars/4 ≈ 330 token（下限）、bytes/4 ≈ 670 token。日本語が 1 char ≈ 1 token なら約 1,300 token（未検証の仮定）。
  - `disable-model-invocation: true` の 3 本（handoff・igosleep・iwokeup）は、Claude Code の仕様上モデルへ description を出さない可能性があるが、docs では未確認。含めて数えている。
- CLAUDE.md（グローバル）: `~/.claude/CLAUDE.md` は 70 行、3,964 chars、8,140 bytes（`wc -lcm`）。うち harness 節は `global/CLAUDE.harness.md` の 42 行、3,360 chars、6,824 bytes（`codingenv status` が「up to date」）。換算: 8,140 bytes / 4 ≈ 2k token（上限側）、3,964 chars / 4 ≈ 1k token（下限）。※会話に注入された CLAUDE.md の全文は上の 70 行より長く見えるが、ファイル計測値を採用。差の原因は未確認。
- project の CLAUDE.md（この repo）: 38 行、2,084 chars、3,698 bytes。先頭に「目安：全体で150行以内」と自己制約。
- SessionStart hook（`harness-session-start.sh`）は、handoff が 1 件なら `head -n 60` で最大 60 行を毎セッションの `startup|clear|compact` で注入、複数なら 1 行ずつの一覧、無ければ何も出さない。resume / fork では出さない。件数と内容で変動する（この repo の handoff は 1 件）。
- agent / command の description: この repo は 0（`~/.claude/agents|commands` の 4 ファイルは repo 外で、agmsg.md と test.md はそれぞれ約 30 KB）。
- skill 本文（呼び出し時のみ）: 24,176 bytes 全体、最大 iwokeup 4,328 bytes。1 回の発火で読まれるのは 1〜数本（例: taskman → design → changegraph → dispatch の全 4 本で 1,888 + 1,725 + 2,577 + 2,619 = 8,809 bytes、bytes/4 ≈ 2.2k token 上限側）。

### Inferences
- description だけなら常時 1k 未満で、ECC の minimal（約 17k）の数十分の 1、agent-skills の約 3k の 1/3〜1/2 程度と見積もれる（両者の値は報告書からの引用で、換算条件は揃っていない）。
- 実際の常時コストの主役は skill description ではなく、グローバル CLAUDE.md（harness 節 + 個人ルール）。合計 8 KB。ここは harness 側の設計判断で増減する。

### Gaps
- `/context` による実測は未実施。トークナイザ換算は推定。`disable-model-invocation` skill の description が一覧に載るかは未確認。

## 5. 設計思想（コードから読み取れること）

### Takeaway
「最小構成・1 部品 1 責務・Input/Output で境界を切る・状態は外部（Issue と handoff）に置く・repo を唯一の正本にして symlink で配る」が、コードと CLAUDE.md の両方に一貫して出ている。検証は installer と hook に集中し、skill の中身は検証しない。

### Cited Findings
- 小さい足跡: CLAUDE.md（グローバル）に「最初から機能に厚みを持たせない。機能・実装・テストは最低限動くもの（少し足りないくらい）から始め」とあり（`global/CLAUDE.harness.md`）、実際に skill は 20〜50 行、hook は 2 本、setupca が扱うルールは `tickets` の 1 種だけ。
- 1 部品 1 責務: 各 skill が「やらないこと」で他 skill の責務を名指しで除外（taskman は設計・分割・登録・配送をしない、dispatch は記録・worktree・wave 進行・完了判断をしない）。ファイル分割の規約も「1 ファイル 1 責務」「共有ファイルは自動登録に」（`global/CLAUDE.harness.md`、`bin/codingenv` は skills/tools を走査で自動登録）。
- 関数スタイルの Input / Output: 「どの境界も Input を受け取り Output を返すもの」と CLAUDE.harness.md に書き、changegraph と dispatch は冒頭に Input / Output を箇条書きし、design は境界ごとに概要・Scope・Input・Output のテンプレートを課し、Mermaid の Node Graph を必須にする。taskman の「流れ」は `X → a → x → b → y` の矢印表記。
- 自前でものを持たない: 「キューや代替のタスク管理を自作しない」。Task = GitHub Issue、状態 = handoff ファイル + Issue コメント。spec = label `doc:spec` の Issue。skill が `gh` を直接使う。
- オーケストレーター専任: main は実装せず、`/taskman` → `/dispatch`（Subagent）。完了判断は main が done コマンドを再実行して行う（鵜呑みにしない）。dispatch がモデルを毎回明示し、報告形式を 15 行以内に固定。
- 人間の確認を残す: 本番操作（igosleep / iwokeup）は Claude が実行せず owner にコピペさせる。`~/.claude` への配備は Claude ではなく owner が `codingenv install` を実行する。
- 可逆性: install はバックアップと冪等性、uninstall は「自分が入れたものだけ」を除去（symlink のリンク先を確認して消す）、`status` で drift 検出（読み取りのみ、Claude が実行可）。ECC の broad な書き込みとの対比になる。
- フェイル・オープンな hook: `harness-scope-check.ts` は JSON パース失敗や対象外コマンドで exit 0（許可）、対象は `gh (issue|pr) create` に限定。session-start は jq が無い・読めない場合も exit 0 で静か。ECC の fail-closed dispatcher との対比。
- プロジェクトへの持ち込み最小: 各 project が持つのは任意のデータだけ（`.claude/handoff/`、`docs/context/`、`harness.json`）で、ハーネス本体は global（#92 が「テンプレートを各 repo にコピーする方式はやめた」と README に記載）。
- 事実確認の規律: 「出力・インターフェースを推測で断言しない」「Claude Code の仕様は docs の原文で確認する。WebFetch の要約は誤答したことがある」（CLAUDE.md）。
- 規約で守る並列開発: 1 PR = 1 変更・差分目安 400 行、ChangeGraph（files の重なりで wave を出す）、PR 本文に検証コマンドとチェックリスト、merge に独立レビュー必須（`ai-review`）。skill と Go/shell ツールに落ちているのは changegraph（waves.py）、scope-check hook（PR / Issue 本文の形式）、ai-review 系ツール。それ以外は CLAUDE.harness.md の prose のままで、強制機構は無い。
- 検証の置き場: installer と hook にはテストがあり（`test/harness.test.sh` 143 行で 30 以上の check、`test/hooks.test.sh` 106 行、`test/scope-check.test.sh` 50 行）、skill の文面には無い。skill の品質は本人の使用とレビューに委ねられている。

### Inferences
- 設計は「宣言的な manifest や lint を足さず、規約と直線的な参照で複雑さを抑える」方向で、agent-skills（lint と routing eval で品質を保つ）や ECC（CI の validator 群と registry）と反対の位置にある。9 skill の規模なら成立するが、skill が増えたときに dangling 参照・description の肥大・harness.json 読み手の不整合を検出する仕組みが無いのが弱点候補（推論）。
- 取り込める可能性が高いのは、ECC / agent-skills 側の skill lint（frontmatter・行数・参照先の存在）と、description のトリガー判定 eval。この repo の「skill 走査で自動登録」方式とは相性が良く、共有ファイルを増やさずに足せる（推論）。

### Gaps
- `.claude/design/*.md`（placement-strategy 142 行など）、`cad/README.md`、`orchd/README.md` は概要のみで精読していない。
- 実運用での skill の発火精度・context 消費は未計測。skill 間の依存が実際にどれだけ守られているか（例: taskman が本当に design を呼ぶか）は動作確認していない。
