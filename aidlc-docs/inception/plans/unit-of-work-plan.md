# Unit of Work Plan — noshi

> このファイルは Units Generation の **計画 + 質問 + チェックリスト** を兼ねます。
> §A の質問に `[Answer]:` タグで回答し、最後に「完了」とお知らせください。
> 回答内容に応じて §B の生成ステップを実行します。

> **ターミノロジー**:
> - **Service**: 独立してデプロイ可能なコンポーネント (例: receive-service)
> - **Module**: Service 内の論理グルーピング
> - **Unit of Work**: 計画上の単位 (= 担当者 / Sprint 単位の作業塊)

---

## §A 質問一覧 (Units Generation で確定が必要な事項)

各質問について `[Answer]:` の後にレターを記入してください。
迷ったら **`E) おすすめに任せる`** を選んでください (推奨候補をその場で採用)。

---

### Question 1: ユニット数のターゲット
2〜4 名チームでの並行開発を想定。ユニットは何個に分けますか?

A) **コンポーネント別 1:1** (10〜12 ユニット — 細かすぎ、3 名担当 = 1 人 3〜4 ユニット)
B) **5 ユニット** (担当人数 ≈ ユニット数、1 人 1 ユニット中心)
C) **6〜7 ユニット** (ドメイン別、一部共有持ち)
D) **3〜4 ユニット** (大粒、各ユニット内部で複数モジュール並行)
E) おすすめに任せる (推奨: **B 5 ユニット** — 4 名チーム前提で各人 1 ユニット主担当 + 1 名がインフラ + 横断レビュー)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 2: Story グルーピング戦略
23 Story をユニットに割り当てる戦略はどれにしますか?

A) **サービス境界別** (1 unit = 1 deployable service): 例えば `receive-service` ユニットは US-04〜US-09
B) **ドメイン別** (Receive / Give / Analytics / Identity / Frontend など): User Stories の Epic 分けに沿う
C) **ユーザージャーニー別**: アカウント立ち上げ / Receive ジャーニー / 履歴・分析 / デモ で機能横断的に分ける
D) **機能関心の組み合わせ** (例: 全ての AI 推論を 1 unit、全ての永続化を 1 unit、全 UI を 1 unit)
E) おすすめに任せる (推奨: **B ドメイン別** — User Stories の Epic と整合し、トレーサビリティが明快)
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

### Question 3: C-WEB と C-AGENT のサブユニット分解 (ラバーダックレビュー §M3)
C-WEB は全 23 Story の UI を、C-AGENT は AgentCore + 5 ツール実装を担う。担当 1 名で抱えるとボトルネック化するため、サブユニット分解を検討。

A) **分けない** — C-WEB / C-AGENT 各 1 ユニットで担当 1 名 (シンプルだがボトルネック)
B) **両方フル分解** — C-WEB を 5 サブ (Auth/Receive/Person/Give/Analytics) + C-AGENT を 3 サブ (Core/Tools-Extract/Tools-Judgment) — ユニット数膨張
C) **C-AGENT のみ 2 サブ** (Core (AgentCore Agent + Memory + Guardrails 構成) / Tools 実装本体)、C-WEB は単一
D) **C-WEB のみ 2 サブ** (Receive UI 系 / その他 UI 系)、C-AGENT は単一 — Receive UI が重いため
E) おすすめに任せる (推奨: **D + C-AGENT 内部モジュール 2 つ** — C-WEB は 2 担当、C-AGENT は 1 担当 + Tools 実装は内部モジュール分け、Q1=B 5 ユニット内部に収める形)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 4: Shared Packages の扱い
共有パッケージ (`L-RULES` / `L-MODELS` (`@noshi/models`) / `@noshi/person-ledger` / `@noshi/auth-context` / `@noshi/logger`) はどう持つ?

A) **1 つの `shared` ユニット** にまとめる (1 名がオーナー、PR レビューで他メンバーも貢献)
B) **個別に複数ユニット化** (各パッケージ 1 ユニット、ユニット数が膨らむ)
C) **ユニットに含めず、全員共通の責務** (PR 横断レビュー、オーナー不在)
E) おすすめに任せる (推奨: **A `shared` ユニット** — 共有パッケージの API は早期 freeze 必要。1 名のオーナーがいるとブロッカー解消が速い)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 5: Infrastructure (CDK) の所有方式
AWS CDK スタック (Cognito / API Gateway / Function URL / DynamoDB / S3 / Bedrock 設定 / SES / EventBridge / Amplify) は誰が持つ?

A) **独立した `infra` ユニット** (1 名フル担当) — 集中管理、ボトルネック化リスク
B) **各サービスユニットが自分の CDK スタックを書く** (分散、整合性は CI で担保)
C) **1 名がメインオーナー、各サービスユニットがオプショナルに contribute** (ハイブリッド)
E) おすすめに任せる (推奨: **C ハイブリッド** — メインオーナーが Cognito / API Gateway / 共有部分を書き、各サービス担当が Lambda 設定スタックを追加)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 6: モノレポレイアウト (Greenfield コード組織戦略)
pnpm workspaces の構造はどれにしますか?

A) **`packages/<unit>/`** — 全ユニットが flat に並ぶ
B) **Nx 風** `apps/<service>/` + `libs/<library>/` — Nx を使うかは別として、論理分離が明確
C) **役割別** `services/<service>/` + `web/` + `infra/` + `libs/` — 直感的でハッカソン向け
D) **役割別 + 詳細** `services/<svc>/` + `apps/web/` + `apps/web-bff/` + `infra/<stack>/` + `libs/<lib>/`
E) おすすめに任せる (推奨: **C 役割別** — シンプルでコードナビゲートしやすい、2〜4 名規模に最適)
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

### Question 7: ユニット並行実行戦略
ユニット間の依存をどう調整しながら並行作業しますか?

A) **階層型** (Sprint 1: 共有パッケージ → Sprint 2: services → Sprint 3: web → Sprint 4: infra) — ボトルネック化
B) **全並行** (Sprint 1 から全員開始、interface contract が崩れる前提でリファクタ) — リスク高
C) **Sprint 0 で contract freeze + Sprint 1 から全並行** — Sprint 0 (3〜5 日) で `@noshi/models` の型定義 / API スキーマ / DynamoDB key 設計を確定 → 以降全並行
D) **Pair-based** — 2 名ペアでローテーション、契約は会話で詰める (チームが小さいとき有効)
E) おすすめに任せる (推奨: **C Sprint 0 freeze + 全並行** — ハッカソン規模で最も予測可能)
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

## §B 生成ステップ (PART 2: GENERATION で実行)

> 上記 §A の回答が確定した後、以下を順番に実行します。各ステップを完了したら `[x]` に更新します。

- [x] B1. 各質問の回答を整理し、ユニット定義テーブルを作成 (unit-of-work.md §1)
- [x] B2. `unit-of-work.md` を生成 (11 ユニットの定義 / 責務 / 含むコンポーネント / 含むストーリー / モジュール内分割案)
- [x] B3. `unit-of-work-dependency.md` を生成 (依存マトリクス + DAG 検証 + 並行可能性 + Sprint 別 Critical Path + リスク要因 + DoD)
- [x] B4. `unit-of-work-story-map.md` を生成 (US-XX ↔ ユニットマッピング、全 23 Story 割当検証完了)
- [x] B5. モノレポディレクトリ構造を `unit-of-work.md` §3 に明記 (役割別レイアウト Q6=C 採用)
- [x] B6. Sprint 0 contract freeze スコープを `unit-of-work.md` §6 に明示 (ドメイン型 / API スキーマ / AgentCore tool スキーマ / DynamoDB key 設計 / L-RULES シグネチャ / requireUserScope / Problem Details)
- [x] B7. ユニット粒度検証 (`unit-of-work.md` §7) — 11 ユニットは細粒寄り、4 名で 1 人 2〜3 ユニット担当となる点を明示
- [x] B8. ラバーダック §M3 との整合確認 — Q3=A 尊重しつつ U01 内部 6 区画 / U08 内部 5+1 ファイル分割で並行作業可能性を補填 (`unit-of-work.md` §5)
- [x] B9. 完了チェック — `aidlc-docs/aidlc-state.md` を更新し、ユーザー承認受領 (2026-05-10T07:50:00Z)

---

## §C 生成しない方針 (Out of Scope)

- **詳細な Sprint Backlog / 工数見積** → Project Management 範疇 (本ステージ外)
- **個別ユニットの詳細設計** → Functional Design (per-unit, CONSTRUCTION) で扱う
- **ユニット内部のクラス設計** → Code Generation で扱う
- **CI/CD パイプライン詳細** → Infrastructure Design / Build and Test で扱う
