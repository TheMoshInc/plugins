# 収集プラン（正本）

領域ごとに「呼ぶツール → パラメータ → スナップショットに書く値 → 取れない物」を定める。ツール名は OpenAPI の `operationId`（MCP クライアント上では `mcp__<server>__` が付く）。件数上限・PII・失敗時の書き方など全体に効く規約は SKILL.md「必須ルール」が正本で、ここには再掲しない。

## 呼び出し順と並列化

依存のない領域は同時に呼ぶ。依存があるのは「商品 → プラン」「商品 → 購読者」「会員サイト → 会員数」の3つだけ。

```
並列A: 商品一覧 / LP一覧(上位20件1回 + 状態別件数3回) / WF一覧(limit=100) / LINE公式 / リッチメニュー / 会員サイト一覧 /
       売上サマリー / 顧客件数(4回) / コンタクト件数(メール2回・LINE2回) / コンタクトタグ / 顧客タグ / 流入経路 / ウェビナー / 素材
  ↓
並列B: 商品ごとのプラン（上位20商品）/ サブスク商品の購読者件数（3回） / 会員サイトごとの会員数
  ↓
Step 2.5: 運用中の SNS をユーザーに確認
```

## 1. 商品・プラン

| 項目 | 内容 |
|---|---|
| ツール | `getCreatorProducts`（`limit=20`）→ 商品ごとに `getCreatorProductPlans` |
| 書く値 | 商品数（`publishingStatus`: PUBLIC / LIMITED / PRIVATE 別）、商品ごとに id・名前・planCount・publicUrl の有無。プランは id・名前・`billingCycle`（ONE_TIME / MONTHLY / YEARLY）・`price.amount`・`installmentType`・`isSuspended` |
| 派生 | 「買い切りのみの商品」= 全プランが ONE_TIME。「サブスクあり商品」= MONTHLY / YEARLY のプランを1つ以上持つ商品（買い切りプランを併設していても含める）。この2区分は排他で、合計は商品数と一致する。後続の設計・分析で前提になるので必ず書く |
| 取れない物 | 購入者数・売上（売上サマリーで補う）。旧サービス（サービス2.0の商品として登録されていないもの）は一覧に出ない |

## 2. 購読者（サブスク商品のみ）

| 項目 | 内容 |
|---|---|
| ツール | `postCreatorProductSubscribers`。全フィールド必須: `planIds: []`, `keyword: ""`, `statuses: [...]`, `startDateTime: null`, `endDateTime: null`, `limit: 1`, `offset: 0`, `order: "-startedDateTime"`。`statuses` を `["active"]` / `["pastDue"]` / `["canceled"]` で3回呼び、`totalCount` だけ使う |
| 書く値 | 商品ごとの active / pastDue / canceled の件数（生のステータス名で書く）。「継続中」と言うときは active + pastDue（pastDue は決済エラーだが購読は継続している） |
| 取れない物 | 旧サービスのサブスク購読者（この商品一覧に出ない） |

## 3. LP

| 項目 | 内容 |
|---|---|
| ツール | 上位20件の表用に `getCreatorLandingPages`（`limit=20`, `sort=updatedAtDesc`, `status` 未指定 = 既定 `PUBLISHED_DRAFT`）を1回。**状態別の件数は1ページ目の内訳から数えず**、`status=PUBLISHED` / `DRAFT` / `ARCHIVED` を `limit=1` でそれぞれ呼び、`totalCount` を使う（計4回） |
| 書く値 | 状態別件数（PUBLISHED / DRAFT / ARCHIVED。各 `totalCount`）、上位20件の id・タイトル・状態・更新日 |
| 取れない物 | アクセス数・CVR |

## 4. ワークフロー・流入経路・LINE公式

| 項目 | 内容 |
|---|---|
| ツール | `getCreatorScenarios`（`lifecycle=INACTIVE_ACTIVE`, `limit=100`。ACTIVE だけで絞る指定が無いため、返った全件から ACTIVE / INACTIVE を数える。`totalCount` が100を超えるときは `offset` を進めて全件取る。500件を超えるときは上位100件の内訳と書き、全体の内訳は「未集計」とする）、`getCreatorScenariosInflowActions`（`limit=20`）、`getCreatorLineChannels`、`getCreatorLineRichMenus` |
| 書く値 | WF件数（ACTIVE / INACTIVE 別。全件を数えた値）、表は更新日の新しい上位20件の id・名前・lifecycle・紐づく LINE公式 id。流入経路の件数と名前。LINE公式は接続中（`isConnected: true`）の件数と表示名だけ（総数は書かない。`getCreatorLineChannels` は件数を返さず、切断済みの旧行が多く、数え間違いが起きやすいため）。リッチメニュー件数 |
| 取れない物 | 実行履歴。ステージ別件数は `getCreatorScenario`（個別）が必要で、スナップショットでは取らない |

## 5. 売上（直近12ヶ月・月次）

| 項目 | 内容 |
|---|---|
| ツール | `getCreatorPaymentSalesOrdersSummary`（`unit=monthly`） |
| 期間 | 両端を含めて最長365日。`endDate` = 今日、`startDate` = 1年前の翌日（例: 今日 2026-10-02 なら `startDate=2025-10-03`）。同日にすると366日でエラー |
| 書く値 | 期間合計（`summary.totalAmount` / `totalCount`）、月別の `totalAmount` / `count`（返却順に依らず**古い順に並べ替えて**書く。申込のなかった月（0件の月）は返らないので、ある場合のみ「申込のあった月だけ」と表の下に注記する。0 円でも申込があれば行は返る）、商品別上位20 |
| 商品別の合算 | `rows[].plans` を `productId` で合算する。`productId` が `null`（旧サービス）のものは `serviceName` 単位で集計し、商品名列に「（旧サービス）{serviceName}」と書く。黙って落とさない |
| 定義 | 申込ベース（申込月に契約満額・税込・手数料控除前・キャンセルも残る）。詳細は sales-reporter の `references/metrics-definitions.md` |
| 取れない物 | 入金・返金・手取り・購入者ユニーク数 |

## 6. 顧客（サービス購入者）

| 項目 | 内容 |
|---|---|
| ツール | `postCustomersSearch`。全フィールド必須: `serviceIds: []`, `subscriptionIds: []`, `eventDateTime: null`, `searchQuery: ""`, `subscriptionState: null`, `paymentMethod: null`, `paymentType: null`, `paymentStatus: null`, `isEmailReceivingAllowed: null`, `tagIds: []`, `excludeTagIds: []`, `userIds: []`, `page: 1`, `limit: 1`。これを基準に、`subscriptionState: "active"` / `subscriptionState: "canceled"` / `isEmailReceivingAllowed: true` を1つずつ変えて計4回呼び、`totalCount` だけ使う |
| 書く値 | 顧客総数、サブスク継続中 / 解約済みの人数、メール受信許可の人数。顧客タグは `getCreatorCustomerTags` の件数と名前。継続中と解約済みは排他ではない（1人の顧客が両方に数えられうる）ので、合計が総数を超えても正常 |
| 注記 | 顧客は旧サービスの購入者も含む（商品一覧・購読者とは母集団も集計元も違う。購読者セクションの件数と一致しなくて正常）。雛形の顧客セクションの「母集団」の1行を必ず書く。購入日・最終購入日は取れない |

## 7. コンタクト（LINE友だち・メール購読者）

| 項目 | 内容 |
|---|---|
| ツール | `postCreatorContactsEmailSearch` / `postCreatorContactsLineSearch`。`keyword: ""`, `limit: 1`, `offset: 0`, `sort: "createdAtDesc"`。`filters` は親キー必須で、使わない条件は `null`（`tag` は配列ではなく `null`。空配列は 400）。`filteredCount` / `totalCount` だけ使う |
| メールの filters | 購読中: `{"contact": {"active": null, "subscribed": {"value": true, "operator": "eq"}}, "email": {"createdAt": null, "address": null}, "tag": null}`。有効全件: `subscribed` を `null` にしてもう1回 |
| LINEの filters | 未ブロック: `{"contact": {"active": null, "blocked": {"value": false, "operator": "eq"}}, "line": {"createdAt": null, "name": null}, "tag": null, "inflowAction": {"id": null}, "richMenu": {"id": null}}`。有効全件: `blocked` を `null` にしてもう1回。`lineChannelId` は指定しない（全アカウント横断） |
| 書く値 | メール: 有効件数・購読中件数。LINE: 有効件数・未ブロック件数。コンタクトタグは `getCreatorContactTags`（`limit=20`。`isContactCountsIncluded` は指定しない。人数集計は重く、件数と名前しか使わない）の件数と名前 |

## 8. 会員サイト・素材・ウェビナー

| 項目 | 内容 |
|---|---|
| ツール | `getCreatorMembershipSites` → サイトごとに `getCreatorMembershipSiteDashboardMembers`（`activeMemberCount`）。`getCreatorAutoWebinars`（`limit=20`）。`getCreatorMaterials` |
| 書く値 | 会員サイト件数（公開 / 非公開）、サイトごとの名前と会員数。ウェビナー件数とタイトル。素材は件数を書かず、「あり（件数は未集計）」または「なし」とだけ書く（理由は §4 の LINE公式と同じ。種別内訳も書かない） |
| 取れない物 | 視聴完了率・コンテンツ数（フォルダ取得が必要。スナップショットでは取らない） |

## 9. SNS（運用中のアカウント）

| 項目 | 内容 |
|---|---|
| 取得元 | ユーザー申告のみ。MCP にプロフィールリンクを読むツールが無く、公開ページの取得も不確実なため、Step 2.5 で「運用中の SNS の URL（あればフォロワー数）」を聞く |
| 書く値 | プラットフォーム（Instagram / X / YouTube / TikTok / Threads / Facebook / LINE公式 / note / 自社サイト 等）・URL またはハンドル・フォロワー数（申告があるもののみ「（申告）{n}」） |
| 取れない物 | 投稿頻度・エンゲージメント。フォロワー数は外部サイトに取りに行かない（鮮度を保証できない数値を黒板に載せない） |
| 回答が無いとき | 「取得失敗（未回答）」と書く。「運用していない」と答えたときだけ「該当なし」。続けてヒアリングを行うために聞かないときは、「未確認（後続のヒアリングで確認）」と書く |
