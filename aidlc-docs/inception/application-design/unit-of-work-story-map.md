# Story ↔ Unit Map — noshi

23 Story がどのユニットに割り当てられるかを示す。Sprint 1 以降の機能実装で、各ユニット担当が自身の Story を完成させていく。

> **Note**: 多くの Story は **複数ユニットを跨る** (Frontend + Backend + Shared)。これは Q1=A の細粒選択の必然。**主担当ユニット (Owner)** = Story の主役を成すユニット。

---

## 1. 全 Story → ユニット 割当 (主担当 / 関与)

| Story | タイトル | ラベル | 主担当 (Owner) | 関与ユニット |
|---|---|---|---|---|
| **US-01** | アカウント登録 | β-should | U02 identity | U01 (UI), U10 (auth-context, models), U11 (Cognito CDK) |
| **US-02** | 自分情報セットアップ | β-should | U03 person-ledger | U01 (UI), U10 (models, person-ledger) |
| **US-03** | 親族・友人 (Person) 登録 | β-should | U03 person-ledger | U01 (UI), U10 |
| **US-04** | ご祝儀の画像取り込み | β-must | U04 receive | U01 (UI), U10, U11 (S3) |
| **US-05** | AI による構造化抽出と補正 | β-must | U04 receive | U08 agent (extract tool), U01, U10 |
| **US-06** | 半返しレンジとお返し品候補 | β-must | U04 receive | U10 (L-RULES), U08 (suggest tool), U01 |
| **US-07** | 文化的締切日の確認 | β-must | U04 receive | U10 (L-RULES.culturalDeadline), U01 |
| **US-08** | 礼状文面の AI 生成と微修正 | β-must | U04 receive | U08 (letter tool), U01, U11 (Function URL) |
| **US-09** | 内祝い承認 → 発送 → Give 履歴連携 | β-must | U04 receive | U03 (write), U07 (delivery), U09 (ec-sandbox), U01 |
| **US-10** | Person 詳細での統合タイムライン | β-must | U03 person-ledger | U01 (UI), U10 |
| **US-11** | お返し済 / 未対応の俯瞰 | β-should | U03 person-ledger | U10 (L-RULES.deadline), U01 |
| **US-12** | 贈与の手動登録 | β-should | U05 give | U03 (write), U01, U10 |
| **US-13** | 内祝いから元 Receive への遡り | β-should | U05 give | U03 (lookup), U01 |
| **US-14** | 年齢別相場 + 親族間バランスでの推奨 | β-should | U05 give | U10 (L-RULES), U08 (recommend tool), U01 |
| **US-15** | 贈答イベントのリマインド受信 | β-stretch | U05 give | U07 (mail send), U11 (EventBridge cron) |
| **US-16** | 累計総額ダッシュボード | β-should | U06 analytics | U03 (read), U01 |
| **US-17** | 親族 ROI ヒートマップ | β-must | U06 analytics | U03 (read), U01 |
| **US-18** | あげすぎ / 少なすぎ警告 | β-should | U06 analytics | U10 (L-RULES.balanceWarning), U01 |
| **US-19** | 贈与税 110 万円枠通知 | β-should | U06 analytics | U10 (L-RULES.taxThresholdStatus), U01 |
| **US-20** | アカウントとデータの削除 | β-should | U02 identity | U03 (deleteAllUserData), U01 |
| **US-21** | デモアカウント自動ログイン | β-must | U02 identity | U03 (preset data), U01, U11 (Lease + Pool) |
| **US-22** | デモシナリオ S1 — 義両親への内祝い | β-must | U04 receive | U08, U07, U09, U03, U01 (デモ パス特化、§S1 事前計算キャッシュ含む) |
| **US-23** | デモシナリオ S2 / S3 余力枠 | β-stretch | U05 give + U06 analytics | U03, U01, U10 |

---

## 2. 担当ユニット別 Story 集計

| Unit | β-must Stories | β-should Stories | β-stretch Stories | 計 |
|---|---|---|---|---|
| **U01 frontend** (関与のみ、主担当 0) | (関与) US-04〜US-09, US-10, US-17, US-21, US-22 | (関与) その他 | US-15, US-23 | (関与 23 全件) |
| **U02 identity** (主担当 3) | US-21 | US-01, US-20 | — | **3** |
| **U03 person-ledger** (主担当 4) | US-04 (関与強い), US-10 | US-02, US-03, US-11 | — | **4** (主) |
| **U04 receive** (主担当 7) | US-04, US-05, US-06, US-07, US-08, US-09, US-22 | — | — | **7** (主、最多) |
| **U05 give** (主担当 4) | — | US-12, US-13, US-14 | US-15, US-23 | **5** |
| **U06 analytics** (主担当 4) | US-17 | US-16, US-18, US-19 | US-23 (共有) | **4** |
| **U07 delivery** (主担当 0、cross-cutting) | (関与) US-09, US-22 | — | (関与) US-15 | **0** (cross) |
| **U08 agent** (主担当 0、cross-cutting) | (関与) US-05, US-06, US-08, US-22 | (関与) US-14 | — | **0** (cross) |
| **U09 ec-sandbox** (主担当 0、cross-cutting) | (関与) US-09, US-22 | — | — | **0** (cross) |
| **U10 shared** (主担当 0、横断 enabler) | (関与) 全 Story | (関与) | (関与) | **0** (cross) |
| **U11 infra** (主担当 0、横断 enabler) | (関与) 全 Story | (関与) | (関与) | **0** (cross) |

> **観察**: U04 receive が β-must 7 件と最多。M3 (作業量偏り) はラバーダックで指摘済、U04 は受け持つ Story 数で見ても重い。**Sprint 1 の最重要ユニット**。

---

## 3. 検証: 全 23 Story が割り当てられているか

| Story 番号 | 主担当ユニットあり? |
|---|---|
| US-01 | ✅ U02 |
| US-02 | ✅ U03 |
| US-03 | ✅ U03 |
| US-04 | ✅ U04 |
| US-05 | ✅ U04 |
| US-06 | ✅ U04 |
| US-07 | ✅ U04 |
| US-08 | ✅ U04 |
| US-09 | ✅ U04 |
| US-10 | ✅ U03 |
| US-11 | ✅ U03 |
| US-12 | ✅ U05 |
| US-13 | ✅ U05 |
| US-14 | ✅ U05 |
| US-15 | ✅ U05 |
| US-16 | ✅ U06 |
| US-17 | ✅ U06 |
| US-18 | ✅ U06 |
| US-19 | ✅ U06 |
| US-20 | ✅ U02 |
| US-21 | ✅ U02 |
| US-22 | ✅ U04 |
| US-23 | ✅ U05 + U06 |

→ **23 Story すべてに主担当ユニットが割り当てられている**。漏れなし。

---

## 4. ユニット担当が「他ユニットの Story の関与役」になるパターン

Q1=A 細粒選択の必然として、cross-cutting (横断) で関与するユニットが多い。代表例:

### U10 shared (横断)
- 全 Story に関与: 型 / API スキーマ / L-RULES / person-ledger / auth-context / logger
- Sprint 0 freeze の品質が全 Story の品質を決める

### U11 infra (横断)
- 全 Story に関与: Cognito / API Gateway / Function URL / DynamoDB / S3 / Bedrock / SES / EventBridge / Amplify を立てる
- 各ユニットの Lambda が動くかどうかは U11 設定次第

### U07 delivery (cross-cutting)
- US-09, US-15, US-22 で Lambda invocations を提供
- 主担当 Story 数 0 だが Sprint 2〜3 で重要

### U08 agent (cross-cutting)
- US-05, US-06 (suggest tool 部分), US-08, US-14, US-22 で AI 推論を提供
- 主担当 Story 数 0 だが β-must の AI 機能を支える

### U09 ec-sandbox (cross-cutting)
- US-09, US-22 で発注 mock を提供
- 軽量、主担当 Story 数 0、半日実装

---

## 5. β-must / β-should / β-stretch の分布

| ラベル | Story 数 | 主担当ユニット |
|---|---|---|
| **β-must** (8) | US-04, US-05, US-06, US-07, US-08, US-09, US-10, US-17, US-21, US-22 | U02 (1), U03 (2), U04 (7), U06 (1) — 計 8 主担当 (US-09 など重複は除いた集計) |
| **β-should** (12) | US-01, US-02, US-03, US-11, US-12, US-13, US-14, US-16, US-18, US-19, US-20 | U02 (2), U03 (3), U05 (3), U06 (3) |
| **β-stretch** (2) | US-15, US-23 | U05 (1), U06+U05 (1) |

> **Sprint 1 の最優先ユニット**: U04 receive (β-must 7 件) > U03 person-ledger (β-must 2 件 + β-should 3 件) > U02 identity (β-must 1 件 + β-should 2 件) > U06 analytics (β-must 1 件)

---

## 6. ユニット完成度に応じた β 版動作シナリオ

時間切れ時のリカバリ計画 (β-stretch ラベルに合わせてカット候補):

| 完成度 | 動くデモシナリオ |
|---|---|
| **β-must 全部** + 必須 β-should | デモ S1 完走 + 永続ユーザーが Receive→Give 連携 + アカウント登録/削除 + ROI ヒートマップ |
| **β-must のみ** | デモ S1 完走 + ROI ヒートマップ表示。他は触れない |
| **デモ S1 + β-stretch カット** | US-15 / US-23 をカット、ハッカソン本番に間に合う |
