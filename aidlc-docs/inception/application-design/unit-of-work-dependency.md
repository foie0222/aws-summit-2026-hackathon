# Unit of Work Dependencies — noshi

ユニット間の依存関係 + 並行可能性 + Critical Path。

---

## 1. 依存マトリクス (行: 呼ぶ側 / 列: 呼ばれる側)

| → 依存先<br>↓ ユニット | U01<br>frontend | U02<br>identity | U03<br>person<br>ledger | U04<br>receive | U05<br>give | U06<br>analytics | U07<br>delivery | U08<br>agent | U09<br>ec<br>sandbox | U10<br>shared | U11<br>infra |
|---|---|---|---|---|---|---|---|---|---|---|---|
| **U01 frontend** | — | ● HTTPS | ● HTTPS | ● HTTPS+SSE | ● HTTPS | ● HTTPS | (間接) | (間接) | (間接) | ● Build | ● Deploy |
| **U02 identity** | — | — | ● SDK (Cleaner) | — | — | — | ● SDK (magic mail) | — | — | ● Build | ● Deploy |
| **U03 person-ledger** | — | — | — | — | — | — | — | — | — | ● Build | ● Deploy |
| **U04 receive** | — | — | ● SDK | — | — | — | ● SDK | ● SDK (Bedrock) | — | ● Build | ● Deploy |
| **U05 give** | — | — | ● SDK | — | — | — | ● SDK (reminder) | ● SDK | — | ● Build | ● Deploy |
| **U06 analytics** | — | — | ● SDK | — | — | — | — | — | — | ● Build | ● Deploy |
| **U07 delivery** | — | — | — | — | — | — | — | — | ● HTTP | ● Build | ● Deploy |
| **U08 agent** | — | — | ● SDK (tool 内) | — | — | — | — | — | — | ● Build | ● Deploy |
| **U09 ec-sandbox** | — | — | — | — | — | — | — | — | — | ● Build | ● Deploy |
| **U10 shared** | — | — | — | — | — | — | — | — | — | — | (✗) |
| **U11 infra** | — | — | — | — | — | — | — | — | — | — | — |

凡例:
- `●` 直接依存 (パッケージ取り込み or HTTPS 呼出)
- `(間接)` 別ユニット経由で結びつくが直接依存はない
- `Build` U10 のパッケージをビルド時取り込み
- `Deploy` U11 が CDK でデプロイ
- `(✗)` U10 から U11 への依存はなし (U10 は純粋ライブラリ)

---

## 2. DAG 検証 (循環依存なし)

```
U10 shared (libraries)            ←ベースライン (依存なし)
    ↑
    │
U11 infra (CDK)                   ←U10 をビルド資産として参照する場合あり、循環なし
    │
    ├─ デプロイ→ U02〜U09 サービス群
    │           ↓ それぞれ U10 を build dep
    │
    └─ U01 frontend を Amplify にデプロイ
                ↑
                │ U02〜U06 を HTTPS で呼ぶ
                ↓
            (依存ライン上 U07 / U08 / U09 は U04・U05 経由)
```

依存グラフは DAG (循環なし) であることを確認。

---

## 3. ユニット間の通信パターン (再掲)

| パターン | 該当ユニットペア |
|---|---|
| **API Gateway REST** | U01 → U02 / U03 / U04 / U05 / U06 |
| **Lambda Function URL (SSE)** | U01 → U04 (letter stream) / U02 (デモ Lease) |
| **shared パッケージ取り込み (build-time)** | 他全ユニット → U10 |
| **DynamoDB 共有テーブル経由** | U02 / U03 / U04 / U05 / U06 / U08 のすべてが U10 `@noshi/person-ledger` 経由でアクセス |
| **Bedrock SDK (InvokeAgent)** | U04 / U05 → U08 |
| **HTTPS 内部呼出** | U07 → U09 (ec-sandbox) |
| **EventBridge cron Rule** | infra → U05 (ReminderDispatcher) |
| **Cognito Lambda Triggers** | Cognito → U02 (3 trigger Lambda) |
| **SQS** | U02 (Cleaner キューイング) → U02 内 Cleaner Lambda、U07 のメール失敗 → DLQ |
| **CDK デプロイ** | U11 → 他全ユニット (Lambda 関数 / Function URL / Cognito / DynamoDB / S3 / SES / EventBridge / Amplify) |

---

## 4. Critical Path (実行順序)

> **Q7=C Sprint 0 freeze + 全並行 を採用**

### 4.1 Sprint 0: Contract Freeze (3〜5 日)

```
U10 shared (リード)
  ├─ ドメイン型 freeze (libs/models)
  ├─ API スキーマ freeze
  ├─ AgentCore tool スキーマ freeze
  ├─ DynamoDB key 設計 freeze
  ├─ L-RULES シグネチャ + PBT 不変条件 freeze
  └─ requireUserScope 戻り型 freeze
       ↓
U11 infra (リード、並行)
  ├─ Cognito UserPool 立ち上げ
  ├─ DynamoDB single-table 立ち上げ
  ├─ API Gateway スケルトン
  ├─ Bedrock Model Access リクエスト
  ├─ SES Production Access 申請
  ├─ AgentCore Agent stub 作成
  └─ デモアカウント Pool 10 個作成 (Custom Resource)
       ↓
全ユニット skeleton (200 OK レベル)
       ↓
Sprint 1 へ
```

### 4.2 Sprint 1〜3: 全並行 (機能実装)

```
全 11 ユニットが並行進行。各ユニットの完了マイルストーン:

U10 shared          ─→ Sprint 1 で全関数実装 + PBT テスト
U03 person-ledger   ─→ Sprint 1 で CRUD + Timeline 動作
U02 identity        ─→ Sprint 1 で マジックリンク Plan A 試行 / Plan B フォールバック判断
U04 receive         ─→ Sprint 1 で extract → suggest までの API、Sprint 2 で letter stream + approveAndShip
U08 agent           ─→ Sprint 1 で extract / suggest 実装、Sprint 2 で letter / recommend / estimate
U09 ec-sandbox      ─→ Sprint 1 (簡単、半日)
U07 delivery        ─→ Sprint 2 (U04 / U05 / U09 が出揃ってから統合)
U05 give            ─→ Sprint 2 (CRUD) + Sprint 3 (推奨 + リマインド cron)
U06 analytics       ─→ Sprint 3 (集計表示)
U01 frontend        ─→ Sprint 1 (auth + receive UI 骨格) + Sprint 2 (give + analytics) + Sprint 3 (デモ + 仕上げ)
U11 infra           ─→ Sprint 1〜3 で Lambda / Function URL / EventBridge を順次追加
```

### 4.3 Sprint Final: Build and Test

```
全ユニット統合 → E2E (デモ S1) 通し → 性能検証 → セキュリティ チェック → 本番デプロイ準備
```

---

## 5. 並行可能性 / ブロッキング解析

### 5.1 Sprint 1 で並行可能なユニット

Sprint 0 freeze 後、以下は **完全並行** で進められる:
- U10 shared (Sprint 0 から継続的に実装拡充)
- U03 person-ledger
- U02 identity
- U04 receive (skeleton)
- U05 give (skeleton)
- U06 analytics (skeleton)
- U07 delivery
- U08 agent
- U09 ec-sandbox
- U01 frontend (auth ページ + receive 画面の骨)
- U11 infra (継続的に CDK 拡充)

### 5.2 ブロッキング関係 (Critical Dependencies)

| ブロッカー | ブロックされる側 | 解消条件 |
|---|---|---|
| U10 shared (型 freeze) | 他全ユニット | Sprint 0 終了 |
| U11 infra (DynamoDB / Cognito / Bedrock) | U02 / U03 / U08 のテスト | Sprint 0 終了 |
| U03 person-ledger (CRUD 動作) | U04 / U05 / U06 の統合 | Sprint 1 中盤 |
| U08 agent (extract ツール動作) | U04 (デモ S1 通し試験) | Sprint 1 終盤 |
| U07 delivery (SES 実送信) | U04 / U05 (E2E 完成) | SES Production Access 取得 + Sprint 2 |
| U09 ec-sandbox | U07 → U04 (発注完了) | Sprint 1 半日 |

### 5.3 リスク要因

| リスク | 影響 | 緩和策 |
|---|---|---|
| **SES Production Access** が間に合わない | U02 マジックリンク + U07 メール送信 が動かない | Sprint 0 で **即申請** / Plan B = Hosted UI 認証 |
| **U10 freeze の遅延** | 全ユニット Sprint 1 に入れない | Sprint 0 を 5 日でタイムボックス、未確定領域は仮値 freeze で進行 |
| **Bedrock Model Access** 遅延 | U08 / U04 / U05 の AI 機能が動かない | Sprint 0 で即申請 / 利用モデルを Haiku 単独に絞ることでリスク軽減 |
| **U01 frontend の作業量超過** | UI 完成度が β-must レベルに届かず | 内部モジュール分割 (§4 U01) で 2 名ローテーション、`(app)/settings` などは β-stretch カット可 |
| **U08 AgentCore tool の挙動不安定** | デモ S1 が安定しない | デモパスは事前計算キャッシュ (§S1 ラバーダック) でフォールバック |

---

## 6. デプロイ順序 (Sprint Final / 本番)

```
1. U11 infra (CDK)
   └─ network → identity (Cognito) → storage (DDB / S3) → compute (Lambdas) → agent → delivery → frontend → observability
2. U10 shared 配信 (パッケージは Lambda zip にバンドル済 = 実体デプロイなし)
3. U02〜U09 の Lambda アセット
4. U08 agent (AgentCore Agent デプロイ + Tool Lambda)
5. U01 frontend (Amplify Hosting)
6. デモアカウント Pool seed
7. Smoke test (デモ S1 通し / マジックリンク受信 / Analytics 表示)
```

---

## 7. ユニット完了 (Definition of Done)

各ユニットの DoD は Functional Design / Code Generation / Build and Test の各ステージで詳細化されるが、Application Design 段階での暫定 DoD:

| Unit | DoD (Application Design 段階の暫定) |
|---|---|
| U01 | 全ページが U10 型でビルド成功、auth + receive + analytics の主要画面が表示できる |
| U02 | マジックリンク Plan A or Plan B が動作、デモ Lease 発行 + 解放が動作 |
| U03 | CRUD + Timeline + 削除が動作、PBT (`@noshi/person-ledger.invariants`) パス |
| U04 | extract → suggest → letter stream → approveAndShip が E2E で完走 (デモパス含む) |
| U05 | 手動登録 + 推奨 + 内祝い遡り + リマインド cron が動作 |
| U06 | 4 種ダッシュボード (累計 / ROI / 警告 / 110 万) が表示 |
| U07 | SES 礼状送信 + ec-sandbox 発注 + リマインド送信が動作 |
| U08 | 5 ツールが AgentCore Agent として動作、Guardrails 入出力フィルタが効く |
| U09 | mock 発注エンドポイントが動作 (3% rejected 含む) |
| U10 | 全パッケージビルド成功、PBT パス、API スキーマ stable |
| U11 | 全 CDK Stack が Tokyo にデプロイ可能、SES Production / Bedrock Access 取得済 |
