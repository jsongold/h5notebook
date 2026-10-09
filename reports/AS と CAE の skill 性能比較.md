# AS と CAE の skill 性能比較

目的：addyosmani/agent-skills（AS）と codingagentenv（CAE）の skill 本文が、agent をどれだけ良い結果に誘導するか（「性能」）を、機能が重なる組ごとに 8 つの基準で比べ、CAE に取り込む価値のある技法と、取り込むと肥大化するものを切り分ける。

前提：構造・hook・installer・footprint・安定性は既存レポート（`ECC universal OSS 調査.md`、`codingagentenv と ECC の違い.md`）が扱っており、ここでは本文の内容だけを見る。人気・保守状況は判定しない。

対象コミット：AS `2686b62`（2026-09-25）、CAE `17fcab9`（2026-09-30）。AS は `mktemp -d` に shallow clone し、確認後に削除した。行番号は各 SKILL.md の `cat -n` の値。AS のパスは repo 相対、CAE は絶対パス。

## 結論（先に）

- **AS が強い基準**：(a) トリガーの検証、(c) 手抜きへの防御、(d) 完了条件の明示、(h) 可搬性。全 25 skill が Overview / When to Use / Common Rationalizations / Red Flags / Verification を持ち、linter と offline eval で検証されている（実行して確認：25 skill lint pass、trigger rank-1 89/89）。
- **CAE が強い基準**：(b) 手順の実行可能性（実コマンド・固定書式・計算スクリプト）、(e) skill 間の Input / Output 契約と「やらないこと」による責務の閉じ方、(f) 1 回の読み込みコスト（AS の 1/3〜1/8）、(g) 自環境（/clear・gh・並列 worktree）の端の処理。
- **CAE の最大の穴**：Task に完了条件（done コマンド）の欄が無い。`CLAUDE.harness.md` L6 は「main が完了条件のコマンドを再実行して確認」と言うが、`/changegraph` の Task 書式（L36-41）にも `/dispatch` の報告書式（L18）にも done を書く場所が無く、鎖が切れている。
- **AS の最大の穴**：CAE 的な意味の契約が無い。各 skill が何を受け取り何を返すかは `tasks/plan.md` などファイル規約に埋め込まれ、spec-driven と planning は互いに内容を重複させている（spec-driven L180, L196 の「canonical source」注記が重複の自認）。
- 取り込みは「小さく」。AS の Rationalizations 表を丸ごと写すと CAE の低コストが消える。各 skill に 3〜5 行の完了チェックと、実際に agent が省きたがる箇所だけの反論行を足すのが釣り合う。

## 1. 機能の対応表と片側にしか無いもの

| CAE | AS の対応物 | 重なり方 |
|---|---|---|
| taskman | using-agent-skills（振り分け）、spec-driven-development の Gated Workflow（承認ゲート） | 依頼の分類と、設計→分割→配送の流れ |
| design | api-and-interface-design、spec-driven-development Phase 1、documentation-and-adrs（設計を Issue/PR に残す） | 境界と契約の設計、設計判断の記録 |
| changegraph (+waves.py) | planning-and-task-breakdown、git-workflow の worktree 節、code-review の Change Sizing | Task 分割、並列化、PR サイズ |
| dispatch | `commands/build.toml`、`references/orchestration-patterns.md`、incremental-implementation の「Working with Agents」 | Subagent への委譲（AS には skill が無い） |
| pickup / handoff | context-engineering の「Restartable Session Boundaries」（L123-137）、planning の「Never overwrite an incomplete plan」（L150-155） | セッション境界の引き継ぎ |
| setupca | context-engineering Level 1（rules file）| 弱い。CAE は設定 schema、AS は CLAUDE.md の中身 |
| igosleep / iwokeup | 無し | 夜間運用の runbook |

**AS にしか無いもの**：TDD、debugging、code-review、security、performance、code-simplification、frontend、ci-cd、deprecation、observability、shipping、source-driven、doubt-driven、interview-me、idea-refine、constraint-driven、browser-testing。CAE はこのうち review（`ai-review`）、PR 本文の検証手順、done 再実行を `global/CLAUDE.harness.md` の箇条書き（L6, L21-22, L28）で持つが、skill ではない。

**CAE にしか無いもの**：Subagent 配送（dispatch）、セッション引き継ぎの固定書式（handoff/pickup）、ファイル重なりから並列 wave を計算する仕組み（waves.py）、project 設定の schema（setupca）、運用 runbook（igosleep/iwokeup）。AS の `orchestration-patterns.md` L316-326 は「orchestrator agent が /spec→/plan→/build を代行する」ことを anti-pattern C として退けており（要約で文脈が落ちる・token 2 倍・人の checkpoint が消える）、CAE の main=orchestrator 設計への正面からの批判になっている。CAE は「Task の本文をそのまま渡し」（dispatch L17）で要約劣化を、報告 15 行制限（L18）で token を抑えているが、token 2 倍の指摘は部分的に当たる。

## 2. 組ごとの比較

基準：(a) トリガー精度 (b) 手順の実行可能性 (c) 手抜き・言い訳・scope creep・偽の完了への防御 (d) 検証・完了条件 (e) Input/Output と次工程への受け渡し (f) 本文の token 対効果 (g) 端の処理 (h) 可搬性。

### 2.1 taskman ↔ using-agent-skills / spec-driven-development

| 基準 | CAE taskman | AS | 判定 |
|---|---|---|---|
| a | 「複数ステップの実装・調査を始める前に使う」（L3）。否定条件は skill 内に無く `CLAUDE.harness.md` L4「1ファイルの小さな修正…は main が直接やる」が担う | spec-driven L20「When NOT to use: Single-line fixes, typo corrections…」、using-agent-skills L16-43 の決定木。eval で positive/negative prompt を検証 | AS |
| b | 5 手順（L16-24）。classify の判定例が具体（L16-17「skill・hook の手順のように、コード以外の形で書かれたロジックも含む」） | spec テンプレート L117-148、仮定の明示 L73-80、曖昧要件の再定義 L158-166 | 同等（役割が違う：CAE は router、AS は spec 作成） |
| c | `auto=false`（既定）「OK が出るまで次に進まない」（L22）のみ | Rationalizations L221-229、Red Flags L231-239。`build.toml` step 4「Treat hedged responses ("looks reasonable", "I guess") as NOT approved」 | AS |
| d | 無し（Output は Task 一覧） | Verification L241-251（「The spec is saved to a file」「The human has reviewed and approved」） | AS |
| e | 流れを 2 行で明示（L11-13）、やらないこと（L26-29） | L181-182「Save the plan to tasks/plan.md and record the task list in the task list target」。工程間はファイル規約 | 同等 |
| f | 29 行 | 251 + 192 行 | CAE |
| g | 無し（/design が失敗、依頼がロジック混在、既に設計済み Issue、のいずれも未記述） | 既存 plan の未完了 task を上書きしない（planning L151-154）、外部 spec ツール（L150-154）、複数 capability の分解（Phase 0） | AS |
| h | 日本語、/design・/changegraph・/dispatch・Issue 登録に依存 | 英語、tracker 非依存（planning L162-163） | AS |

**判定：AS が強い**。taskman は router として薄く正しいが、承認ゲートの定義が甘く（「OK」の判定基準が無い）、失敗経路が無い。

### 2.2 design ↔ api-and-interface-design / spec-driven / documentation-and-adrs

| 基準 | CAE design | AS | 判定 |
|---|---|---|---|
| a | 「DB スキーマとインフラには使わない」（L3）と否定条件を持つ | api の description に否定条件なし。spec は L20 にあり | 同等 |
| b | 6 手順（L12-17）：1 行の流れ→境界を新規/変更/不変に分類→テンプレート→隣接 Output=Input の一致確認→Mermaid→Issue/PR へ `gh issue create --label`。テンプレート L21-37 | api は番号付き手順が無く原則 1〜6 と TS 例（L37-215）。spec は 6 領域＋テンプレート | CAE（手順として）。AS（エラー形式・冪等性・ページングの知識として） |
| c | 「ずれていたら 2 に戻る」（L15）、やらないこと「既存の境界の内部を設計し直す」（L41） | api Rationalizations L323-336、Red Flags L338-350（「A SELECT for an idempotency key followed by an INSERT — that's a race」） | AS |
| d | 手順 4 の一致確認は測れる。終了チェックリストは無い | api Verification L352-367（12 項目）、adrs Verification L279-288 | AS |
| e | 境界ごとの Input/Output が成果物そのもの。Output は /changegraph へ（L3） | api の「Input/Output Separation」L293-311 は TS 型の話で skill の契約ではない | CAE |
| f | 41 行 | 367 + 251 + 288 行 | CAE |
| g | Issue/PR が無ければ harness.json に従い spec チケットを作る、tickets 未設定なら会話に出す、置き換え時の状態行（L17-18） | adrs「Match the existing convention first」L36-44、supersede の lifecycle L93-100 | 同等 |
| h | harness.json・gh・label に依存（手順 6 のみ）。手順 1-5 は可搬 | 汎用 | AS |

**判定：同等（用途が違う）**。設計「手順」としては CAE が明確で安い。設計「知識」（何を境界で守るか）は AS が厚く、CAE の design にはエラー・境界での検証・後方互換の観点が無い。

### 2.3 changegraph (+waves.py) ↔ planning-and-task-breakdown

| 基準 | CAE changegraph | AS planning | 判定 |
|---|---|---|---|
| a | 「設計（/design の出力）を…分割」（L3）と入力前提が明確。否定条件なし | L20「When NOT to use: Single-file changes…or when the spec already contains well-defined tasks」 | AS |
| b | 「触るファイルをコードを読んで列挙する…推測で書かない」（L15）、「1 ノード = 1 PR = 差分目安 400 行以内」（L16）、YAML schema（L21-28）、実コマンド（L29）、ルール 1・2（L30-31）、戻り条件「wave が 1 ノードずつしか並ばない、または serial が多いときは、手順 2 に戻って」（L32） | Task テンプレート L83-104（Acceptance criteria / Verification / Dependencies / Files / Estimated scope）、サイズ表 L127-134、分割の兆候 L137-141（「writing "and" in the task title」）、並列化は散文 L213-219 | CAE（並列安全性は計算）、AS（Task ごとの受入条件） |
| c | 「推測で書かない」、「conflicts_with を持たない。重なりは files から計算する」（L49）、waves.py が循環・未知依存で例外 | Rationalizations L221-229（「The tasks are obvious」→「Write them down anyway」）、Red Flags L231-241（「All tasks are XL-sized」）、未完了 plan の上書き禁止 L151-155 | AS（言い訳への反論）、CAE（機械的強制） |
| d | wave 出力が検証。終了チェック無し。**Task 書式（L36-41）に完了条件の欄が無い** | Verification L243-253「Every task has acceptance criteria / a verification step / No task touches more than ~5 files / The human has reviewed」 | AS |
| e | Input/Output を冒頭に明記（L11-12）、Task → /dispatch | 出力先 `tasks/plan.md` + `tasks/todo.md` か tracker（L144-165）、/build が消費 | 同等 |
| f | 49 行 + script 77 行 | 257 行 | CAE |
| g | YAML は「プロジェクトにコミットしない」（L19）ため、worktree や cloud worker には渡らない。handoff にも changegraph の言及が無い。waves.py は `nodes` 欠落で `KeyError`、files 無しノードは無条件に並列 | 既存 plan の扱い、外部 tracker、2〜3 task ごとの checkpoint | AS |
| h | `uv` と `~/.claude/skills/changegraph/waves.py` の固定パス（L29）。日本語 | 汎用 | AS |

**実測**：`waves.py --selftest` → `selftest ok`。5 ノードのサンプル（c0 済、c1/c2 が c0 依存、c3 が c1・c2 依存、c4 が c0/c3 と `app/registry.py` を共有）で `wave 1: c1, c2, c4 / wave 2: c3 / serial c3 <-> c4: app/registry.py` を出力。ルール 1・2 どおりに動く。`~/.claude/skills/changegraph` は repo への symlink で、L29 のコマンドはそのまま通る。

**判定：CAE が強い（並列分割の機械化）、AS が強い（Task の完了条件）**。総合は同等。CAE は「done の欄」を足すだけで AS の (d) の大半を埋められる。

### 2.4 dispatch ↔ build.toml / orchestration-patterns / incremental-implementation

| 基準 | CAE dispatch | AS | 判定 |
|---|---|---|---|
| a | 「Task を受け取り、次の Agent を呼ぶ」（L3）。slash 呼び出し前提 | `/build` は slash。AS に skill としての dispatch は無い | 同等 |
| b | Task 本文をそのまま渡す（L17）、報告書式「15 行以内で、変更ファイル / 実行した検証コマンドと結果 / 指示と違えた点 / 未解決事項」（L18）、model 選択表（L19-23） | build.toml：8 手順の loop、auto モードの前提（spec の既知パス、`git status --porcelain` で clean baseline、「never `git add -A` blindly」）、停止条件（不可逆操作の列挙） | 同等。AS は実装側、CAE は委譲側 |
| c | 「ログ・diff・ファイル本文を貼らせない」（L18）、「`CLAUDE_CODE_SUBAGENT_MODEL` で一律に下げない」（L23）。Subagent の偽の完了報告への防御は skill 内に無く `CLAUDE.harness.md` L6 が担う | build.toml「Treat hedged responses…as NOT approved」、停止条件「auth/permission changes, destructive data migrations, payments, deletions, deploys…」、incremental の Rationalizations L213-222 | AS |
| d | 無し（Output = 呼び出し）。ただし報告書式が「検証コマンドと結果」を要求するので証拠は取れる | build：RED→GREEN→回帰→build→commit を task ごと | AS |
| e | Input/Output 明記（L11-12）、やらないこと（L26-28）で main loop / taskman / Subagent の責務を切る | `orchestration-patterns.md` L5「the user (or a slash command) is the orchestrator. Personas do not invoke other personas」 | CAE |
| f | 28 行 | build.toml は約 60 行、orchestration-patterns は 370 行（参照用） | CAE |
| g | 依頼をそのまま受けたら 1 Task 扱い（L15）。Subagent の失敗・timeout・部分完了は未記述（timeout は `CLAUDE.harness.md` L26） | build：失敗→debugging skill、blocker 解消後は次の pending task から再開 | AS |
| h | model 名（haiku/sonnet）は Claude Code 依存。日本語 | build.toml も Claude Code / OpenCode 依存 | 同等 |

**判定：同等**。dispatch は意図的に薄い。Subagent へ渡すプロンプトに「不可逆操作は止めて報告」を 1 行足すと AS の停止条件を安く取り込める。

### 2.5 pickup / handoff ↔ context-engineering「Restartable Session Boundaries」

| 基準 | CAE | AS | 判定 |
|---|---|---|---|
| a | handoff は `disable-model-invocation: true`（L4）で user 専用、pickup は「『前回の続き』と言われたら使う」（L3） | description は「Use when starting a new session, when agent output quality degrades…」と広く、353 行中 15 行が該当。用途に対してトリガーが薄まる | CAE |
| b | handoff：名前の kebab-case 検査（L9）、再利用・衝突 `-2`（L14）、30 行制限と固定見出し（L16-17）、`Issue: #<番号>` 形式（L20）、`git add <path>` / `git commit <path>` で自分のファイルだけ（L23）。pickup：0/1/複数件の分岐（L11-14）、`gh issue list --label`（L15）、`git status` / `git log --oneline -10` でずれ確認（L19）、4 項目の復唱（L21） | L125-131 に残すべき 5 項目、L136「read the rules, spec, plan, task status, and actual git status before acting」。書式・場所・命名は無い | CAE |
| c | 「他セッションの未コミット変更は巻き込まない」（handoff L23）、「選ぶまで作業を始めない」（pickup L13）、「他セッションの Issue を、自分のものとして進めない」（L20）、旧 Task ID を `gh` に渡さない（L18）、「確認を待たずに進めてよいのは…具体的で、ずれが無い場合だけ」（L22） | L136「Re-run verification when its recorded baseline is missing, the code has moved…」「Do not infer approval from a previous conversation unless the durable artifact records it」 | 同等（CAE は具体、AS は検証の再実行を要求） |
| d | 30 行・固定見出し・commit が証拠。pickup の復唱が理解の証拠 | この節に checklist 無し | CAE |
| e | handoff の Output（固定見出し）= pickup の Input。対になっている | L137「The skill defines the handoff contract」と言うが書式は無い | CAE |
| f | 25 + 22 行 | 353 行の skill を丸ごと読む | CAE |
| g | PROGRESS.md 移行（handoff L15、pickup L14）、gh が無ければ黙って飛ばす（pickup L15）、旧 env var の警告（L16） | 汎用 | CAE |
| h | gh・harness.json・SessionStart hook・日本語 | 汎用 | AS |

**判定：CAE が強い**。AS から取るべきは 1 点、pickup に「完了条件のコマンドを再実行して baseline を取り直す」を足すこと。

### 2.6 setupca ↔ context-engineering Level 1

重なりは弱い。setupca は schema 表（L13-18）と JSON 例（L20-22）を持ち「ほかのキーは消さない」「`~/.claude/` には直接書かない」（L25）と境界を切る。ただし「対話で聞き」（L3, L11）と言いながら質問文が無く、書いた JSON を検証する手順も無い（README は「読む側は検証しない」と明記）。AS の L43-72 は CLAUDE.md の中身のテンプレートで目的が違う。**判定：比較不能。setupca は質問一覧と `jq` 検証を足すべき**。

### 2.7 igosleep / iwokeup（CAE のみ）

runbook として構造は良い。本番操作を Claude が実行せず owner にコピペで渡す（igosleep L9）、貼られた出力を確かめてから次へ（L15, L22）、「取れない範囲…は推測で書かない」（iwokeup L26）、「『レビュー待ち』などと決めつけない」（L24）。弱点は可搬性ゼロ（`--project suggestorder-dev --zone us-central1-a` の固定、#101/#102/#65 の Issue 参照）だが、これは意図した設計。AS に対応物は無い。

## 3. 基準ごとの総合判定

| 基準 | 判定 | 根拠 |
|---|---|---|
| (a) トリガー精度 | AS | 全 skill に「Use when」＋否定条件、offline eval で positive 89/89 rank-1、negative prompt 付き。CAE は slash 専用（handoff/igosleep/iwokeup は `disable-model-invocation`）で構造的に誤発火しないが、自然文からの発火は未検証。AS の evals/README は skill 発火が確率的（7 回中 5 回）で description の 1 句で発火率が変わると報告しており、CAE の slash 専用設計はこの揺れを避けている |
| (b) 実行可能性 | CAE | 実コマンド・固定書式・計算スクリプト。例外は changegraph の Task に done 欄が無いこと |
| (c) 防御 | AS | 全 skill に Rationalizations / Red Flags。CAE は点の防御（推測で書かない・決めつけない・巻き込まない）と機械的防御（waves.py、`disable-model-invocation`） |
| (d) 検証・完了条件 | AS | 全 skill に Verification checklist、`references/definition-of-done.md`。CAE は `CLAUDE.harness.md` L6 と dispatch の報告書式に分散 |
| (e) 契約・受け渡し | CAE | Input/Output 行（changegraph, dispatch）、やらないこと（4/9 skill）、handoff→pickup の固定書式。AS はファイル規約に埋め込み、spec-driven と planning が重複 |
| (f) token 対効果 | CAE | 1 回の読み込み：CAE 約 0.6〜1.9k token（日本語の概算、tokenizer 未計測）、AS 約 2.4〜5.2k（chars/4）。ただし CAE は `CLAUDE.harness.md` 42 行が常時載る |
| (g) 端の処理 | 分かれる | CAE は自環境の端（/clear、gh 不在、PROGRESS.md 移行）に厚い。AS は工程の端（既存 plan、外部 tracker、blocker 後の再開）に厚い |
| (h) 可搬性 | AS | CAE は日本語・gh・harness.json・uv・固定パス・GCP 固定 |

**総合**：CAE は「自分の環境で agent を正確に動かす手順書」として AS より実行可能で安い。AS は「agent が手を抜く瞬間に踏みとどまらせる仕組み」と「done の定義」で一貫して上。どちらが上かは一つに決まらないが、**CAE が今すぐ埋められる穴（done 欄、終了チェック）は小さく、AS が CAE から取るもの（契約の明示）は構造変更を要する**。

## 4. 検証したこと・していないこと

実行して確認：
- `waves.py --selftest` → `selftest ok`。サンプル YAML で wave と serial を正しく計算。`~/.claude/skills/changegraph` → repo の symlink。
- AS `node scripts/validate-skills.js` → 25 skill、0 error。`node scripts/run-evals.js`（Tier 2、offline）→ 141 checks pass、rank-1 89/89。
- AS の linter を CAE の 9 skill に当てると全件 error（「Use when」が英語正規表現、必須 5 節が無い）。これは linter の英語前提で、CAE の description が「〜前に使う」「〜と言われたら使う」と日本語でトリガーを書いていることは linter が読めない。CAE の品質判定ではなく、AS の検証器の可搬性の限界。
- CAE `test/harness.test.sh` は install の symlink だけを見る。skill 本文の検証は無い。

未検証：
- 実際の token 数（`/context` 未計測。数値は文字数からの概算）。
- 自然文からの CAE skill 発火率。
- AS Tier 3（behavioral eval）は token を使うため未実行。README の数値は AS 側の報告をそのまま引用。

## 5. 改善提案

### CAE に取り込む（小さく）

1. **changegraph の Task 書式に「完了条件（done コマンド）」欄を足す**（L36-41）。dispatch の報告書式（L18）の「検証コマンドと結果」と、`CLAUDE.harness.md` L6 の「main が再実行」がこれで繋がる。2 行。
2. **各 skill の末尾に 3〜5 行の終了チェック**。changegraph なら「触るファイルはコードから列挙した / waves を実行した / serial が過半でない / 全 Task に done がある」。AS Verification L243-253 の形だけ借りる。
3. **反論行は agent が実際に省く箇所だけ**。changegraph「ファイルは推測で足りる」、pickup「git log を見れば handoff は不要」、handoff「次の一手は覚えている」。各 2〜3 行。AS の 8〜10 行の表は写さない。
4. **pickup に baseline の取り直し**：「handoff の完了条件コマンドを再実行し、結果を復唱に含める」（AS context-engineering L136 から）。1 行。
5. **taskman の承認ゲートを定義**：「曖昧な返事は OK と見なさない」（AS build.toml step 4 から）。1 行。
6. **dispatch の Subagent プロンプトに停止条件**：「不可逆操作（削除・deploy・secrets・migration）は実行せず報告」（build.toml 手順 6 から）。1〜2 行。
7. **changegraph YAML の引き継ぎ**：handoff の「現在の状態」に `.claude/changegraph/<名前>.yaml` のパスを書くか、コミット方針を決める。cloud worker に渡らない問題への手当て。
8. **setupca に質問一覧と `jq` 検証**。

### CAE に取り込まない（肥大化する）

- Overview の散文、ASCII 図、ドメイン参照知識（REST パターン、TS 例）。CAE の低コストが消える。
- 全 skill への Rationalizations / Red Flags の機械的追加。router 的な taskman・dispatch には効かない。
- 英語の「Use when」句。Claude Code の発火には不要で、AS linter 互換のためだけになる。

### AS が CAE から取れるもの

1. **skill 冒頭の Input / Output 行**（changegraph L11-12、dispatch L11-12）。今はファイル規約に埋もれている。
2. **「やらないこと」による責務の閉鎖**。spec-driven Phase 2/3 と planning の重複を、参照ではなく除外で解く。
3. **並列安全性の計算**（waves.py）。planning L213-219 の散文を機械化できる。
4. **Subagent 報告の上限と禁止事項**（dispatch L18）。orchestration-patterns は「digest」と言うだけで書式が無い。
5. **委譲ごとの model 明示**（dispatch L19-23）。
