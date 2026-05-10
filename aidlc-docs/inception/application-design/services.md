# Services & Orchestration — noshi

各ユースケースを実現するサービス境界とオーケストレーションパターン。詳細フロー (シーケンス図含む) は `application-design.md` の Mermaid を参照。

---

## サービス境界 (1 サービス = 1 デプロイ単位)

| サービス | 含まれるコンポーネント | デプロイ単位 |
|---|---|---|
| **frontend-service** | C-WEB | Next.js (Amplify Hosting) |
| **auth-service** | C-AUTH | Lambda + Cognito |
| **person-ledger-service** | C-PERSON | Lambda + DynamoDB |
| **receive-service** | C-RECEIVE | Lambda + S3 + AgentCore 呼び出し |
| **give-service** | C-GIVE | Lambda + EventBridge |
| **analytics-service** | C-ANALYTICS | Lambda |
| **delivery-service** | C-DELIVERY | Lambda + SES + 外部 EC |
| **agent-service** | C-AGENT | AgentCore (マネージド) — Tools の一部は Lambda 実装でも可 |

> 共有ライブラリ (L-RULES / L-MODELS) はサービスではなく、各 Lambda パッケージにビルド時取り込み。

---

## オーケストレーションパターン

### O1: 内祝い完成までの Receive オーケストレーション (US-04〜US-09 / デモ S1)

**主導**: receive-service (C-RECEIVE)

> **改訂 (design-revisions §C4 反映)**: 純粋計算 (`halfReturnRange`, `culturalDeadline`) は AgentCore を通さず receive-service が直接 L-RULES を呼ぶ。AgentCore は LLM 判断系 (extract / suggest / letter / estimate) のみ。
> **改訂 (design-revisions §C1 反映)**: ステップ 8 の礼状ストリーミングは API Gateway ではなく Lambda Function URL 経由。
> **改訂 (design-revisions §S1 反映)**: デモパス (Pool アカウントからの呼出) では事前計算キャッシュを優先利用。

```
1. C-WEB → receive-service (REST): issueUploadUrl()
   → receive-service: S3 プリサインド URL 発行
2. C-WEB → S3: 画像直接アップロード
3. C-WEB → receive-service (REST): createReceiveDraft({ objectKeys })
4. C-WEB → receive-service (REST): extractFromImages(receiveId)
   - デモパス: DemoPrecomputed テーブルから事前計算結果を取得 → 即返却
   - 通常パス: agent-service.invokeAgent(task='extract')
       → C-AGENT.tool('extract_gift_image', s3Keys) → Bedrock マルチモーダル
       → ExtractedGiftInfo を receive-service が ReceiveDraft に保存
5. C-WEB: ユーザーが補正 → applyExtractionCorrection()
6. C-WEB → receive-service (REST): suggestReturnGifts(receiveId)
   - receive-service が直接呼ぶ:
     - L-RULES.halfReturnRange(amount, purpose)        // 純粋関数、AgentCore 経由しない
     - L-RULES.culturalDeadline(purpose, today)         // 純粋関数
   - LLM 判断のみ AgentCore へ:
     - agent-service.invokeAgent(task='suggest_products', range)
       → C-AGENT.tool('suggest_return_gifts') → Bedrock 判断
   → ReturnSuggestion + 締切日 を C-WEB へ
7. C-WEB: ユーザーが品物選定 → selectReturnGift()
8. C-WEB → receive-service (Lambda Function URL, SSE): generateLetterStream(receiveId)
   ※ API Gateway ではなく Lambda Function URL + Response Streaming
   - デモパス: 事前テンプレ文面を即ストリーム
   - 通常パス: agent-service.invokeAgent(task='letter')
       → C-AGENT.tool('generate_letter') → Bedrock + Output Guardrails
       → ストリーミングで C-WEB へ
9. C-WEB: ユーザーが文面編集 → saveLetterDraft()
10. C-WEB → receive-service (REST): approveAndShip(receiveId)
    → receive-service が `@noshi/person-ledger` を直接呼ぶ:
      a. personLedger.appendReceiveRecord (status=pending)
      b. personLedger.appendGiveRecord (origin='receive-return', sourceReceiveId)
      c. personLedger.linkReceiveToGive() → status=returned
    → delivery-service へ並行ディスパッチ:
      d. C-DELIVERY.sendReturnLetter (SES)
      e. C-DELIVERY.placeOrder (ec-sandbox-service Function URL)
    → 結果集計を C-WEB へ
11. C-WEB: 「のしを、軽やかに。関係を、ずっと。」を表示
```

### O2: 手動贈与登録と推奨 (US-12 / US-14)

**主導**: give-service (C-GIVE)

> **改訂 (§C4 反映)**: `otoshidamaSuggestion` と `balanceWarning` は give-service が直呼び。AgentCore は微妙な額判定の LLM 部分のみ担当。

```
1. C-WEB → give-service (REST): recommendNextAmount({ personId, purpose })
   - give-service が直接呼ぶ:
     - L-RULES.otoshidamaSuggestion(ageGroup)         // 年齢別相場、純粋関数
     - personLedger.getTimeline(personId)             // 過去履歴
     - L-RULES.balanceWarning(received, given, t)     // 親族間バランス、純粋関数
   - LLM 判断 (微妙な調整) のみ AgentCore へ:
     - agent-service.invokeAgent(task='judge_amount', { baseRange, history })
       → C-AGENT.tool('recommend_amount') → LLM が最終的な numeric を提案
   → AmountRecommendation を C-WEB へ
2. C-WEB → give-service: createGiveRecord(input)
   → personLedger.appendGiveRecord (パッケージ直呼び)
```

### O3: 内祝いから元 Receive 遡り (US-13)

**主導**: give-service

```
C-WEB → give-service: resolveSourceReceive(giveId)
  → C-PERSON: GiveRecord を取得 → sourceReceiveId を確認
  → C-PERSON.getReceive(sourceReceiveId)
  → Receive 詳細を C-WEB へ
```

### O4: Person 詳細タイムライン (US-10)

**主導**: person-ledger-service

```
C-WEB → person-ledger-service: getTimeline(personId)
  → DynamoDB Single-table クエリ (PK=USER#userId, SK begins_with PERSON#personId)
  → Receive / Give を時系列に並べた LedgerTimelineEntry[] を返却
```

### O5: お返し未対応の俯瞰 (US-11)

**主導**: person-ledger-service

```
C-WEB → person-ledger-service: listPendingReturns(userId)
  → DynamoDB クエリ: ReceiveRecord で status='pending' のもの
  → 締切残日数を L-RULES.culturalDeadline() で計算 (Lambda 内)
  → PendingReturn[] を C-WEB へ
```

### O6: Analytics 集計 (US-16〜US-19)

**主導**: analytics-service

```
C-WEB → analytics-service:
  - getTotals → DynamoDB 集計 (Person × 年 × 用途)
  - getRoiHeatmap → Receive/Give 集計 → ヒートマップセル生成
  - getBalanceWarnings → L-RULES.balanceWarning() を各 Person に適用
  - getTaxThresholdAlerts → L-RULES.taxThresholdStatus() を年単位累計に適用
→ ダッシュボードへ
```

### O7: 認証 (US-01) — マジックリンク (Plan A) + Hosted UI (Plan B)

**主導**: auth-service

> **改訂 (§S5 反映)**: 詳細手順を明示し、Plan B 切替条件を残す。

**Plan A (本線): Cognito Custom Auth + Magic Link**

```
1. C-WEB → auth-service: requestMagicLink(email)
   → Cognito InitiateAuth (CUSTOM_AUTH flow)
2. Cognito が Lambda Triggers を順次呼び出し:
   a. DefineAuthChallenge: 1 ターン目 → CUSTOM_CHALLENGE
   b. CreateAuthChallenge:
      - nonce = UUID v4
      - DynamoDB MagicLinkTokens に { token=nonce, email, expiresAt=now+15min } を put
      - SES で email にリンク送信 (https://noshi.example/auth/callback?token=<nonce>)
3. ユーザーがメール内リンク → C-WEB → auth-service: verifyMagicLink(token)
   → Cognito RespondToAuthChallenge with token
   → Lambda Trigger VerifyAuthChallenge:
      - DynamoDB MagicLinkTokens を Get → expiresAt 内なら一致確認
      - 一致したら token を即削除 (使い切り = リプレイ防止)
4. Cognito → ID/Access/Refresh token 発行
5. C-WEB: token を HttpOnly + Secure + SameSite=Lax Cookie で保持
6. Refresh Token Rotation を Cognito 設定で有効化
```

**Plan B (フォールバック): Cognito Hosted UI + メール+パスワード**

- Sprint 1 終了時点で Plan A の 3 つの Lambda Trigger が動かない場合、Hosted UI に切替
- C-WEB の認証画面を `https://<userpool-domain>/login?...` にリダイレクト
- 切替判断: Sprint 1 レビュー時にリードが意思決定

### O8: デモアカウント自動ログイン (US-21) — Pool 方式

**主導**: auth-service

> **改訂 (§C2 / §m2 反映)**: Pool 方式 + AdminInitiateAuth + Lease 管理。

```
1. ブースデバイスのブラウザが起動 → C-WEB が Function URL の /auth/demo/start?deviceId=<> を叩く
2. auth-service.startDemoSession(deviceId):
   a. LeaseLock テーブルを楽観ロック Query → leaseStatus='available' な demoUser を 1 つ選ぶ
   b. UpdateItem with ConditionExpression: leaseStatus='available' → 'leased'
      attrs: leaseDeviceId=<deviceId>, leaseExpiresAt=now+30min
   c. Secrets Manager から Pool 共通 password を取得
   d. Cognito AdminInitiateAuth (USER_PASSWORD_AUTH) で demoUser のトークン取得
   e. token + demoUserId を返却
3. C-WEB: token を Session Storage に保持 (Cookie ではなく揮発)
4. リース終了トリガ:
   - TTL 30 分経過 (DynamoDB TTL)
   - 「次の来場者へ」ボタン → /auth/demo/release を叩く
5. Lease 解放 → leaseStatus='cleaning' → SQS にメッセージ → Cleaner Lambda 起動:
   a. personLedger.deleteAllUserData(demoUserId) でデータ全削除
   b. S3 prefix 削除
   c. seed/persona-data/<demoUserId>.json から再投入
   d. leaseStatus='available' に戻す
6. Pool 全 leased 状態のとき: フロントが「デモが満員です」表示
```

### O9: リマインド送信 (US-15) — cron + DynamoDB query 方式

**主導**: give-service + delivery-service (定期 cron トリガ)

> **改訂 (§S3 反映)**: ユーザー単位 EventBridge Rule を作ると上限超過。**1 つの cron rule + Lambda が DynamoDB を query** に変更。

```
1. give-service.scheduleReminder(input):
   - DynamoDB に { userId, reminderId, fireAt, channel, payload } を put
   - SK = REMINDER#<personId>#<reminderId>、GSI1PK = REMINDER_FIRE_DATE / GSI1SK = fireAt
2. EventBridge cron rule (毎日朝 8:00 JST 1 つだけ) → ReminderDispatcher Lambda
3. ReminderDispatcher Lambda:
   - GSI1 を Query (fireAt <= today) で due reminders を取得
   - delivery-service.sendReminderMail(reminder) を並列呼び出し
   - 送信完了したリマインドは DynamoDB から削除 or 送信履歴 SK にステータス反映
4. delivery-service が SES.SendEmail
```

> **将来案 (β-stretch)**: ユーザー数が増えたら EventBridge Scheduler に移行 (1 アカウント 100 万スケジュール対応)。

---

## サービス間通信パターン

> **改訂 (§C1, §S3, §S6, §M1 反映)**: SSE は Function URL 経由 / リマインドは cron + DDB / 共通ロジックは workspace パッケージ / EC は自前 mock Lambda。

| パターン | 採用箇所 |
|---|---|
| **API Gateway → Lambda (REST)** | C-WEB ↔ 各 Lambda サービス (Cognito JWT Authorizer) |
| **Lambda Function URL + Response Streaming (SSE)** | C-WEB ↔ C-RECEIVE.generateLetterStream (礼状)、C-AUTH のデモ起動 (Pool acquire) |
| **Lambda → AgentCore (Bedrock SDK InvokeAgent)** | C-RECEIVE / C-GIVE → C-AGENT (LLM 判断系のみ) |
| **Lambda → Bedrock (Direct InvokeModel)** | 一部の単発推論 (Guardrails 適用込み) で AgentCore を介さないケース |
| **Lambda → Lambda (Direct invoke)** | 原則使わない (疎結合維持)。DynamoDB や Event 経由 |
| **`@noshi/person-ledger` パッケージ呼び出し** | C-RECEIVE / C-GIVE / C-ANALYTICS / C-AUTH 各 Lambda が同パッケージを取り込んで直呼び (型レベル userId 強制) |
| **Lambda → DynamoDB (AWS SDK)** | パッケージ経由が原則 |
| **Lambda → S3 (presigned URL)** | C-RECEIVE → ブラウザ経由アップロード |
| **EventBridge cron Rule → Lambda** | `0 23 * * ? *` (毎朝 8:00 JST) → ReminderDispatcher。1 つだけ |
| **DynamoDB Streams → Aggregator Lambda** | (β-stretch) Analytics 集計の増分更新 |
| **SQS → Cleaner Lambda** | デモアカウント Lease 解放後のクリーンアップ |
| **Lambda → SES (AWS SDK)** | C-DELIVERY (礼状 / リマインド / マジックリンク) |
| **Lambda → ec-sandbox-service (Function URL)** | C-DELIVERY → 自前 EC mock Lambda |
| **Cognito Lambda Triggers** | C-AUTH のマジックリンク Custom Auth (DefineAuthChallenge / CreateAuthChallenge / VerifyAuthChallenge) |

---

## 横断関心事 (Cross-Cutting)

> **改訂 (§S6 / §m3 / §M2 / §M5 / §m5 反映)**

| 関心 | 採用方針 (Application Design 段階) |
|---|---|
| **認証認可** | API Gateway Lambda Authorizer (Cognito JWT) + Function URL の Lambda Authorizer。`@noshi/auth-context` パッケージの `requireUserScope(event)` を全 Lambda の入口で呼ぶ規約 |
| **データアクセス** | `@noshi/person-ledger` パッケージ経由。`userId` を必須引数とし型で強制 |
| **ロギング** | CloudWatch Logs。`@noshi/logger` (L-MODELS 内) で構造化 JSON 共通ロガーを提供 |
| **エラー応答** | API レスポンスは Problem Details (RFC 9457) スタイルで統一 |
| **シークレット参照** | Secrets Manager / SSM。Lambda 環境変数で参照 ARN のみ |
| **観測性** | CloudWatch Logs Insights。AgentCore Observability / X-Ray の採用は NFR Design で要否決定 |
| **Input Guardrail** | ユーザー入力 / 画像内テキストに対し PROMPT_ATTACK 検出 / DENY_TOPICS でブロック |
| **Output Guardrail** | LLM 出力 (礼状 / 候補 / 推奨) に対し SEXUAL / HATE / VIOLENCE / MISCONDUCT mask |
| **リトライ / フォールバック** | AgentCore Tool 内 (Bedrock 呼び出し) は SDK 標準 retry (exp backoff、最大 3 回) / orchestration Lambda は モデル A→B フォールバック (Sonnet→Haiku) / 全体 Lambda タイムアウト 60 秒 |
| **アカウント削除** | Soft-delete (STATUS=deleted) + 非同期 Cleaner で物理削除 (SLA: 30 日以内に完了) |
