# 要件確認質問 (Requirement Verification Questions)

以下の質問にお答えください。各質問の `[Answer]:` タグの後に、選択肢のレター (A, B, C ...) を記入してください。
どの選択肢も合致しない場合は最後の `X) Other` を選び、自由記述してください。

すべての回答が終わったら「完了」「done」などとお知らせください。

---

## Question 1: プロジェクト概要
このリポジトリ (`aws-summit-2026-hackathon`) で作る成果物は何ですか?

A) AWS Summit 2026 ハッカソン向けのデモアプリケーション (Web アプリ)
B) AWS Summit 2026 ハッカソン向けの AI エージェント / 生成 AI プロダクト
C) AWS Summit 2026 ハッカソン向けの IoT / エッジ系プロトタイプ
D) AWS Summit 2026 ハッカソン向けのデータ分析・可視化プロダクト
E) まだ題材が決まっていない (アイデア出しから一緒に進めたい)
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Question 2: 主なターゲットユーザー / 利用シーン
誰が、どのような場面で使うことを想定していますか?

A) ハッカソン審査員へのデモ / プレゼンテーション用 (短時間で価値が伝わるデモ)
B) 一般エンドユーザー (AWS Summit 来場者など) が触るインタラクティブ体験
C) 企業内ユーザー (業務効率化・社内ツール) を想定したプロダクト
D) 開発者向けのライブラリ / OSS / ツールキット
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Question 3: 開発スコープと締切
ハッカソン本番までに、どこまで作り込む想定ですか?

A) コア機能だけ動く MVP (Minimum Viable Product) — デモで 1 シナリオが通れば OK
B) 主要機能を一通り揃えた β 版 — 複数シナリオを動作させたい
C) 本番運用も視野に入れた完成度 — 認証・監視・スケールを含む
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Question 4: 主要な AWS サービス / 技術スタックの希望
ハッカソンの題目柄、特に使いたい AWS サービスや技術はありますか?

A) Bedrock / Bedrock AgentCore など 生成 AI / エージェント系
B) Lambda + API Gateway + DynamoDB などサーバーレス系
C) ECS/EKS + RDS などコンテナ + RDB の王道構成
D) IoT Core / Kinesis / Greengrass など IoT・ストリーミング系
E) 特に指定なし — 要件に応じておすすめしてほしい
X) Other (please describe after [Answer]: tag below)

[Answer]: E

---

## Question 5: フロントエンドの形態
ユーザーが触れる UI はどの形が望ましいですか?

A) Web アプリ (React / Next.js などの SPA / SSR)
B) モバイル (iOS / Android / クロスプラットフォーム)
C) CLI / API のみ (UI は別途作らない、または最小限)
D) 物理デバイス + ダッシュボード (IoT 系)
E) UI は不要 (バックエンド/エージェント単体)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 6: 主要な非機能要件 (NFR) の優先順位
最優先で意識したい非機能要件はどれですか? (複数当てはまる場合は最重要のものを 1 つ)

A) デモ時のレスポンスの速さ・体験のスムーズさ (パフォーマンス)
B) 失敗しにくさ・落ちにくさ (信頼性・回復性)
C) 個人情報・認証情報の安全な扱い (セキュリティ)
D) クラウドコストを抑えられること (コスト効率)
E) ハッカソン後にも展示・運用できるメンテナンス性
X) Other (please describe after [Answer]: tag below)

[Answer]: C,D

---

## Question 7: 開発体制・進め方
誰が、どのように開発を進める想定ですか?

A) 個人プロジェクト (自分 1 人)
B) 少人数チーム (2 〜 4 人) で分担
C) 中規模チーム (5 人以上)
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Question 8: ドキュメント・成果物の言語
今後生成されるドキュメント (要件書・設計書・コードコメント等) の主言語はどうしますか?

A) すべて日本語
B) ドキュメントは日本語、コード内のコメント・識別子は英語
C) すべて英語
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 9: Security Extensions (拡張機能)
このプロジェクトで セキュリティ拡張ルールを強制適用しますか?

A) Yes — すべての SECURITY ルールを必須制約として強制 (本番運用クラスのアプリケーションに推奨)
B) No — SECURITY ルールをスキップ (PoC / プロトタイプ / 実験的プロジェクトに適する)
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 10: Property-Based Testing Extension (拡張機能)
このプロジェクトで プロパティベーステスト (PBT) ルールを強制適用しますか?

A) Yes — すべての PBT ルールを必須制約として強制 (ビジネスロジック・データ変換・シリアライズ・ステートフルコンポーネントを含むプロジェクトに推奨)
B) Partial — 純粋関数とシリアライズのラウンドトリップに対してのみ PBT ルールを適用 (アルゴリズム的複雑さが限定的なプロジェクトに適する)
C) No — PBT ルールをスキップ (シンプルな CRUD アプリ / UI のみのプロジェクト / ビジネスロジックの薄い統合層に適する)
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---
