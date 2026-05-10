# Component Methods — noshi

各コンポーネントのメソッド署名 (高レベル)。詳細なビジネスロジック・しきい値・エラーパターンは Functional Design (per-unit, CONSTRUCTION) で確定する。

> **記法**: TypeScript 型表記。永続化 / 認証チェック / バリデーションの詳細は本ステージでは扱わない。

---

## C-WEB (Next.js)

> Next.js の App Router ページとサーバアクション / API ルート。フロント側のメソッド署名は UI コンポーネントに散らばるため、ここでは BFF として外部に露出する API ルートを列挙する。

| メソッド | 説明 |
|---|---|
| `GET /api/me` | 現在のセッションユーザー情報取得 |
| `POST /api/auth/magic-link` | マジックリンクメール要求 (C-AUTH 経由) |
| `GET /api/auth/callback?token=` | マジックリンク検証 → セッション確立 |
| `POST /api/demo/start` | デモセッション開始 (C-AUTH 経由) |
| `* /api/{persons,receives,gives,analytics}/...` | 各バックエンドサービスへのプロキシ (型変換 + 認証チェック) |

---

## C-AUTH

```typescript
requestMagicLink(email: string): Promise<void>
verifyMagicLink(token: string): Promise<{ idToken: string; accessToken: string; refreshToken: string }>
startDemoSession(deviceId: string): Promise<{ idToken: string; accessToken: string; demoUserId: string }>
revokeSession(refreshToken: string): Promise<void>
```

---

## C-PERSON

```typescript
createPerson(input: PersonInput): Promise<Person>
updatePerson(personId: string, patch: Partial<PersonInput>): Promise<Person>
deletePerson(personId: string, options: { force?: boolean }): Promise<void>
listPersons(filter?: PersonFilter): Promise<Person[]>
getPerson(personId: string): Promise<Person>

appendReceiveRecord(personId: string, record: ReceiveRecordInput): Promise<ReceiveRecord>
appendGiveRecord(personId: string, record: GiveRecordInput): Promise<GiveRecord>
linkReceiveToGive(receiveId: string, giveId: string): Promise<void>

getTimeline(personId: string, options?: TimelineOptions): Promise<LedgerTimelineEntry[]>
listPendingReturns(userId: string): Promise<PendingReturn[]>

deleteAllUserData(userId: string): Promise<void>  // FR-U3 対応
```

---

## C-RECEIVE

> **改訂 (§C1, §C4 反映)**: `generateLetter` は Lambda Function URL ルート経由 (API Gateway 不可)。`suggestReturnGifts` は内部で L-RULES 直呼び。

```typescript
// API Gateway 経由 (REST)
issueUploadUrl(input: { mimeType: string; size: number }): Promise<{ uploadUrl: string; objectKey: string }>
createReceiveDraft(input: { objectKeys: string[]; notes?: string }): Promise<ReceiveDraft>

// extract: AgentCore 経由 (LLM)
extractFromImages(receiveId: string): Promise<ExtractedGiftInfo>
// 内部:
//   if (isDemoSession) return DemoPrecomputed.get(receiveId)        // §S1 デモパス
//   else return agentCore.invokeAgent('extract', { s3Keys })

// suggest: L-RULES 直呼び + AgentCore (LLM 判断のみ)
suggestReturnGifts(receiveId: string): Promise<ReturnSuggestion & { deadline: Date }>
// 内部:
//   const range = LRULES.halfReturnRange(amount, purpose)            // 純粋関数
//   const deadline = LRULES.culturalDeadline(purpose, today)         // 純粋関数
//   const products = await agentCore.invokeAgent('suggest_products', { range, categoryHint })
//   return { products, range, deadline }

// letter: Lambda Function URL + Response Streaming (API Gateway ではない)
generateLetterStream(receiveId: string): AsyncIterable<LetterChunk>
// Function URL endpoint: GET /receives/{id}/letter-stream
// Lambda Authorizer で Cognito JWT 検証

// ユーザー補正 → 確定 (REST)
applyExtractionCorrection(receiveId: string, patch: Partial<ExtractedGiftInfo>): Promise<ExtractedGiftInfo>
selectReturnGift(receiveId: string, selection: ProductSelection): Promise<void>
saveLetterDraft(receiveId: string, text: string): Promise<void>

// 内祝い承認 → @noshi/person-ledger 直呼び + C-DELIVERY 並行ディスパッチ
approveAndShip(receiveId: string): Promise<{ receiveRecord: ReceiveRecord; giveRecord: GiveRecord; deliveryJobIds: string[] }>
```

---

## C-GIVE

```typescript
createGiveRecord(input: GiveRecordInput): Promise<GiveRecord>
updateGiveRecord(giveId: string, patch: Partial<GiveRecordInput>): Promise<GiveRecord>
deleteGiveRecord(giveId: string): Promise<void>

recommendNextAmount(input: { personId: string; purpose: GivePurpose }): Promise<AmountRecommendation>
// 内部で AgentCore の recommend_amount ツール呼び出し → L-RULES 計算 + LLM 判断

resolveSourceReceive(giveId: string): Promise<ReceiveRecord | null>  // US-13: 内祝い ↔ 元 Receive 遡り

scheduleReminder(input: ReminderInput): Promise<ReminderId>
listReminders(userId: string): Promise<Reminder[]>
cancelReminder(reminderId: string): Promise<void>
```

---

## C-ANALYTICS

```typescript
getTotals(input: { groupBy: 'person' | 'year' | 'purpose'; direction?: 'receive' | 'give' | 'both' }): Promise<TotalsResult>
getRoiHeatmap(userId: string): Promise<HeatmapCell[]>           // FR-A2 / US-17
getBalanceWarnings(userId: string, threshold?: number): Promise<BalanceWarning[]>  // FR-A3 / US-18
getTaxThresholdAlerts(userId: string, year: number): Promise<TaxAlert[]>           // FR-A4 / US-19

// 内部で L-RULES の `balanceWarning` / `taxThresholdStatus` を呼ぶ
```

---

## C-DELIVERY

```typescript
sendReturnLetter(input: {
  toEmail: string
  toName: string
  letterText: string
  receiveContext: ReceiveContext
}): Promise<DeliveryResult>  // SES

placeOrder(input: {
  productId: string
  receiverName: string
  receiverAddress?: PostalAddress  // モック時は省略可
  amount: number
}): Promise<OrderResult>          // 外部 EC サンドボックス API (Q7=C)

sendReminderMail(input: {
  toEmail: string
  reminder: Reminder
}): Promise<DeliveryResult>        // SES (FR-G5 リマインド)
```

---

## C-AGENT (AgentCore Agent + ツール)

> **改訂 (§C4 反映)**: AgentCore は **LLM 判断系ツールのみ** を持つ。純粋計算は呼び出し側 Lambda が L-RULES を直接呼ぶ。

```typescript
// AgentCore 起動 (C-RECEIVE / C-GIVE から、LLM 判断が必要なタスクのみ)
invokeAgent(input: {
  sessionId: string
  userId: string
  task: AgentTask           // 'extract' | 'suggest_products' | 'letter' | 'estimate_relationship' | 'judge_amount'
  payload: Record<string, unknown>
}): AsyncIterable<AgentEvent>  // ストリーミング応答

// 登録するツール定義 (LLM 判断系のみ):
type Tool =
  | { name: 'extract_gift_image';     input: { s3Keys: string[] };                                                  output: ExtractedGiftInfo }
  | { name: 'suggest_return_gifts';   input: { range: { min: number; max: number }; categoryHint?: string };        output: ReturnSuggestion }
  | { name: 'generate_letter';        input: { recipient: PersonRef; gift: ExtractedGiftInfo; tone?: LetterTone };  output: AsyncStream<string> }
  | { name: 'estimate_relationship';  input: { nameHint?: string; history?: HistorySummary };                       output: RelationshipEstimate }
  | { name: 'recommend_amount';       input: { baseRange: { min: number; max: number }; history: HistorySummary; ageGroup: AgeGroup }; output: AmountRecommendation }
```

> **削除済 (orchestration Lambda が L-RULES を直接呼ぶ)**:
> - ~~`calculate_return_range`~~ → 呼び出し側で `L-RULES.halfReturnRange()` / `thirdReturnRange()`
> - ~~`lookup_cultural_deadline`~~ → 呼び出し側で `L-RULES.culturalDeadline()`
> - ~~`detect_balance_warning`~~ → C-ANALYTICS が `L-RULES.balanceWarning()`
> - ~~`detect_tax_threshold`~~ → C-ANALYTICS が `L-RULES.taxThresholdStatus()`

> **ツール実装方針**:
> - `extract_gift_image` → Bedrock マルチモーダル ワンショット (Q4=A)、Input Guardrails (PROMPT_ATTACK / DENY_TOPICS) 適用
> - `suggest_return_gifts` / `generate_letter` / `estimate_relationship` / `recommend_amount` → Bedrock LLM、Output Guardrails 適用
> - リトライ: SDK 標準 (exp backoff / 3 回) / モデルフォールバック: Sonnet→Haiku

---

## L-RULES (純粋関数)

```typescript
halfReturnRange(amount: number, purpose: Purpose): { min: number; max: number }
thirdReturnRange(amount: number, purpose: Purpose): { min: number; max: number }
otoshidamaSuggestion(ageGroup: AgeGroup): { min: number; recommended: number; max: number }
culturalDeadline(purpose: Purpose, baseDate: Date): Date
taxThresholdStatus(yearlyTotal: number): { percentage: number; warningLevel: 'none' | 'caution' | 'warning' }
balanceWarning(receivedTotal: number, givenTotal: number, threshold: number): BalanceWarning | null
```

> **特徴**: 副作用なし、決定的、外部 I/O なし。**Property-Based Testing (Partial)** の主対象。

### PBT 不変条件 (§S7 で確定)

| 関数 | 不変条件 (Properties) |
|---|---|
| `halfReturnRange(amount, purpose)` | (1) `min ≤ max` (2) `min ≥ amount × 0.4` (3) `max ≤ amount × 0.6` (purpose 別の例外あり、例外は明示テスト) (4) amount=0 のとき range=0 |
| `thirdReturnRange(amount, purpose)` | (1) `min ≤ max` (2) `min ≥ amount × 0.25` (3) `max ≤ amount × 0.4` (4) `thirdReturnRange ⊆ halfReturnRange` のような包含関係はない (別レンジ) |
| `otoshidamaSuggestion(ageGroup)` | (1) `min ≤ recommended ≤ max` (2) `min ≥ 0` (3) ageGroup の順序 (未就学児 < 小学生 < ...) で `recommended` は単調非減少 |
| `culturalDeadline(purpose, baseDate)` | (1) `result > baseDate` (2) `result - baseDate ≤ 365 日` (3) **純粋関数性**: 同 `purpose` で baseDate を D 日ずらすと結果も D 日ずれる (4) purpose 不明な場合は例外 throw |
| `taxThresholdStatus(yearlyTotal)` | (1) `yearlyTotal` 単調増 → `percentage` 単調増 (2) `yearlyTotal=0` → `warningLevel='none'` (3) `yearlyTotal=1100000` 以上 → `warningLevel='warning'` (4) `percentage = yearlyTotal / 1100000 × 100` (誤差許容範囲内) |
| `balanceWarning(received, given, threshold)` | (1) `received` と `given` を swap すると warning の方向だけ反転 (受贈過多 ↔ 贈与過多) (2) `threshold ≥ 1` のとき判定可能 (3) `received == given` で warning なし (4) `threshold` 単調増で warning 件数は単調非増加 |

> 詳細な fast-check 等のテスト実装は Code Generation 段階。本表は **不変条件の合意ベース** として機能する。

---

## L-MODELS (型)

抜粋:

```typescript
type Person = {
  personId: string
  userId: string
  name: string
  relationship?: string  // 続柄
  birthday?: string      // ISO date
  notes?: string
  createdAt: string
}

type ReceiveRecord = {
  receiveId: string
  userId: string
  personId: string
  amount: number
  purpose: Purpose
  receivedAt: string
  imageKeys: string[]
  status: 'pending' | 'returned'
  linkedGiveId?: string  // FR-L5 自動判定の根拠
}

type GiveRecord = {
  giveId: string
  userId: string
  personId: string
  amount: number
  purpose: GivePurpose
  givenAt: string
  origin: 'manual' | 'receive-return'  // 内祝い起点か手動か
  sourceReceiveId?: string             // origin === 'receive-return' のとき
}

type LedgerTimelineEntry = {
  recordType: 'receive' | 'give'
  recordId: string
  date: string
  amount: number
  purpose: string
  origin?: 'manual' | 'receive-return'
}

type Purpose = '出産祝い' | '結婚祝い' | '新築祝い' | '内祝い' | '香典' | 'お年玉' | 'お中元' | 'お歳暮' | 'その他'
type GivePurpose = Purpose
type AgeGroup = '未就学児' | '小学生' | '中学生' | '高校生' | '大学生' | '社会人'
```

> 詳細な制約 (列挙の網羅 / 文字列長 / 範囲) は Functional Design で確定。
