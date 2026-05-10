# Story Generation Plan — noshi

> このファイルは User Stories 生成の **計画 + 質問 + チェックリスト** を兼ねます。
> §A の質問に `[Answer]:` タグで回答し、最後に「完了」とお知らせください。
> 回答内容に応じて §B の生成ステップを実行します。

---

## §A 質問一覧 (PART 1: PLANNING で確定が必要な事項)

各質問について `[Answer]:` の後にレター (A, B, C ...) を記入してください。
どの選択肢も合致しない場合は最後の `X) Other` を選び、自由記述してください。

### Question 1: 追加ペルソナの要否
要件文書には永続ユーザー (30 代エンジニア) と来場者 (デモ参加者) の 2 系統の触り手がいます。Story 生成時にどこまでペルソナを追加しますか?

A) 既存 2 系統 (永続ユーザー / 来場者) のみで進める
B) +1 追加: 義両親世代など「贈り主側」も登場ペルソナとして書く (ただしユーザーとしては触らない、シナリオ上の人物)
C) +2 追加: 「贈り主側」と「ハッカソン審査員」両方をシナリオ上の人物として加える
D) 永続ユーザー側に複数バリエーションを作る (例: エンジニアと、贈答が苦手な妻側ペルソナ)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 2: Story 分解アプローチ
Story の分解・整理方針はどれが望ましいですか?

A) **User Journey-Based** (ユーザーの一連の流れに沿って Story を並べる — 例: アカウント作成 → 受領登録 → 提案確認 → 内祝い発送)
B) **Feature-Based** (機能カテゴリで束ねる — Receive / Give / Analytics / Account ごとに Story を整理)
C) **Persona-Based** (永続ユーザー / 来場者 ごとに Story を分ける)
D) **Epic-Based ハイブリッド** (デモシナリオ S1〜S3 を Epic として、その下に Story をぶら下げ、それとは別に永続ユーザー向け Epic を立てる) — 推奨候補
E) **Domain-Based** (Person ledger / 文化ルール / 通知 / 認証 などドメインごと)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 3: Story の粒度
INVEST の "Small" 観点で、Story 1 つの理想サイズはどれくらいに揃えますか?

A) **小粒** (1〜2 日で実装完了する見立て、合計 30〜50 Story) — トラッキングしやすいが分量多
B) **中粒** (2〜5 日で実装完了する見立て、合計 15〜25 Story) — バランス重視 — 推奨候補
C) **大粒** (1 週間規模、合計 8〜12 Story) — Story 数は少ないが、実装時に Sub-task 分解が必要
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

### Question 4: Story テンプレート / フォーマット
Story 本文の書式はどれを採用しますか?

A) **標準形式**: 「**As a** [ペルソナ], **I want** [目的], **so that** [理由]」 + 受け入れ基準
B) **ジョブストーリー形式**: 「**When** [状況], **I want to** [動機], **so I can** [結果]」 + 受け入れ基準
C) **日本語自然文**: 「[ペルソナ] は [状況] のとき、[目的] を達成したい。それは [理由] のためである。」 + 受け入れ基準
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

### Question 5: 受け入れ基準 (Acceptance Criteria) フォーマット
受け入れ基準の書き方はどれがいいですか?

A) **Given / When / Then (BDD 形式)** — 厳密、テスト自動化と接続しやすい
B) **箇条書きチェックリスト形式** — 軽量、可読性高
C) **Given/When/Then と チェックリスト 併用** — 重要 Story は BDD、それ以外はチェックリスト — 推奨候補
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

### Question 6: β 版 (ハッカソン本番) スコープの絞り込み
Requirements の機能要件のうち、β 版 Story で必ず含めるものはどれですか? (最も近いセットを 1 つ選択)

A) **コア + 連携のみ**: Receive (画像→提案→礼状生成) + Give (履歴登録 + 相場提示) + Receive→Give 自動連携。Analytics と通知 (FR-G5) は未対応
B) **コア + 連携 + Analytics**: A に加え Analytics ダッシュボード (ROI ヒートマップ含む) を含める — テーマ整合のため強く推奨
C) **B + アカウント基盤完備**: B に加え Cognito 等によるアカウント登録・削除 (FR-U1〜U3) も β 版に含める
D) **全機能**: 通知 (FR-G5) や 発送ワークフロー まで含めて β 版に詰め込む
X) Other (please describe after [Answer]: tag below)

[Answer]: D

---

### Question 7: デモアカウント (FR-U4) の体験設計
来場者向けデモアカウントの体験はどう設計しますか?

A) **完全プリセット**: 来場者は事前作成済みのデモアカウントに自動ログインし、ペルソナ事前データが既に入った状態で開始
B) **ワンタップ生成**: 来場者がボタン 1 つで一時アカウントを作成、ペルソナ事前データが自動投入される (TTL 自動失効)
C) **来場者ごとに本物のアカウント登録**: メール入力等の手間を経て登録 (デモ向きでない)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 8: デモシナリオ Story の優先順位
デモシナリオ S1 (内祝い) / S2 (お年玉) / S3 (Analytics ROI) のうち、ハッカソン本番までに必ず動く順位は?

A) **S1 最優先** (S1 だけは完璧 → 余力で S2, S3) — Receive ↔ Give 連携の見せ場が S1 にあるため推奨候補
B) **S2 最優先** (お年玉判断は理解されやすい)
C) **S3 最優先** (テーマ整合性のインパクトが最大)
D) **3 つ並行で同程度に作る** (β 版で 3 つとも安定動作)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 9: 永続ユーザー向け Story と デモ Story のバランス
Story 件数として、永続ユーザー向け と 来場者デモ向け をどう配分しますか?

A) **7:3** (永続ユーザー中心、デモは派生として最小限)
B) **5:5** (両方均等)
C) **3:7** (デモ向け中心、永続ユーザーは将来要件として最小限)
D) **永続ユーザー Story = デモ Story の親**: デモ Story はあくまで永続 Story のサブセット (重複を作らない)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 10: 倫理 / 法務面の Story 化
要件 §10.2 の倫理 / 法務留意点 (TBD-L1 税務助言誤認 / TBD-L2 退化体感の UI コンテキスト) は Story として書きますか?

A) **書く**: それぞれ独立した Story として受け入れ基準を明示 (例: 「贈与税画面に税理士相談推奨の注意書きが表示されること」)
B) **横断的非機能要件として記述**: Story 一覧の冒頭にガイドラインを書き、各 Story の受け入れ基準で参照
C) **後段 (Application Design / Code Generation) に委ねる**: User Stories では扱わない
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

## §B 生成ステップ (PART 2: GENERATION で実行)

> 上記 §A の回答が確定した後、以下を順番に実行します。各ステップを完了したら `[x]` に更新します。

- [x] B1. `aidlc-docs/inception/user-stories/personas.md` を生成 — Q1 の回答に応じたペルソナを作成 (P1 タクヤ / P2 Summit ビジター の 2 体)
- [x] B2. `aidlc-docs/inception/user-stories/stories.md` を生成 — User Journey-Based で 7 Epic / 24 Story
- [x] B3. 各 Story を日本語自然文テンプレートで記述
- [x] B4. 各 Story にチェックリスト形式の受け入れ基準を付与
- [x] B5. β-must / β-should / β-stretch の 3 段階ラベルを全 Story に付与 (全機能 β = Q6=D)
- [x] B6. デモアカウント完全プリセット仕様を US-21 に反映 (Q7=A)
- [x] B7. S1 最優先 (Q8=A) に従い、デモ Story は US-21→US-22(S1)→US-23(S2/S3 余力) の順で配置
- [x] B8. 永続ユーザー Story 20 件 / デモ Story 3 件 (Q9=A 永続中心方針と整合 — US-15 削除に伴い 21→20)
- [x] B9. 倫理 / 法務 Story は本ステージでは作成せず、TBD-L1 / TBD-L2 として US-19 等の受け入れ基準で軽く言及 (Q10=C)
- [x] B10. INVEST セルフチェック表を `stories.md` 末尾に追加 (主要依存も明示)
- [x] B11. ペルソナ ↔ Story マッピング表を `stories.md` 末尾に追加
- [x] B12. 要件 ↔ Story トレーサビリティ表を `stories.md` 末尾に追加
- [ ] B13. 完了チェック — `aidlc-docs/aidlc-state.md` を更新し、ユーザーに承認依頼を提示

---

## §C 生成しない方針 (Out of Scope for User Stories Stage)
- 技術選定 (フレームワーク / モデル / DB スキーマ) — Application Design 以降
- 詳細 UI ワイヤーフレーム — Application Design 以降
- スプリント / 工数見積 — User Stories 範囲外
- ハードコードされたビジネスロジック詳細 — Functional Design 以降
