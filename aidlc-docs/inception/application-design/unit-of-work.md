# Units of Work — noshi

**ステージ**: INCEPTION / Units Generation
**作成日**: 2026-05-10
**ユニット総数**: **11**
**チーム規模**: 2〜4 名 (推奨 4 名)

---

## 1. 採用方針 (回答ベース)

| 判断 | 採用 | 補足 |
|---|---|---|
| Q1 ユニット数 | **10〜12 ユニット (細粒、コンポーネント別 1:1)** | 担当 4 名なら 1 人 2〜3 ユニット |
| Q2 グルーピング | **ドメイン別 (Q1 の細粒 = サービス境界 = ドメイン境界として扱う)** | User Stories の Epic と整合 |
| Q3 サブユニット | **分けない** | ラバーダック §M3 の懸念は "ユニット内部のモジュール並行作業ガイド" で吸収 (§5) |
| Q4 Shared | **1 つの `shared` ユニット (U10)** | API freeze の責任が集中、ブロッカー解消が速い |
| Q5 IaC | **独立 `infra` ユニット (U11)** | 集中管理、CDK 整合性を 1 名がガード |
| Q6 レイアウト | **役割別** `web/` + `services/` + `agent/` + `libs/` + `infra/` | §3 ディレクトリ構造で明示 |
| Q7 並行戦略 | **Sprint 0 で contract freeze + Sprint 1 から全並行** | Sprint 0 のスコープを §6 で明示 |

---

## 2. ユニット一覧 (11 ユニット)

| Unit ID | Name | Domain | 主要コンポーネント | デプロイ単位 | 主担当目安 |
|---|---|---|---|---|---|
| **U01** | frontend | Frontend | C-WEB (Next.js) | Amplify Hosting | 1 名 |
| **U02** | identity | Identity | C-AUTH (+ Cognito triggers) | Lambda + Cognito | 共有 |
| **U03** | person-ledger | Person | C-PERSON | Lambda + DynamoDB | 1 名 |
| **U04** | receive | Receive | C-RECEIVE | Lambda + S3 + Function URL | 1 名 |
| **U05** | give | Give | C-GIVE | Lambda + EventBridge | 共有 |
| **U06** | analytics | Analytics | C-ANALYTICS | Lambda | 共有 |
| **U07** | delivery | Delivery | C-DELIVERY | Lambda + SES | 共有 |
| **U08** | agent | Agent (AI) | C-AGENT | Bedrock AgentCore + Tool Lambdas | 1 名 |
| **U09** | ec-sandbox | Mock | ec-sandbox-service | Lambda Function URL (mock) | 共有 |
| **U10** | shared | Shared Libs | L-RULES / L-MODELS / `@noshi/person-ledger` / `@noshi/auth-context` / `@noshi/logger` | npm packages (workspace) | 1 名 (リード) |
| **U11** | infra | Infrastructure | AWS CDK スタック群 | CDK Pipeline | 1 名 (リード) |

> **担当配分例 (4 名チーム想定)**:
> - **担当 A (リード)**: U10 shared + U11 infra (Sprint 0 主導)
> - **担当 B (バックエンド)**: U03 person-ledger + U02 identity + U07 delivery + U09 ec-sandbox
> - **担当 C (バックエンド + AI)**: U04 receive + U08 agent
> - **担当 D (フロントエンド + 機能横断)**: U01 frontend + U05 give + U06 analytics

---

## 3. モノレポディレクトリ構造 (Greenfield 必須)

> **Q6=C 採用**: 役割別レイアウト。pnpm workspaces。

```
noshi/
├── package.json                     # pnpm workspace root
├── pnpm-workspace.yaml
├── tsconfig.base.json
├── .editorconfig
├── .eslintrc.cjs
├── .github/                         # CI/CD (Sprint 0 で立てる)
│   └── workflows/
│
├── web/                             # U01 frontend (Next.js)
│   ├── app/                         # App Router pages
│   │   ├── (auth)/
│   │   ├── (app)/
│   │   ├── demo/
│   │   └── api/                     # BFF API routes
│   ├── components/
│   ├── package.json                 # @noshi/web
│   └── next.config.mjs
│
├── services/                        # U02〜U07, U09 backend services
│   ├── auth-service/                # U02 identity (Cognito triggers + magic link + demo lease)
│   ├── person-ledger-service/       # U03 person-ledger (Lambda 群)
│   ├── receive-service/             # U04 receive (Lambda 群 + Function URL letter stream)
│   ├── give-service/                # U05 give (Lambda 群 + ReminderDispatcher)
│   ├── analytics-service/           # U06 analytics (Lambda 群)
│   ├── delivery-service/            # U07 delivery (SES + EC mock 呼び出し)
│   └── ec-sandbox-service/          # U09 ec-sandbox (mock EC API)
│
├── agent/                           # U08 agent
│   └── noshi-agent/
│       ├── definition/              # AgentCore Agent 定義 (prompt, tools schema)
│       ├── tools/                   # Tool 実装 Lambda 群 (extract / suggest / letter / estimate / recommend)
│       └── package.json
│
├── libs/                            # U10 shared
│   ├── models/                      # @noshi/models (型定義 + ロガー)
│   │   └── src/
│   │       ├── domain.ts            # Person / ReceiveRecord / GiveRecord / ...
│   │       ├── api-schema.ts        # API 入出力スキーマ
│   │       ├── agent-tool-schema.ts # AgentCore tool 入出力スキーマ
│   │       └── logger.ts            # 構造化 JSON ロガー
│   ├── rules/                       # L-RULES (純粋関数)
│   │   └── src/
│   │       ├── return-range.ts      # halfReturnRange / thirdReturnRange
│   │       ├── otoshidama.ts        # otoshidamaSuggestion
│   │       ├── deadline.ts          # culturalDeadline
│   │       ├── tax.ts               # taxThresholdStatus
│   │       └── balance.ts           # balanceWarning
│   ├── person-ledger/               # @noshi/person-ledger (DynamoDB 共有アクセス)
│   │   └── src/
│   │       ├── client.ts            # DynamoDB クライアント
│   │       ├── repository.ts        # appendReceive / appendGive / linkRG / getTimeline / ...
│   │       └── invariants.ts        # 整合性ロジック (PBT 対象)
│   ├── auth-context/                # @noshi/auth-context (Lambda 認可ヘルパー)
│   │   └── src/
│   │       └── require-user-scope.ts
│   └── logger/                      # @noshi/logger (薄いラッパー、再エクスポート)
│       └── src/
│           └── index.ts
│
└── infra/                           # U11 infrastructure (CDK)
    ├── bin/
    │   └── noshi.ts                 # CDK app entry
    ├── lib/
    │   ├── stacks/
    │   │   ├── network-stack.ts
    │   │   ├── identity-stack.ts    # Cognito User Pool + magic link Triggers
    │   │   ├── api-stack.ts         # API Gateway + Lambda Authorizer
    │   │   ├── storage-stack.ts     # DynamoDB single-table + S3 buckets
    │   │   ├── compute-stack.ts     # Lambda functions (各 service)
    │   │   ├── agent-stack.ts       # AgentCore Agent + Tools
    │   │   ├── delivery-stack.ts    # SES + EventBridge cron
    │   │   ├── frontend-stack.ts    # Amplify Hosting / Route53 / CDN
    │   │   └── observability-stack.ts
    │   └── constructs/              # 共通 Construct
    ├── cdk.json
    └── package.json
```

### 3.1 監査用パッケージ命名規約

| ディレクトリ | npm package 名 |
|---|---|
| `web/` | `@noshi/web` |
| `services/<name>/` | `@noshi/service-<name>` |
| `agent/noshi-agent/` | `@noshi/agent` |
| `libs/<name>/` | `@noshi/<name>` |
| `infra/` | `@noshi/infra` |

---

## 4. ユニット詳細

### U01: frontend
- **責務**: ユーザーが触る全 UI (認証 / Receive / Give / Analytics / 設定 / デモ)。Next.js App Router で SSR + API ルート。礼状ストリーミングは Function URL を fetch with `text/event-stream` で受信。
- **コンポーネント**: C-WEB
- **Story (主)**: 全 23 Story の UI 部分 (UI レイヤとしては全 Story が U01 を経由)
- **依存**: U10 (型 / ロガー / auth-context は edge / API ルートで利用)、U11 (Amplify Hosting / Route53 / CDN)
- **モジュール内分割案 (M3 補填)**:
  - `web/app/(auth)/` — US-01, US-21
  - `web/app/(app)/receive/` — US-04〜US-09, US-22 (重め、ペアプロ推奨)
  - `web/app/(app)/persons/` — US-03, US-10, US-11
  - `web/app/(app)/give/` — US-12〜US-15
  - `web/app/(app)/analytics/` — US-16〜US-19, US-23
  - `web/app/(app)/settings/` — US-02, US-20
  - 上記 7 区画を 2 名でローテーションする運用が現実的

### U02: identity
- **責務**: 認証 (Cognito Custom Auth マジックリンク + Hosted UI Plan B) / デモアカウント Lease 取得 / アカウント削除のエントリ。
- **コンポーネント**: C-AUTH (Lambda + Cognito Triggers `DefineAuthChallenge` / `CreateAuthChallenge` / `VerifyAuthChallenge`)
- **Story**: US-01, US-20 (削除トリガ)、US-21 (デモログイン)
- **依存**: U10 (auth-context / models)、U11 (Cognito UserPool)、U07 (デモ Cleaner 用 SQS), U03 (Cleaner からのデータ削除呼出 SDK)

### U03: person-ledger
- **責務**: Person CRUD / Receive・Give 永続化 / 統合タイムライン / お返し済 / 未対応自動判定 / アカウントデータ全削除。`@noshi/person-ledger` の薄い API エンドポイント + 直接呼び出し用パッケージ。
- **コンポーネント**: C-PERSON
- **Story**: US-02, US-03, US-09 (書込み主役), US-10, US-11, US-12 (書込)、US-13 (lookup)、US-20 (deleteAllUserData)、US-21 (preset データ投入)
- **依存**: U10 (`@noshi/person-ledger` / models / rules)、U11 (DynamoDB Single-table)

### U04: receive
- **責務**: 画像取り込み → AgentCore extract → L-RULES 計算 → AgentCore suggest → 礼状ストリーム → 承認 → Person ledger / Delivery 連携。
- **コンポーネント**: C-RECEIVE (REST API + Lambda Function URL 両系統)
- **Story**: US-04, US-05, US-06, US-07, US-08, US-09 (orchestration 主導), US-22 (デモ S1 主役)
- **依存**: U10 (rules / person-ledger / auth-context)、U08 (agent)、U03 (書込み呼び出し)、U07 (delivery)、U11 (API GW + Function URL + S3)

### U05: give
- **責務**: 手動贈与 CRUD / 推奨額 (L-RULES + AgentCore) / 内祝い遡り / リマインド登録 + ReminderDispatcher cron。
- **コンポーネント**: C-GIVE (Lambda 群)
- **Story**: US-12, US-13, US-14, US-15
- **依存**: U10、U03 (記録読書)、U08 (agent recommend)、U07 (リマインドメール)、U11 (EventBridge cron rule)

### U06: analytics
- **責務**: 累計 / ROI ヒートマップ / バランス警告 / 110 万枠通知。Person ledger を読み、L-RULES を直呼びで集計。β 版は全件読みで割切。
- **コンポーネント**: C-ANALYTICS (Lambda 群)
- **Story**: US-16, US-17, US-18, US-19, US-23 (デモ S2/S3 余力)
- **依存**: U10、U03、U11

### U07: delivery
- **責務**: 礼状 SES 送信 / リマインドメール送信 / 品物発注 (ec-sandbox 呼出) / 失敗時の SQS DLQ。
- **コンポーネント**: C-DELIVERY
- **Story**: US-09 (発送 part)、US-15 (リマインド配信)、US-22 (デモ S1)
- **依存**: U10、U09 (ec-sandbox)、U11 (SES / SQS)

### U08: agent
- **責務**: AgentCore Agent 定義 (prompt / memory / guardrails) と Tool 実装 5 種 (extract_gift_image / suggest_return_gifts / generate_letter / estimate_relationship / recommend_amount)。
- **コンポーネント**: C-AGENT
- **Story (cross-cutting)**: US-05, US-06 (suggest_products LLM 部分のみ), US-08, US-14 (judge_amount LLM 部分), US-22
- **依存**: U10 (models)、U11 (Bedrock AgentCore + Bedrock Guardrails 設定)
- **モジュール内分割案**:
  - `agent/noshi-agent/definition/` (Agent prompt / config / Memory / Guardrails)
  - `agent/noshi-agent/tools/extract.ts` (画像理解)
  - `agent/noshi-agent/tools/suggest.ts` (候補生成)
  - `agent/noshi-agent/tools/letter.ts` (文面 + ストリーミング)
  - `agent/noshi-agent/tools/estimate.ts` (関係性)
  - `agent/noshi-agent/tools/recommend.ts` (推奨額)

### U09: ec-sandbox
- **責務**: 自前 mock EC API。`POST /orders` で `{orderId, status, expectedDelivery}` を返却。3% で `rejected` を返してリトライ動作の確認材料を提供。
- **コンポーネント**: ec-sandbox-service (Lambda Function URL)
- **Story (cross-cutting)**: US-09, US-22
- **依存**: U10 (logger 程度)、U11 (Function URL)

### U10: shared
- **責務**: 全ユニットが取り込む共有パッケージ群の所有 + Sprint 0 の **API freeze** をリードする。
  - `@noshi/models` (型 / API スキーマ / ロガー)
  - `L-RULES` (純粋関数、PBT)
  - `@noshi/person-ledger` (DynamoDB アクセス + 整合性ロジック)
  - `@noshi/auth-context` (認可ヘルパー)
  - `@noshi/logger`
- **依存**: なし (他全ユニットが U10 に依存)
- **PBT 対象**: rules / person-ledger.invariants

### U11: infra
- **責務**: AWS CDK スタック全部。Cognito / API Gateway / Function URL / DynamoDB / S3 / Bedrock / AgentCore / SES / EventBridge / Amplify Hosting / Route53 / ACM / SQS。CI/CD パイプライン (GitHub Actions)。
- **依存**: なし (他ユニットの Lambda 関数 / Asset を取り込んでデプロイ)
- **特殊責務**:
  - SES Production Access 申請の起票・追跡
  - Bedrock Model Access リクエストの起票・追跡
  - AgentCore Agent 設定の CDK Construct 化

---

## 5. M3 (C-WEB / C-AGENT 並行作業) との整合確認

| 懸念 (ラバーダック §M3) | 本ユニット計画での対応 |
|---|---|
| C-WEB が 1 ユニット = 1 名担当だとボトルネック | U01 内部を **6 区画 (auth/receive/persons/give/analytics/settings)** に分割し、2 名でローテーション可能 (§4 U01 モジュール内分割案) |
| C-AGENT が 1 ユニット = 1 名担当だとボトルネック | U08 内部を **definition + 5 tool ファイル** に分割し、Sprint 0 で tool スキーマ freeze 後は並行実装可能 (§4 U08 モジュール内分割案) |

> Q3=A の選択 (分けない) を尊重しつつ、内部モジュール分割で並行作業可能性を確保。

---

## 6. Sprint 0: Contract Freeze スコープ

> **Q7=C 採用**。Sprint 0 (3〜5 日) で以下の contract を **U10 shared が主導** して確定。Sprint 1 開始時にこれらは "freeze" され、変更には全員レビューを要する。

### 6.1 Freeze 対象 (Sprint 0 内に確定)

| カテゴリ | 内容 | 場所 |
|---|---|---|
| **ドメイン型** | Person / ReceiveRecord / GiveRecord / Purpose / GivePurpose / AgeGroup / LedgerTimelineEntry / ExtractedGiftInfo / ReturnSuggestion / LetterDraft / Warning / TaxAlert / Reminder | `libs/models/src/domain.ts` |
| **API スキーマ** | 各 Lambda の Request / Response (Zod or io-ts スキーマ) | `libs/models/src/api-schema.ts` |
| **AgentCore Tool スキーマ** | 5 ツールの入出力 | `libs/models/src/agent-tool-schema.ts` |
| **DynamoDB key 設計** | PK / SK / GSI のパターン (Person / Receive / Give / Reminder / Lease / DemoPrecomputed) | `libs/person-ledger/src/keys.ts` |
| **L-RULES シグネチャ** | 6 関数の型と PBT 不変条件 | `libs/rules/src/*.ts` |
| **`requireUserScope` API** | 認可ヘルパーの戻り型 | `libs/auth-context/src/require-user-scope.ts` |
| **共通エラー / Problem Details 形** | `application/problem+json` の標準形 | `libs/models/src/errors.ts` |

### 6.2 Sprint 0 Definition of Done

- [ ] U10 の全 stub 実装が `pnpm build` 通る (関数本体は throw でも可)
- [ ] U11 で Cognito UserPool / DynamoDB Single-table / API Gateway / Bedrock 接続まで CDK で立つ
- [ ] U02 〜 U09 の各サービスで「最低 1 つの Lambda が 200 OK を返す」レベルの skeleton が動く
- [ ] U01 のホーム画面が U10 の型を使ってビルドできる
- [ ] **デモアカウント Pool が 10 個事前作成される (CDK + Custom Resource)**

### 6.3 Sprint 1 以降

- 全ユニットが並行で機能実装に入る
- `unit-of-work-dependency.md` の Critical Path に沿う

---

## 7. ユニット粒度検証

| 検証項目 | 結果 |
|---|---|
| 全 23 Story がいずれかのユニットに割当 | ✅ (`unit-of-work-story-map.md` で確認) |
| 各ユニットが 1 つのデプロイ単位に対応 | ✅ (一部 cross-cutting あり、表 §2 に記載) |
| ユニット間の循環依存なし | ✅ (§unit-of-work-dependency.md DAG 検証) |
| 担当 4 名で 11 ユニットを分担可能 | ✅ (§2 担当配分案、1 人 2〜3 ユニット) |
| 共有パッケージのオーナーシップが明確 | ✅ (U10 リード集中) |
| Greenfield コード組織が定義済 | ✅ (§3) |
| Sprint 0 contract freeze スコープが明確 | ✅ (§6) |

> **粒度に関する自己批評**: 11 ユニットは 4 名チームには細粒寄り。Q1=A の選択を尊重したが、運用上は **「1 人 2〜3 ユニット」を前提** にする必要がある。これは単なる擬似担当の問題で、ハッカソン期間でこなせる現実的なボリューム。
