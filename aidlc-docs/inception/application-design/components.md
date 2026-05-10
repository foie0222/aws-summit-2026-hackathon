# Components — noshi

Application Design レベルのコンポーネント識別。詳細メソッドは `component-methods.md`、サービス境界は `services.md`、依存関係は `component-dependency.md` を参照。

---

## 設計判断テーブル (回答ベース)

| 判断項目 | 採用 | 理由 |
|---|---|---|
| フロント配信 (Q1) | **Next.js (SSR + API ルート)** | フル機能 / Amplify 親和 / SSE ストリーミング容易 |
| AgentCore 活用度 (Q2) | **LLM 判断のみ AgentCore** (旧「全活用」を §C4 で再定義) | 純粋計算は L-RULES 直呼び。AgentCore は画像理解 / 文面生成 / 候補選定 / 関係性推定 / 推奨額判定 のみ |
| 文化ルール実装 (Q3) | **ハイブリッド (コード + LLM)** | 計算は決定的純粋関数 (PBT 適用)、文面/微妙判断は LLM |
| OCR (Q4) | **Bedrock マルチモーダル ワンショット** | レイテンシ単純化、運用コスト最小 |
| 認証 (Q5) | **Cognito + マジックリンク** | パスワード管理不要、ハッカソン映え |
| Person ledger DB (Q6) | **DynamoDB Single-table** | 1 クエリでタイムライン取得、コスト効率最良 |
| 発注 / 礼状 (Q7) | **SES (礼状) + EC サンドボックス (発注ダミー)** | 動いた感を最大化 |
| モノレポ + IaC (Q8) | **pnpm workspaces + AWS CDK (TypeScript)** | 言語統一、型共有 |

---

## コンポーネント一覧 (10 件)

### フロントエンド

#### C-WEB: Web Frontend
- **責務**: 全ユーザー操作の入口。Next.js (App Router) で SSR + API ルート。Cognito ログインフロー、画像アップロード UI、Person タイムライン、お返し提案画面、礼状エディタ、Analytics ダッシュボード、デモアカウント自動ログインなど全 UI を提供。
- **インターフェース**:
  - 公開: ブラウザ HTTP(S) アクセス (Amplify Hosting)
  - サーバ間: API Gateway 経由でバックエンド呼び出し / Cognito SDK / SSE ストリーミング受信
- **対応 Story**: 全 US-XX が UI 経由で C-WEB に依存

### バックエンドサービス (Lambda 群)

#### C-AUTH: Authentication & Session
- **責務**: Cognito User Pool 統合、マジックリンク発行 / 検証 (Cognito Custom Auth Flow + Lambda triggers)、デモアカウント自動ログイントークン発行、セッション管理。
- **インターフェース**:
  - 入力: `/auth/magic-link/request`, `/auth/magic-link/verify`, `/auth/demo/start`
  - 出力: ID token / Access token (Cognito 標準)
- **対応 Story**: US-01 (登録), US-21 (デモ自動ログイン)

#### C-PERSON: Person Ledger Service
- **責務**: Person (人物) CRUD、Receive/Give レコードの永続化、Person 詳細統合タイムライン読み出し、お返し済 / 未対応の自動判定 (FR-L5)、Single-table DynamoDB アクセス層。
- **インターフェース**:
  - REST: `/persons` (CRUD), `/persons/{id}/timeline`, `/persons/{id}/pending-returns`
- **対応 Story**: US-03, US-09 (Give 自動連携書込), US-10, US-11, US-12, US-13, US-21 (デモ data 投入)

#### C-RECEIVE: Receive Workflow Service
- **責務**: 受領レコードのライフサイクル管理 (画像受け取り → AgentCore 起動 → 抽出結果取得 → 提案受領 → 承認 → 内祝い登録)。S3 プリサインド URL 発行、AgentCore 呼び出しのオーケストレーション。
- **インターフェース**:
  - REST: `/receives/upload-url` (presigned), `/receives/{id}/extract` (起動), `/receives/{id}/suggest`, `/receives/{id}/letter`, `/receives/{id}/approve` (発送承認 → C-PERSON + C-DELIVERY)
  - SSE: `/receives/{id}/stream` (礼状文面ストリーミング)
- **対応 Story**: US-04, US-05, US-06, US-07, US-08, US-09 (オーケストレーション主導), US-22 (デモ S1 主役)

#### C-GIVE: Give Workflow Service
- **責務**: 手動贈与記録 (お年玉 / 香典 / お祝い)、年齢別相場 / 親族間バランス推奨 (AgentCore 経由 → L-RULES)、内祝いから元 Receive への遡りリンク提供、リマインドジョブの仕込み (EventBridge スケジュール登録)。
- **インターフェース**:
  - REST: `/gives` (CRUD), `/gives/recommendations`, `/gives/{id}/source-receive`, `/reminders` (登録/解除)
- **対応 Story**: US-12, US-13, US-14, US-15

#### C-ANALYTICS: Analytics Service
- **責務**: 累計総額集計 (人別 / 年別 / 用途別)、親族 ROI ヒートマップ、あげすぎ/少なすぎ警告、贈与税 110 万円枠通知。L-RULES の純粋関数を使った警告判定。
- **インターフェース**:
  - REST: `/analytics/totals`, `/analytics/heatmap`, `/analytics/balance-warnings`, `/analytics/tax-thresholds`
- **対応 Story**: US-16, US-17, US-18, US-19, US-23 (デモ S2/S3 で参照)

#### C-DELIVERY: Delivery Service
- **責務**: 内祝い承認後の **礼状送信 (SES)** と **品物発注 (EC サンドボックス API)** を統括。承認イベントをトリガに 2 系統並行実行。EventBridge からのリマインドメール送信もここから。
- **インターフェース**:
  - 内部 SDK / Event:
    - `sendReturnLetter(recipient, letterText, attachmentMeta)` (SES)
    - `placeOrder(productId, recipient, deliveryAddress)` (EC サンドボックス API)
    - `sendReminderMail(userEmail, eventInfo)` (SES)
- **対応 Story**: US-09 (発送承認), US-15 (リマインド配信), US-22 (デモ S1)

### エージェントランタイム

#### C-AGENT: AgentCore Agent
- **責務**: noshi の **LLM 判断層** (画像理解 / 文面生成 / 候補選定 / 関係性推定 / 推奨額判定)。Bedrock AgentCore に登録された 1 つの Agent 定義として運用。
- **改訂 (§C4 / §M4 反映)**: 純粋計算系ツールは削除。純粋計算は orchestration Lambda が L-RULES を直接呼ぶ。
- **AgentCore 機能採用リスト** (§M4 で確定):
  | 機能 | 採用 |
  |---|---|
  | Runtime / Memory / Tools / Guardrails | ✅ |
  | Identity (Cognito で代替) / Built-in Tools (Browser/Code Interpreter 不要) / Gateway / Policy / Evaluations | ❌ |
  | Observability | 🟡 (NFR Design で判断) |
- **AgentCore 構成要素**:
  - **Memory**: 短期会話文脈・1 セッション内の中間結果 (永続データは `@noshi/person-ledger` 経由)
  - **Guardrails (Bedrock)**:
    - **Input**: PROMPT_ATTACK / DENY_TOPICS (噂話・暴力・個人攻撃)
    - **Output**: SEXUAL / HATE / VIOLENCE / MISCONDUCT mask
  - **Tools (LLM 判断系のみ)**:
    - `extract_gift_image` → Bedrock マルチモーダル ワンショット (画像 → 構造化 JSON)
    - `suggest_return_gifts` → カテゴリ × 価格帯から候補生成 (Bedrock LLM 判断)
    - `generate_letter` → Bedrock 推論 (Output Guardrails 適用、SSE ストリーミング)
    - `estimate_relationship` → LLM 判断 (履歴 + 名前ヒント参照)
    - `recommend_amount` → LLM 判断 (orchestration が L-RULES の baseRange と history を渡し、LLM が numeric を提案)
- **削除されたツール** (orchestration Lambda が L-RULES を直呼び):
  - ~~`calculate_return_range`~~ → L-RULES.halfReturnRange / thirdReturnRange
  - ~~`lookup_cultural_deadline`~~ → L-RULES.culturalDeadline
  - ~~`detect_balance_warning`~~ → L-RULES.balanceWarning
  - ~~`detect_tax_threshold`~~ → L-RULES.taxThresholdStatus
- **インターフェース**:
  - 内部: `bedrock-agentcore.InvokeAgent(agentId, sessionId, task, payload)` (C-RECEIVE / C-GIVE から呼ぶ)
  - リトライ / フォールバック: SDK 標準リトライ (exp backoff、最大 3 回)、Sonnet→Haiku フォールバック
- **対応 Story**: US-05, US-08, US-14 (推奨判断部分), US-22 (デモ S1 主役)

### 共有ライブラリ (モノレポ workspace パッケージ)

> **改訂 (§S6, §m3 反映)**: 共有ライブラリを 4 つに拡張。

#### L-RULES: Cultural Rule Library
- **責務**: 純粋関数のみで構成された日本贈答文化ルール:
  - `halfReturnRange(amount, purpose) -> {min, max}`
  - `thirdReturnRange(amount, purpose) -> {min, max}`
  - `otoshidamaSuggestion(ageGroup) -> {min, recommended, max}`
  - `culturalDeadline(purpose, baseDate) -> Date`
  - `taxThresholdStatus(yearlyTotal) -> {percentage, warningLevel}`
  - `balanceWarning(receivedTotal, givenTotal, threshold) -> warning?`
- **特徴**: TypeScript pure functions、副作用なし、決定的、PBT (Partial) 対象。
- **利用者**: C-AGENT のツール / C-ANALYTICS / C-GIVE / C-RECEIVE
- **対応 Story**: US-06, US-14, US-18, US-19 の計算部分

#### L-MODELS (`@noshi/models`): Shared Domain Types
- **責務**: TypeScript 型定義 (DTO / DynamoDB レコード / API スキーマ / AgentCore tool 入出力スキーマ) + 構造化ロガー (`@noshi/logger` をサブモジュールとして同梱)。バックエンドとフロントの両方から import。
- **主要型**: `Person`, `ReceiveRecord`, `GiveRecord`, `LedgerTimelineEntry`, `ExtractedGiftInfo`, `ReturnSuggestion`, `LetterDraft`, `Warning`, `User`, etc.
- **対応 Story**: 全 Story (型は横断利用)

#### `@noshi/person-ledger`: Person Ledger Shared Package (新設)
- **責務**: DynamoDB Single-table へのアクセスロジック + 整合性ロジックを内包。**`userId` を必須引数とし型レベルで強制**。
- **主要関数**: `appendReceiveRecord(userId, ...)` / `appendGiveRecord(userId, ...)` / `linkReceiveToGive(userId, ...)` / `getTimeline(userId, ...)` / `listPendingReturns(userId)` / `deleteAllUserData(userId)` 等
- **取り込み元**: C-PERSON Lambda が API 経由で公開、加えて C-RECEIVE / C-GIVE / C-ANALYTICS / C-AUTH (Cleaner) が同パッケージを直接取り込み (Direct Lambda Invoke は使わない)
- **PBT 対象**: 整合性ロジック (linkReceiveToGive 後の status 計算 / お返し済み判定など) を Property テスト

#### `@noshi/auth-context`: Authorization Helper (新設)
- **責務**: Lambda 入口での認可ヘルパー
- **主要関数**: `requireUserScope(event): { userId: string; isDemoSession: boolean }`
  - JWT 検証 / userId 抽出 / スコープチェック / 失敗時 401/403 返却
- **取り込み元**: 全 Lambda の入口で必ず呼ぶ (規約 + lint rule)

---

## ストア / 外部サービス (コンポーネントではないが言及)

| 種別 | 名称 | 用途 |
|---|---|---|
| AWS マネージド | DynamoDB | Person ledger Single-table |
| AWS マネージド | S3 | 画像ストア (TTL ライフサイクル) |
| AWS マネージド | Cognito User Pool | C-AUTH のバックエンド |
| AWS マネージド | API Gateway | バックエンド Lambda 群の入口 |
| AWS マネージド | Lambda | C-AUTH / C-PERSON / C-RECEIVE / C-GIVE / C-ANALYTICS / C-DELIVERY のホスト |
| AWS マネージド | Bedrock + AgentCore | C-AGENT のホスト |
| AWS マネージド | Bedrock Guardrails | C-AGENT 内で参照 |
| AWS マネージド | SES | C-DELIVERY (礼状 + リマインド配信) |
| AWS マネージド | EventBridge | C-GIVE / C-DELIVERY (リマインドスケジュール) |
| AWS マネージド | Secrets Manager / SSM | NFR-1 シークレット管理 |
| AWS マネージド | CloudWatch | 観測性 |
| 自前 mock | **ec-sandbox-service** (Lambda Function URL) | C-DELIVERY が叩く品物発注 mock。`POST /orders` → `{orderId, status, expectedDelivery}`。3% で `rejected` を返してリトライ動作の確認に使う (§M1 で確定: 公開 EC sandbox は事実上ないため自前 mock として実装) |
