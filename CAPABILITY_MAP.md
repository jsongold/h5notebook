# Capability Map: h5notebook

目的：学習の目標管理・日記公開・コンテンツ収集アプリを、独立してテストできるモジュールに分け、作る順番を決める。

| Module id | 責務 | 依存先 |
|---|---|---|
| auth | 自分1人だけがログインできる認証 | — |
| llm | 複数の LLM プロバイダを共通の口で呼ぶ。どのモデルを使うかは呼び出し側のモジュールが決める | — |
| goals | Goal / Milestone / 月・週・日目標の CRUD とデータモデル | auth |
| agent-chat | チャット UI と、Agent が goals などをツールとして操作する仕組み | goals, llm |
| classifier | 入力（日記・コンテンツ）がどの目標に当たるかを自動判定する | goals, llm |
| journal | 学習日記の作成・保存と、公開ページの表示 | goals, classifier |
| collector | コンテンツ収集（最初は URL の受け口から） | classifier |
| publisher | X / Medium / メルマガ / YouTube / Instagram への配信 | journal |

作る順番: auth, llm → goals → agent-chat, classifier → journal, collector → publisher

MVP: auth, llm, goals, agent-chat（チャットで目標を登録できる状態）

## 決定事項
- classifier は独立したモジュールにする
- LLM は複数のプロバイダを切り替えられるようにし、どれを使うかは各モジュールが判断する
- llm の実装には Vercel AI SDK と AI Gateway を使う（暫定）。タスクに応じたモデルの自動選択が必要になったら、OpenRouter の auto を AI SDK 経由で足す

## 未決定
- 日記の正本（GitHub の Markdown か Supabase か）
- 認証の方式（無料であることが条件）
- フロントのフレームワーク（Vercel にデプロイする前提）
- GitHub / Google との連携範囲
- publisher の自動化の度合い（下書きまでか、完全自動か）
- collector のスコープ
