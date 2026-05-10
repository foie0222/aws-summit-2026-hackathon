# Design Revisions — Rubber Duck Review Resolutions

ラバーダックレビュー (2026-05-10) で抽出した 15 件以上の指摘とその解決を記録する。
本ドキュメントは **設計判断の差分台帳** であり、最新状態は各個別ファイル (`components.md` 等) と `application-design.md` を参照する。

---

## 解決ステータス凡例

- ✅ 設計に反映済 (本日適用)
- 🟡 ユーザー判断待ち / Plan B あり
- ⏭ 後続ステージ (Functional / NFR / Infrastructure / Code Generation) へ持ち越し
- 📋 準備タスクとして登録 (実作業項目)

---

## 🔴 Critical 級 (4 件)

### C1. SSE via API Gateway は不可 → Lambda Function URL に切替
**指摘**: API Gateway REST/HTTP は SSE 非対応。礼状ストリーミング (US-08) が動かない。

**解決**: ✅
- `C-RECEIVE.generateLetter` だけ **Lambda Function URL + Response Streaming** で公開
- 他の REST エンドポイントは API Gateway のまま (Cognito JWT Authorizer は API Gateway 側で完結)
- Function URL 側の認証は **AWS IAM (SigV4)** または **CUSTOM (Lambda Authorizer)** で別途実装
  - Cognito JWT を `Authorization: Bearer` で受け取り、Function URL に紐付く Lambda 内で検証する pattern を採用
- `C-WEB` は両方のエンドポイントを叩く (REST と Function URL)

**反映先**: `services.md` §横断 / `component-dependency.md` §通信プロトコル / `application-design.md` §5

---

### C2. デモアカウント分離戦略 → **Pool 方式 (A 案) を採用**
**指摘**: US-21 受け入れ基準の「他来場者から見えない」と「事前作成済デモアカウント」が両立してない。

**解決**: ✅ **A 案: デモアカウントプール** を採用
- **Cognito User Pool に専用グループ `noshi-demo-pool`** を作成、その配下に **N 個 (初期値 10)** のデモアカウントを事前作成 (`demo-001@noshi.example` 〜 `demo-010@noshi.example`)
- 各アカウントはペルソナ事前データ (Person / Receive / Give) を投入済み
- ブースデバイスごとに Pool からアカウントを **取得 / 占有 (Lease)**
- Lease 取得は DynamoDB `LeaseLock` テーブルで楽観ロック実装 (deviceId, leaseExpiresAt)
- **TTL 30 分** か **明示的な「次の来場者へ」ボタン** で Lease 解放 → reset Lambda がそのアカウントの user 配下データを削除 → 初期ペルソナデータ再投入 → Lease 解放完了
- Pool 全空のとき: フロントが「デモが満員です。少々お待ちください」と表示 (β 版で現実的にあり得るほど来場が殺到する想定は薄い)

**プール初期サイズの根拠**: ブース同時稼働デバイス 5〜10 台想定 + リセット中の予備 = 10 で十分。

**他案の理由**:
- B 案 (オンザフライ作成): Cognito ユーザー作成 + 削除のリードタイム / orphan リスク高 → 却下
- C 案 (シングル共有 + 揮発化): Receive↔Give 連携を DynamoDB に書く設計と矛盾 → 却下

**反映先**: `application-design.md` §12 / `components.md` C-AUTH 責務 / `services.md` O8

---

### C3. AgentCore のリージョン可用性 → ap-northeast-1 で全機能利用可能
**指摘**: AgentCore が Tokyo で動くか不明、PII 越境リスク。

**解決**: ✅ **AWS 公式ドキュメントで確認済 (2026-05-10 時点)**

| AgentCore 機能 | ap-northeast-1 (Tokyo) |
|---|---|
| Runtime | ✅ |
| Memory | ✅ |
| Gateway | ✅ |
| Identity | ✅ |
| Built-in Tools | ✅ |
| Observability | ✅ |
| Policy | ✅ |
| Evaluations | ✅ |
| Agent Registry | ✅ |
| harness (preview) | ❌ |
| payments (preview) | ❌ |

→ **本プロジェクトに必要な全機能が Tokyo で利用可能**。PII 越境問題は発生しない。

**ソース**: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agentcore-regions.html

**反映先**: `application-design.md` §11

---

### C4. AgentCore "全活用" → **「LLM 判断のみ AgentCore」** に再定義
**指摘**: 純粋計算 (半返し計算等) を AgentCore 経由で行うのはレイテンシと課金の無駄。

**解決**: ✅ アーキテクチャ原則を入れ替え。**Pure Rules First (旧 #2) を最上位 / AgentCore は LLM 判断のみ** に変更。

**運用ルール**:
- **L-RULES の純粋関数を呼ぶだけのケース** → orchestration Lambda (C-RECEIVE / C-GIVE / C-ANALYTICS) が **直接 L-RULES** を呼ぶ。AgentCore を経由しない。
- **LLM 判断 (画像理解 / 礼状文面 / 候補選定 / 関係性推定 / 微妙な額判定)** → AgentCore を経由。
- 結果として、AgentCore の Tools 数が縮小:
  - **削除**: `calculate_return_range`, `lookup_cultural_deadline`, `detect_balance_warning`, `detect_tax_threshold` (これらは Lambda 内で L-RULES 直呼び)
  - **残留 / 拡張**: `extract_gift_image` / `suggest_return_gifts` / `generate_letter` / `estimate_relationship` / `recommend_amount` (LLM 判断あり)

**反映先**: `application-design.md` §1, §5 / `component-methods.md` C-AGENT セクション / `services.md` O1, O2

---

## 🟠 Significant 級 (8 件)

### S1. デモ S1 の 3 分制約 → デモパス専用キャッシュ + Provisioned Concurrency
**指摘**: バックエンド時間 12〜32 秒 + ユーザー操作時間で 3 分超過リスク。

**解決**: ✅ デモパス専用最適化を設計に明記。
- **サンプル画像の extraction 結果を事前計算** → DynamoDB `DemoPrecomputed` テーブルに格納 (`extract_gift_image` の結果として返す代替パス)
- **礼状文面のテンプレ化** → ペルソナ × 用途で 5〜10 種のテンプレを事前準備、LLM 呼び出しを skip (デモパスのみ)
- **Provisioned Concurrency**: デモ用 Lambda 群 (`receive`, `agent`, `delivery` など) に最低 1 並列を予約
- **3 分タイマーのフロント表示** → ユーザー側に体験尺の透明性

**反映先**: `application-design.md` §13 / 後続 Functional Design でデモパス分岐を実装

⏭ Functional Design で分岐ロジック詳細化

---

### S2. SES Production Access → 準備タスクとして即起票
**指摘**: SES sandbox では verified 宛先のみ。本番来場者へのメール (礼状 / マジックリンク) が出せない。

**解決**: 📋 **Pre-launch Preparation Tasks** にトップ優先で登録。
- **担当**: 開発リード (本人)
- **期限**: Application Design 承認直後 (今日 / 明日)
- **タスク**:
  1. SES Production Access 申請 (用途: noshi 礼状 + リマインド + マジックリンク)
  2. ドメイン認証 (DKIM / SPF / DMARC)
  3. バウンス / 苦情処理 Lambda の準備 (ハッカソン期間中の運用負荷低減)

**反映先**: `application-design.md` §13

---

### S3. EventBridge ルール上限 → **EventBridge Scheduler** に変更
**指摘**: ユーザー単位 Rule 作成だと上限 (300/region) を超える。

**解決**: ✅ **EventBridge Scheduler** (1 アカウントあたり 100 万スケジュール、ユーザー単位 Schedule 作成可能) に変更。
- 代替案: 1 つの cron (毎日朝) + Lambda が DynamoDB を query (`reminderDueDate <= today`) → SES 送信
- ハッカソン規模 (来場者 + 永続ユーザー数百名想定) では Scheduler が過剰、後者で十分との判断もあり
- **β 版採用**: **後者 (cron + DynamoDB query)** をデフォルト、Scheduler は将来オプション

**反映先**: `services.md` O9 / `component-dependency.md` 通信パターン

---

### S4. DynamoDB aggregation コスト → 事前集計 + Streams 増分更新
**指摘**: ROI / 110 万 / バランス警告は user 配下全件読みが必要、コスト膨張。

**解決**: ✅ **事前集計キャッシュ パターン** を設計に明記。
- DynamoDB Single-table 内に集計用 SK を持つ:
  - `SK = ANALYTICS#<year>#TOTALS` → 用途別累計
  - `SK = ANALYTICS#<year>#PERSON#<personId>` → Person × 年累計
  - `SK = ANALYTICS#<year>#TAX#<personId>` → 110 万枠累計
- 書き込み (US-09 / US-12) 時に **DynamoDB Streams → Aggregator Lambda** で集計テーブルを更新
- 読み出し (US-16〜US-19) は集計 SK を直接 Get
- **β 版簡易版**: ハッカソン期間は「全件読み + メモリ集計」で動かし、Streams 増分更新は β-stretch / 将来課題

**反映先**: `application-design.md` §10 / NFR Design で performance threshold 確認

⏭ NFR Design で読み込みコスト試算 / 確定

---

### S5. Cognito Custom Auth (Magic Link) 実装複雑度 → 詳細手順を明示 + Plan B
**指摘**: 3 つの Lambda Trigger 必要、リプレイ対策など。

**解決**: ✅ **詳細手順を documents-revisions に記載 + Plan B 設定**。

**詳細実装手順 (Plan A)**:
1. Cognito User Pool で **Custom Auth Flow** を有効化
2. Lambda Trigger 3 種を実装:
   - `DefineAuthChallenge`: 1 ターン目 → CUSTOM_CHALLENGE / 2 ターン目以降 → 結果に応じて成功/失敗
   - `CreateAuthChallenge`: nonce 生成 (UUID v4) → DynamoDB `MagicLinkTokens` に `{token, email, expiresAt(15min)}` を記録 → SES でリンク送信 (`https://noshi.example/auth/callback?token=<>`)
   - `VerifyAuthChallenge`: token を DynamoDB 参照 → expiresAt チェック → 一致なら成功 + token を削除 (使い切り)
3. Refresh Token Rotation を有効化 (Cognito 設定)

**Plan B (時間切れ時)**:
- Cognito **Hosted UI** + メール+パスワード (Q5=A) に切替
- メリット: 実装時間 1 日以下に短縮
- デメリット: ハッカソン的見栄えは下がる
- **判断基準**: Sprint 1 終了時点で Plan A の Lambda Trigger 3 種が動かなければ B に切替

**反映先**: `services.md` O7 / `application-design.md` §13 / Plan B は Functional Design で実装条件を明示

---

### S6. C-PERSON 内部呼び出し → **同梱 TS モジュール (主) + クロス Lambda は HTTP API 経由**
**指摘**: SDK module / Direct invoke / 共通テーブル直接アクセスのどれか未決定。

**解決**: ✅ ハイブリッド方針を確定。
- **データアクセスロジック (DynamoDB クエリ + 整合性ロジック) を `@noshi/person-ledger` という workspace パッケージ** にする
- C-PERSON 専用 Lambda + 他 Lambda (C-RECEIVE / C-GIVE / C-ANALYTICS) **両方が同パッケージを取り込む**
- C-WEB からのリクエストは API Gateway → C-PERSON Lambda 経由 (REST API)
- 他 Lambda 内部からは パッケージ関数 (`personLedger.appendReceiveRecord(...)`) を直接呼ぶ — Direct Lambda Invoke は使わない
- **認可**: パッケージ関数の入口で `userId` を必須引数にし、**呼び出し元で必ず session の userId を渡す** (引数チェックを TypeScript の型で強制)
- 利点: 単一ロジック / 速い / 認可境界を型で守る
- 欠点: パッケージ更新時は全 Lambda 再デプロイ (本ハッカソン規模では問題なし)

**反映先**: `services.md` 横断 / `component-dependency.md` / `components.md` L-MODELS 拡張

---

### S7. L-RULES の PBT 不変条件 → **明文化**
**指摘**: 「PBT 対象」と言ったが各関数の invariant が未定義。

**解決**: ✅ 不変条件を `component-methods.md` の L-RULES セクションに追記。

| 関数 | 不変条件 (PBT properties) |
|---|---|
| `halfReturnRange(amount, purpose)` | `min ≤ max` / `min ≥ amount × 0.4` / `max ≤ amount × 0.6` (purpose 別の例外あり) |
| `thirdReturnRange(amount, purpose)` | `min ≤ max` / `min ≥ amount × 0.25` / `max ≤ amount × 0.4` |
| `otoshidamaSuggestion(ageGroup)` | `min ≤ recommended ≤ max` / `min ≥ 0` / 年齢が上がると `recommended` も単調増加 |
| `culturalDeadline(purpose, baseDate)` | `result > baseDate` / `result - baseDate ≤ 365 日` / 同 purpose で baseDate を変えても日数差は同じ (純粋関数性) |
| `taxThresholdStatus(yearlyTotal)` | `yearlyTotal` 単調 → `percentage` 単調 / 110 万円ちょうどで `warningLevel='warning'` / 0 円で `'none'` |
| `balanceWarning(received, given, threshold)` | 対称性: `received` と `given` を swap すると warning の方向だけ反転 / threshold ≥ 1 のとき必ず判定可能 |

**反映先**: `component-methods.md` L-RULES セクション

---

### S8. デモデータ規模 (TBD-S2) → **数値確定**
**指摘**: ROI ヒートマップが見栄えするには最低数値が必要、後送りすぎ。

**解決**: ✅ 確定。
- **Person**: 10 名 (続柄: 父 / 母 / 義父 / 義母 / 兄弟 1 / 姉妹 1 / おじ / おば / 親友 1 / 親友 2)
- **Receive**: 35 件 (出産祝い 8 / 結婚祝い 5 / 新築祝い 3 / 香典 2 / お年玉 (もらう側) 6 / お中元・お歳暮 (もらう) 6 / その他 5)
- **Give**: 28 件 (内祝い 12 (Receive 連携済) / お年玉 8 / 香典 3 / 結婚祝い 2 / その他 3)
- **時系列**: 過去 3 年に分散
- **Linked Receive→Give**: 12 件 (内祝いペア)

**反映先**: `application-design.md` §14 (Demo Data Specification)

---

## 🟡 Moderate 級 (5 件)

### M1. EC サンドボックス (Q7=C) の実体 → **自前 mock Lambda として実装**
**指摘**: 公開 EC サンドボックスは事実上ない、Q7=B と実質同等。

**解決**: ✅ 設計に **「自前 mock Lambda として `ec-sandbox-service`」** を明示。
- 別 Lambda + Function URL で立て、外部の EC API のように振る舞う
- `POST /orders {productId, recipient, amount}` → `{orderId, status: "accepted", expectedDelivery: <date>}` を返す
- 内部でランダムに 3% で `status: "rejected"` を返し、リトライ動作の確認に使う
- ハッカソン的見栄えは保ったまま、実装は 0.5 日

**ユーザー判断**: 🟡 もし「Q7=B 完全モック (画面表示のみ) で十分」と希望なら ec-sandbox-service Lambda は削除可。後方変更可能。

**反映先**: `components.md` 外部サービス欄 / `services.md`

---

### M2. アカウント削除 → **Soft-delete + バッチ物理削除**
**指摘**: 一括 BatchDeleteItem の整合性 / 部分失敗リスク。

**解決**: ✅ 二段階削除を採用。
- **Phase 1 (即時)**: User の `STATUS=deleted` フラグを立てる。以降 API は 410 Gone を返す。
- **Phase 2 (非同期)**: SQS (or DynamoDB Streams) → Cleaner Lambda → BatchDeleteItem ループ + S3 prefix delete
- 失敗時は SQS DLQ に積み、運用者が再実行
- 法的観点 (改正個人情報保護法 / GDPR 同等) で「30 日以内に物理削除完了」を SLA として定める (UI で説明)

**反映先**: `services.md` 横断 / `component-methods.md` C-PERSON.deleteAllUserData

---

### M3. C-WEB と C-AGENT の作業量偏り → **Sub-unit 分解の指針を Units Generation に渡す**
**指摘**: C-WEB / C-AGENT がボトルネック化、並行作業困難。

**解決**: ✅ Units Generation 段階で以下のサブユニットを推奨:

**C-WEB サブユニット案**:
- WEB-AUTH (ログイン / マジックリンクページ / デモ自動ログイン)
- WEB-RECEIVE (画像アップロード / 抽出結果 / 提案 / 礼状エディタ)
- WEB-PERSON (Person 一覧 / Person 詳細タイムライン / Person 編集)
- WEB-GIVE (Give 登録 / 推奨 / リマインド設定)
- WEB-ANALYTICS (累計 / ROI ヒートマップ / 警告)

**C-AGENT サブユニット案**:
- AGENT-CORE (AgentCore Agent 設定 / Prompt / Memory 設定 / Guardrails)
- AGENT-TOOLS-EXTRACT (画像理解ツール群)
- AGENT-TOOLS-JUDGMENT (関係性推定 / 候補生成 / 文面生成 / 推奨額)

**反映先**: Units Generation ステージへ持ち越し ⏭

---

### M4. AgentCore "全活用" 機能スコープ → **明示**
**指摘**: AgentCore のサブ機能のうち本プロジェクトで採用するものが曖昧。

**解決**: ✅ 採用機能リスト確定。

| AgentCore 機能 | 採用 | 理由 |
|---|---|---|
| Runtime | ✅ | エージェント実行基盤 |
| Memory | ✅ | 短期会話文脈 (1 セッション内) |
| Tools | ✅ | LLM 判断系のみ (C4 整理後) |
| Guardrails (Bedrock 機能) | ✅ | 礼状 / 候補出力フィルタ |
| Identity | ❌ | Cognito で管理 |
| Built-in Tools (Browser / Code Interpreter) | ❌ | 不要 |
| Gateway | ❌ | 内部 Tools 直接登録で足りる |
| Observability | 🟡 (NFR Design で判断) | CloudWatch で代替可能性 |
| Policy | ❌ | 本プロジェクト規模では過剰 |
| Evaluations | ❌ | β 版範囲外 |

**反映先**: `components.md` C-AGENT / `application-design.md` §1

---

### M5. Bedrock Guardrails の input/output 適用範囲 → **Input + Output 両方に適用**
**指摘**: input filter (プロンプトインジェクション) と output filter (不適切表現) の責務分離が曖昧。

**解決**: ✅ Guardrail 構成を 2 系統に分離。
- **Input Guardrail**: ユーザー入力 / 画像内テキストに対し:
  - PROMPT_ATTACK カテゴリで jailbreak 検出
  - DENY_TOPICS で「噂話 / 暴力的指示 / 個人攻撃」をブロック
- **Output Guardrail**: LLM 出力に対し:
  - SEXUAL / HATE / VIOLENCE / MISCONDUCT を mask
  - 礼状文面に対する敬語ガイドラインの custom topic (β-stretch)

**反映先**: `application-design.md` §10 / NFR Design

⏭ Custom topic とブロック閾値の具体値は NFR Design

---

## 🔵 Minor 級 (5 件)

### m1. Mermaid 構文 → **Code Generation 段階で Mermaid Live Editor 検証を必須タスク化**
**反映**: 📋 Pre-launch Preparation Tasks に登録 (`application-design.md` §13)

### m2. デモアカウント自動ログイン認証経路 → **Cognito AdminInitiateAuth + USER_PASSWORD_AUTH**
**解決**: ✅ Pool アカウントに固定パスワード (Secrets Manager 管理) を持たせ、Lambda が AdminInitiateAuth を呼ぶ。Function URL で `/auth/demo/start?deviceId=` を叩いて token を返す。

**反映先**: `services.md` O8 / `component-methods.md` C-AUTH.startDemoSession 補足

### m3. Lambda 認可ヘルパー → **`@noshi/auth-context` パッケージ**
**解決**: ✅ モノレポに `@noshi/auth-context` という workspace パッケージを作り、`requireUserScope(event): { userId: string }` を提供。全 Lambda の入口で必ず呼ぶ規約を README に明記。

**反映先**: `components.md` Library 群 / `services.md` 横断

### m4. ROI ヒートマップ画面の退化体感コンテキスト表示 → **明示** (TBD-L2)
**解決**: ✅ S3 デモシナリオ (US-23) の遷移先 ROI ヒートマップ画面の上部に **「これは noshi の中で、唯一あなたを"ダメに"する機能です。意図的に。」** という退化体感コンテキスト注釈を表示する。

**反映先**: `application-design.md` §10 / Functional Design で UI コピー確定

### m5. リトライ / フォールバック責務 → **AgentCore 内 (Bedrock) と orchestration Lambda 内の二段**
**解決**: ✅
- **AgentCore Tool 内** (Bedrock 呼び出し): SDK 標準のリトライ (Exponential backoff、最大 3 回)
- **orchestration Lambda 内** (AgentCore 呼び出し): モデル A 失敗 (例: Sonnet) → モデル B (Haiku) フォールバック、両方失敗時はユーザーにエラー応答
- **タイムアウト**: AgentCore call 30 秒 / Lambda 全体 60 秒

**反映先**: `services.md` 横断 / NFR Design で具体値確定

---

## 集約: 設計差分が及ぶファイル一覧

| ファイル | 主な変更 |
|---|---|
| `application-design.md` | §1 設計判断 / §5 原則順序 / §8 TBD / 新 §10 / §11 / §12 / §13 / §14 |
| `components.md` | C-AGENT 機能スコープ / C-DELIVERY EC mock 注 / Library 群に `@noshi/*` 追加 |
| `component-methods.md` | C-AGENT ツールセット縮小 / L-RULES 不変条件 / C-RECEIVE generateLetter (Function URL) |
| `services.md` | O1 簡素化 / O7 マジックリンク詳細 + Plan B / O8 デモログイン詳細 / O9 EventBridge Scheduler→cron+DDB / 横断 SSE Function URL |
| `component-dependency.md` | 通信プロトコル表 (Function URL 行 / EventBridge Scheduler) |

---

## 📋 Pre-launch Preparation Tasks (準備すべき実作業)

| # | 内容 | 期限 | 担当 |
|---|---|---|---|
| 1 | SES Production Access 申請 | Application Design 承認直後 (本日中) | リード |
| 2 | Bedrock モデル (Haiku / Sonnet 等) Tokyo リージョンでの利用可否確認・Model Access リクエスト | 本日中 | リード |
| 3 | ドメイン取得 + Route 53 設定 + 証明書 (Amplify 配信用) | Sprint 1 内 | インフラ担当 |
| 4 | DKIM / SPF / DMARC 設定 | SES Production と同時 | インフラ担当 |
| 5 | Cognito User Pool 設定 (Custom Auth + Magic Link 用) | Sprint 1 序盤 | バックエンド担当 |
| 6 | デモアカウント Pool 作成スクリプトの整備 | Sprint 2 中盤 | バックエンド担当 |
| 7 | Mermaid 図の Live Editor 検証 | Code Generation 開始前 | 全員 |

---

## ❓ ユーザー判断が残る事項

| # | 事項 | 提案デフォルト | 別案 |
|---|---|---|---|
| U-1 | C2 デモアカウント分離 | A 案 (Pool) | B (オンザフライ) / C (シングル + 揮発) |
| U-2 | M1 EC モック | 自前 mock Lambda 化 | Q7=B 相当 (画面表示のみ) に格下げ |
| U-3 | S5 認証 | Plan A (マジックリンク) を本線 / Plan B (Hosted UI) を裏 | 最初から Hosted UI に倒す |

→ **デフォルトで進めて問題なければ、上記すべて反映済みの設計で User Stories の次工程 (Units Generation) に進めます**。
