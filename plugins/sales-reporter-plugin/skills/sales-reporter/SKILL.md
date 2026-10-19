---
name: sales-reporter
description: MOSH の売上を MCP 経由で確認・集計・分析するスキル。「今月の売上」「先月いくら売れた」「商品別・プラン別の売上」「売上一覧を見せて」「月次会議の資料を作って」「このキャンペーン期間の売上」「売上推移」「決済の明細を見たい」「今月何人が買ってくれた？」「ユニーク購入者数」「申込者数」「◯◯さんの売上はいくら？」「分割払いだけの売上」「一括払いの売上」「sales-reporter」など、売上・注文・決済・購入者数の照会リクエストで使用する。`getCreatorPaymentSalesOrdersSummary` / `getCreatorPaymentSalesOrders` / `postCreatorPaymentSalesAggregate` ツールを使う場合は必ず本スキルを起動し、数値定義の提示ルールを適用する。
---

# MOSH 売上レポーター

## Overview

MCP ツール（`getCreatorPaymentSalesOrdersSummary` / `getCreatorPaymentSalesOrders` / `postCreatorPaymentSalesAggregate`）を使って、MOSH クリエイターの売上を確認・集計する。金額は MOSH 管理画面の「売上一覧」ページと同じ申込ベースの定義で返る。

数値の正確な定義は [references/metrics-definitions.md](references/metrics-definitions.md)、用途別の手順は [references/playbooks.md](references/playbooks.md) を参照する。

> **ツール名表記について**: 本スキル内では MCP ツールを OpenAPI の `operationId`（例: `getCreatorPaymentSalesOrders`）で参照する。MCP クライアントが実際に提示するツール名にはサーバー識別子のプレフィックス（例: `mcp__<server>__getCreatorPaymentSalesOrders`）が付くが、その形はユーザー環境により異なるため、本スキルでは付けない。
>
> **呼称について**: プロダクト上の呼称は「商品」「プラン」。API のキー名では、プランに対応するサービスIDが `moshServiceId` になっている。ユーザーへの提示ではプロダクト上の呼称を使う。
>
> **名前の出し分け**: 明細・サマリーの内訳とも `productName`（商品名）/ `planName`（プラン名）/ `serviceName` の 3 つが返る。ユーザーには「商品名 - プラン名」の形で示し、**`planName` が null のときは `serviceName` にフォールバックする**。`serviceName` は必ず入っており、`productName` / `planName` は対応する商品プランが無いと null になる。`productName` も null なら `serviceName` だけを示す。`postCreatorPaymentSalesAggregate` の内訳（`rows[].services`）は `serviceId` / `serviceName` だけなので、`serviceName` をそのまま示す。

## When to use

- 売上金額・件数の照会（「今月いくら売れた？」）
- 期間比較・売上推移（「先月と比べて」「3ヶ月の推移」）
- 商品別・プラン別の売上内訳
- 個別の注文・決済の確認
- 購入者の人数（ユニーク購入者数・申込者数）
- 特定のゲスト・支払いタイプ（一括 / 分割）に絞った売上
- 月次会議資料・施策振り返りの作成

## When NOT to use

- **入金・振込の確認**（売上とは別概念。別ツール・別スキルの領域）
- **返金・キャンセル操作などの更新系**（提供していない。参照専用）
- **顧客ごとの売上ランキング・LTV の一覧**（提供していない。特定のゲストの合計だけは `postCreatorPaymentSalesAggregate` で取れる）
- **顧客の検索・申込履歴**は `customer-manager`。LP 作成は `lp-builder`、ワークフロー作成は `workflow-builder`
- **商品の販売状況（今月の決済額・サブスク購読者数）・新規申込人数の推移／当月値**は `getCreatorProductSalesStatus` / `getCreatorProductsApplications` / `getCreatorProductsApplicationsSummary` の領域で、売上とは数え方が違う。本スキルの数値と混ぜない

## 3 つのツールの使い分け

| | サマリー (`getCreatorPaymentSalesOrdersSummary`) | 明細 (`getCreatorPaymentSalesOrders`) | 絞り込み集計 (`postCreatorPaymentSalesAggregate`) |
|---|---|---|---|
| 返すもの | 期間の合計金額・件数と、期間別・プラン別の内訳 | 注文 1 件ごとの行 | 期間の合計金額・件数・**申込者数**と、期間別・プラン別の内訳 |
| 数える単位 | 契約単位（分割払いは 1 契約を 1 件） | 注文単位（分割払いは支払回ごとに 1 行） | 契約単位＋ユニーク購入者数 |
| 主な用途 | 「いくら売れたか」「何が効いたか」 | 個別の注文を確認する | 申込者数が要る／ゲスト・支払いタイプで絞る |

**選び方: 基本はサマリー。** 申込者数（ユニーク購入者数）が要るとき、またはゲスト・支払いタイプ（一括 / MOSH の分割払い / カード会社の分割払い）で絞るときだけ `postCreatorPaymentSalesAggregate` を使う。個別の注文を見るときだけ明細を使う。

**各ツールの件数・金額は互いに一致するとは限らない。仕様であって不整合ではない。** 理由は [references/metrics-definitions.md](references/metrics-definitions.md) にある。

サマリーは `unit`（`monthly` / `daily`）を指定したときだけ期間別・プラン別の内訳（`rows`）を返す。**商品別・プラン別の内訳が必要なときは `unit` を必ず指定する。** 合計金額だけでよいときは指定しない。

## 進め方の原則

1. **最初に期間を確定する。** 「今月」「先月」は日本時間（JST）の暦月。「最近」「このあいだ」のような曖昧な指定は推測せず、ユーザーに確認する
2. **サマリーから始める。** 明細から始めない。申込者数・ゲスト・支払いタイプが要る依頼だけは `postCreatorPaymentSalesAggregate` から始めてよい（手順は [references/playbooks.md](references/playbooks.md) の 6〜8）
3. **明細を取るのは、絞り込んだうえで個別の注文を見る必要があるときだけ。** `totalCount` が大きい場合は全件取得せず、件数を伝えて取得するか確認する
4. **商品名から ID を解決するときは `getCreatorProducts`（商品一覧）を使う。** 返る `id` はそのまま `productId` に使える
5. **プラン名から ID を解決するときは、期間サマリーの内訳（`rows[].plans`）の名前から特定する。** プラン単位の検索ツールは無い。ここで得た `moshServiceId` は `postCreatorPaymentSalesAggregate` の `serviceIds` にもそのまま使える

## 提示ルール

### 1. 金額には必ず定義を添える。ただしサマリーと明細で言い方を変える

金額は円・カンマ区切りで示し、定義を添える。**サマリーと明細では金額の意味が違うので、同じ説明を使い回さない。**

サマリーの合計は申込ベース（申込月に契約満額を計上）。

- ✅ 「2026年7月の売上は 580,000円 です（申込ベース・税込・手数料控除前）」
- ❌ 「7月の売上は 580000 です」

明細の各行は注文単位で、分割払いなら**その支払回の金額**。契約満額ではない。

- ✅ 「この行は 3回払いの2回目で 33,000円 です（税込・手数料控除前）」
- ❌ 「この注文は申込月に契約満額を計上した 33,000円 です」

`postCreatorPaymentSalesAggregate` の金額もサマリーと同じ言い方（申込ベース・税込・手数料控除前）を添える。申込者数は「申込者数（ユニーク購入者数）」と書き、件数（契約数）と取り違えない。

### 2. 返却値を足し合わせて合計を出さない

合計はサマリー（またはその依頼で使った `postCreatorPaymentSalesAggregate`）の返却値をそのまま使い、再計算しない（例外は 365 日を超える期間を分けて取った**金額・件数**の合算だけ）。明細とサマリーは対象範囲が異なるため、明細の合計はサマリーと一致しない。**申込者数は特に足さない**: 期間別の行（`rows[].applicantsCount`）の合計や、365 日を超える期間を分けて取った申込者数の合計は、同じ人を重複して数えるため全体の申込者数にならない。

### 3. 件数の差異を聞かれたら仕様として説明する

サマリーの件数は契約単位、明細の行数は注文単位。両者が一致しないのは正常。差異を指摘されたら [references/metrics-definitions.md](references/metrics-definitions.md) の内容で説明する。

### 4. 取得できないものは推測しない

**返金額・手数料控除後の金額は取得できない。** 聞かれたら「このツールでは取得できません」と伝え、管理画面へ誘導する。数字を推測して答えない。

### 5. ゲストの名前・メールアドレスは返らない。特定ゲストの売上はキーワード絞り込みで取る

ゲストの名前・メールアドレスを返さない（明細は `guestId` のみ）。「誰が買ったか」を名前で知りたい場合は `applicant-reporter`（申込者・購読者一覧）を使う。

特定のゲストの売上は `postCreatorPaymentSalesAggregate` の `guestKeyword`（名前・メールアドレスの部分一致）で取れる。ただし**一致したゲスト全員の合算**で、1 人に限定できない。その人の売上として伝える前に `customer-manager` の `postCustomersSearch` で同じキーワードに一致する顧客を確認し、複数いればその旨を添える。検索に使った名前・メールアドレスは、依頼された用途以外に出力・転記しない。手順は [references/playbooks.md](references/playbooks.md) の 7。

### 6. エラーをそのまま出さない

HTTP ステータスコードや API のエラー文をユーザー向け文面に出さない。「期間は最長 1 年までです」のように、ユーザーが次に取るべき行動がわかる日本語に変換する。

## 制約（クエリを組む前に確認する）

| 制約 | 内容 |
|---|---|
| 期間の上限 | 開始日と終了日の**両方を含めて 365 日**まで。「過去 1 年」は `startDate` を 1 年前の**翌日**にする（例: `endDate=2026-08-19` なら `startDate=2025-08-20`）。超える期間は分けて取得する（合算の可否は提示ルール 2） |
| 期間の基準 | すべて**申込日**基準。決済完了日での絞り込みはできない |
| 絞り込みの併用 | `moshServiceIds`（カンマ区切り）と `productId` は**同時に指定できない** |
| 支払い方法の絞り込み | `card` / `bankTransfer` / `cash` の 3 つのみ。明細には PayPay や LINE Pay なども現れるが、それらでは絞り込めない |
| 決済ステータス | 絞り込めない。明細の `paymentStatus` を見て判断する |
| 明細の取得件数 | `limit` は**最大 100・既定 20**。まとめて取得すると決めたら `limit=100` を指定する（既定のままだと往復回数が 5 倍になる） |
| 内訳の並び順 | サマリー・`postCreatorPaymentSalesAggregate` とも `rows` は新しい期間が先頭で返る仕様だが、返却順には依存しない。時系列で見せるときは `period` ラベル（日次 `YYYY-MM-DD`・月次 `YYYY-MM-01`）で古い順に並べ替える |

### `postCreatorPaymentSalesAggregate` だけの差分

サマリーと同じ制約（期間 365 日・申込日基準・支払い方法 3 種）に加えて、次が違う。完成形の呼び出し例は [references/playbooks.md](references/playbooks.md) の「`postCreatorPaymentSalesAggregate` の呼び出し形」。

| 項目 | 内容 |
|---|---|
| 呼び方 | POST。引数はクエリではなく body（MCP では `bodyParams`）に入れる |
| 必須項目 | `startDate` / `endDate` / `period` / `serviceIds` / `paymentMethod` / `paymentType` / `guestKeyword` の **7 つすべて**。絞らない項目も省略せず、`serviceIds: []`・`paymentMethod: null`・`paymentType: null`・`guestKeyword: ""` を入れる |
| 期間内訳 | `period`（`daily` / `monthly`）は必須。サマリーの `unit` と違い省略できない。`rows` は**申込があった期間だけ**返る（0 の期間は行が無い） |
| プランの絞り込み | `serviceIds` は**文字列の配列**（サマリーの `moshServiceIds` はカンマ区切り文字列）。値はサマリーの `moshServiceId` と同じ。`productId` での絞り込みは無いので、商品単位ならその商品のプランの `moshServiceId` を全部並べる |
| 支払いタイプ | `paymentType`: `oneTime`（一括）/ `moshInstallment`（MOSH の分割払い）/ `cardInstallment`（カード会社の分割払い）/ `null`（絞らない） |
| 該当なし | エラーにならない。金額・件数が 0、`rows` が空配列で返る。「該当する申込はありません」と伝える |

## References

| ファイル | いつ読むか |
|---|---|
| [references/metrics-definitions.md](references/metrics-definitions.md) | 数値を提示する前に定義を確かめるとき、「管理画面と数が違う」「件数と人数が合わない」と言われたとき（申込ベース・金額・件数・申込者数・`paymentStatus`・取得できないもの） |
| [references/playbooks.md](references/playbooks.md) | 依頼に着手するとき（月次資料・振り返り・明細・推移・差異説明・申込者数・特定ゲスト・支払いタイプ別の手順と、`postCreatorPaymentSalesAggregate` の呼び出し形） |
