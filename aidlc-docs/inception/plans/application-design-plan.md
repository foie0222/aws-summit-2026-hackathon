# Application Design Plan — noshi

> このファイルは Application Design の **計画 + 質問 + チェックリスト** を兼ねます。
> §A の質問に `[Answer]:` タグで回答し、最後に「完了」とお知らせください。
> 回答内容に応じて §B の生成ステップを実行します。

---

## §A 質問一覧 (Application Design レベルの設計判断)

各質問について `[Answer]:` の後にレターを記入してください。
どの選択肢も合致しない / 判断材料が足りない場合は **`E) おすすめに任せる`** を選んでください (推奨候補をその場で採用します)。

---

### Question 1: フロントエンド配信形態
ユーザーが触る Web アプリの配信形態はどれにしますか? (TBD-T3)

A) **Vite + React (純 SPA)** + S3/CloudFront 配信 + 別途 API バックエンド
B) **Next.js (SSR + API ルート)** — フロントとサーバ機能を 1 つに統合 (Amplify Hosting / AWS App Runner)
C) **Remix** — SSR 重視
D) **モバイル / ハイブリッド** (PWA など)
E) おすすめに任せる (推奨: B Next.js — Amplify と相性良 / ストリーミング SSE が扱いやすい / 2〜4 名チームの並行作業に十分)
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

### Question 2: AgentCore (Bedrock AgentCore) の使い方
プロジェクト環境に AgentCore MCP が整備済みです。どこまで AgentCore に寄せますか?

A) **AgentCore 全部活用** — エージェント実行 / ツール呼び出し / メモリ / Guardrails をすべて AgentCore に委ね、Lambda は薄い API ラッパーに留める
B) **AgentCore 部分活用** — エージェント実行と Guardrails は AgentCore、メモリと永続データは独自 (DynamoDB)
C) **AgentCore 不使用** — Lambda 上でカスタムエージェントループを実装、Bedrock InvokeModel を直接呼ぶ
E) おすすめに任せる (推奨: B — 永続 Person ledger は DynamoDB が適切で、推論側だけ AgentCore に寄せると運用が楽)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 3: 文化ルールエンジンの実装方式 (TBD-T2)
「半返し / 三分返し / 年齢別お年玉相場 / 文化的締切日 / 110 万円贈与税枠」などのルールをどう実装しますか?

A) **コード内ルールテーブル** (TS/Python の純粋関数 + 定数テーブル) — 高速・PBT 適用しやすい・テスト容易
B) **JSON / YAML データ + ルールエンジン** — ルールを設定ファイルに分離、改訂しやすい
C) **RAG (Bedrock Knowledge Bases)** — 文化マナー文献をナレッジ化し LLM が引く
D) **ハイブリッド: コア計算はコード (A) + 微妙な判断・文面ニュアンスは LLM (D 内で組み合わせ)**
E) おすすめに任せる (推奨: D ハイブリッド — 計算は決定的に / 礼状文面と境界ケースは LLM、PBT は A 部分のみに適用)
X) Other (please describe after [Answer]: tag below)

[Answer]: D

---

### Question 4: 画像 OCR / 構造化抽出の実装方式 (TBD-T1)
ご祝儀袋・メッセージカード画像から金額・贈り主・用途等を抽出する実装は?

A) **Bedrock Claude マルチモーダル ワンショット** (画像 → 構造化 JSON を 1 回で取得)
B) **Amazon Textract で OCR → LLM で構造化** (2 段構成、Textract は日本語縦書きに強い)
C) **ハイブリッド** — Bedrock 既定、Textract は失敗時 / 縦書き判定時のみ
E) おすすめに任せる (推奨: A — ワンショット構成でレイテンシと運用が単純。失敗時は LLM のリトライで対処)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 5: 認証方式 (TBD-U1)
永続ユーザー向けのアカウント認証方式はどれにしますか?

A) **Cognito ホスト UI** (一番楽、UI は Cognito が提供)
B) **Cognito + 自前 UI (メール+パスワード)** — UI 自由度高、実装コスト中
C) **Cognito + マジックリンク (パスワードレス)** — モダン、ハッカソン映え◎
D) **Cognito + ソーシャルログイン (Google / Apple)** — 登録ハードル低
E) おすすめに任せる (推奨: C マジックリンク — パスワード管理不要、来場者デモは別の自動ログイン経路で対応)
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

### Question 6: Person ledger のデータストア構造
Person 中心の元帳 (Receive と Give を統合) を DynamoDB でどう保持しますか?

A) **Single-table 設計** — PK=`USER#<userId>` / SK=`PERSON#<personId>#<recordType>#<timestamp>` などで 1 テーブルに集約
B) **Multi-table 設計** — Persons / Receives / Gives / Images を別テーブルに分離 (関心の分離)
C) **RDB (Aurora Serverless v2)** — 関係構造を SQL で扱う、JOIN で集計
E) おすすめに任せる (推奨: A Single-table — Person タイムライン読み出しを 1 クエリで返せる、コスト効率最良。PBT 対象の整合性ロジックは純粋関数化)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 7: 発注 / 礼状送信の「現実度」(TBD-R1, TBD-R2)
ハッカソン β 版での発注・礼状はどこまで実体験させますか?

A) **完全モック** — 「発注/送信しました」の画面表示のみ、外部送信は一切なし
B) **礼状はメール送信 (SES) で実体験 + 品物発注は完全モック**
C) **礼状はメール送信 + 品物発注は外部 EC API ダミー連携 (本番 EC ではなく専用サンドボックス)**
D) **画面表示 + プリンタ風プレビュー** (郵送は最終的に手動)
E) おすすめに任せる (推奨: B — 礼状は SES で来場者のスマホに届くと「動いた感」が出る、品物は Bedrock Guardrails のテストにもならず時間対効果の観点でモック)
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

### Question 8: モノレポ / IaC ツール選定
コードベースの構成と Infrastructure as Code (IaC) ツールはどれにしますか?

A) **モノレポ (pnpm workspaces) + AWS CDK (TypeScript)** — TypeScript 1 言語で全部
B) **モノレポ (pnpm workspaces) + AWS SAM (YAML)** — サーバーレスに最適化
C) **モノレポ + Terraform** — クラウド非依存
D) **Multi-repo + 各レポで IaC** — 大規模向け、本プロジェクトには過剰
E) おすすめに任せる (推奨: A モノレポ + CDK TypeScript — フロント/バック/IaC が同言語で型共有可、2〜4 名チームに最適)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## §B 生成ステップ (回答確定後に実行)

> 上記 §A の回答が確定した後、以下を順番に実行します。各ステップを完了したら `[x]` に更新します。

- [x] B1. 各質問の回答を整理し、設計判断テーブルを作成 (components.md §設計判断テーブル / application-design.md §1)
- [x] B2. `components.md` を生成 (10 コンポーネント: C-WEB / C-AUTH / C-PERSON / C-RECEIVE / C-GIVE / C-ANALYTICS / C-DELIVERY / C-AGENT / L-RULES / L-MODELS)
- [x] B3. `component-methods.md` を生成 (各コンポーネントのメソッドシグネチャ、TypeScript 型)
- [x] B4. `services.md` を生成 (サービス境界 + 9 つのオーケストレーション O1〜O9 + 通信パターン + 横断関心事)
- [x] B5. `component-dependency.md` を生成 (依存マトリクス + データフロー Mermaid + プロトコル一覧 + 循環依存チェック)
- [x] B6. `application-design.md` (統合ドキュメント) を生成
- [x] B7. Mermaid + テキスト代替で コンテキスト図 (§2) / データフロー図 (component-dependency.md §) / S1 シーケンス図 (§4) を作成
- [x] B8. 要件 (FR-R / G / A / L / U) ↔ コンポーネント トレーサビリティを application-design.md §6 に追加
- [x] B9. Story (US-XX) ↔ コンポーネント マッピングを application-design.md §7 に追加
- [x] B10. TBD 解決状況を application-design.md §8 に整理 (確定: TBD-T1 / T2 / T3 / U1 / R1 / R2 / 部分: NAME, S1 / 持ち越し: U2, T5-T8, S2, L1-L2)
- [ ] B11. 完了チェック — `aidlc-docs/aidlc-state.md` を更新し、ユーザーに承認依頼を提示

---

## §C 生成しない方針 (Out of Scope)

- **詳細ビジネスロジック** (具体的な計算式、しきい値、エラーハンドリングの個別フロー) → Functional Design (per-unit, CONSTRUCTION) で扱う
- **NFR の具体値** (レスポンスタイム数値、キャパシティ、スケーリングポリシー) → NFR Requirements / NFR Design で扱う
- **インフラ設定値** (VPC / リージョン / インスタンスサイズ / Cognito UserPool 詳細) → Infrastructure Design で扱う
- **コードレベル実装** → Code Generation で扱う
- **デモアカウントの TTL や事前データ規模 (TBD-U2 / TBD-S2)** → Infrastructure Design / Functional Design で扱う
