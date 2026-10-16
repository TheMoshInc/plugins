# MCP Tool リファレンス

applicant-reporter で使用する MCP ツールのパラメータ詳細。

> ツール名は OpenAPI の `operationId` で記載する。MCP クライアントが実際に提示するツール名にはサーバー識別子の接頭辞（例: `mcp__<server>__postCreatorProductApplicants`）が付くが、その形はユーザー環境により異なるため本ドキュメントでは付けない。

## 商品・プランの特定

**`getCreatorProducts`** — 商品一覧。返る `id` を `productId` に使う。

**`getCreatorProductPlans`** — 商品のプラン一覧。

```
pathParams: { productId }
```

返る `id` を `planIds` の要素に使う。

## 申込者一覧

**`postCreatorProductApplicants`** — 商品の申込者一覧を検索（単発・分割決済・銀行振込の申込。管理画面の「申込管理」タブ）

```
pathParams: { productId }
bodyParams: { planIds, keyword, paymentMethod, paymentPattern, approvedDateTime, limit, offset, order }
```

全項目必須。使わない条件は「絞り込まない」値を渡す。

| 項目 | 型 | 絞り込まない値 | 意味 |
|---|---|---|---|
| `planIds` | `string[]` | `[]` | 絞り込み対象のプラン ID。空配列は商品配下の全プラン。商品配下に無い ID を渡すとエラー |
| `keyword` | `string` | `""` | 名前 / メールアドレスの部分一致 |
| `paymentMethod` | `("card" \| "ticket" \| "cash" \| "bank-transfer")[]` | `[]` | 決済手段。**絞り込みとして効くのは `card` と `bank-transfer` のみ**。`ticket` / `cash` を単独で指定しても絞り込まれない。`card` は 0 円の申込を含む。`paymentPattern` と併用すると両者の AND |
| `paymentPattern` | 下表の値の配列 | `[]` | 決済方法 × 状態の組み合わせ。複数指定は OR |
| `approvedDateTime` | `{ startDateTime, endDateTime } \| null` | `null` | 申込日の範囲。各値は ISO 8601 日時か `null`。日本時間の日付単位で、終了日はその日の終わりまでを含む |
| `limit` | `integer` | — | 1〜1000。閲覧用途は 20〜50 |
| `offset` | `integer` | — | 0〜10000。`totalCount` に達するまで増やして続きを取る |
| `order` | `"createdDateTime" \| "-createdDateTime"` | — | 申込日の昇順 / 降順。通常は `"-createdDateTime"`（新しい順） |

`paymentPattern` の値:

| 値 | 決済 | 状態 |
|---|---|---|
| `oneTimeApproved` | 一括決済 | 決済済み |
| `oneTimeCanceled` | 一括決済 | キャンセル済み |
| `installmentActive` | 分割決済（分割入金） | 分割決済中 |
| `installmentFulfilled` | 分割決済（分割入金） | 分割決済完了 |
| `installmentPastDue` | 分割決済（分割入金） | 決済エラー |
| `installmentCanceled` | 分割決済（分割入金） | キャンセル済み |
| `cardInstallmentsApproved` | 分割決済（一括入金） | 決済済み |
| `cardInstallmentsCanceled` | 分割決済（一括入金） | キャンセル済み |
| `bankPending` | 銀行振込 | 振込待ち |
| `bankPaid` | 銀行振込 | 入金済み（反映待ち + 決済済み） |
| `bankRejected` | 銀行振込 | 振込期限切れ |

**`cardInstallmentsApproved` / `cardInstallmentsCanceled` を含めた場合は `limit` / `offset` が無視され、条件に合う全件が 1 回の応答で返る。** `totalCount` も返った件数と同じになる。件数が多い商品では応答が大きくなるため、先に `planIds` や `approvedDateTime` で対象を絞ってから指定する。

レスポンス: `{ applicants: [...], totalCount }`。各要素の読み方は [content-schema.md](content-schema.md)。

## 購読者一覧

**`postCreatorProductSubscribers`** — 商品のサブスク購読者一覧を検索（管理画面の「サブスク管理」タブ）

```
pathParams: { productId }
bodyParams: { planIds, keyword, statuses, startDateTime, endDateTime, limit, offset, order }
```

全項目必須。使わない条件は「絞り込まない」値を渡す。

| 項目 | 型 | 絞り込まない値 | 意味 |
|---|---|---|---|
| `planIds` | `string[]` | `[]` | 絞り込み対象のプラン ID。空配列は商品配下の全プラン |
| `keyword` | `string` | `""` | 名前 / メールアドレスの部分一致 |
| `statuses` | `("trialing" \| "active" \| "canceled" \| "pastDue")[]` | `[]` | 空配列は解約済みを含む全件。継続中は `["active", "pastDue"]`、解約済みは `["canceled"]`。`trialing` はどのレコードにも付かない |
| `startDateTime` / `endDateTime` | `string \| null` | `null` | 購読開始日の範囲（ISO 8601 日時） |
| `limit` | `integer` | — | 1〜1000。閲覧用途は 20〜50 |
| `offset` | `integer` | — | 0〜10000 |
| `order` | `"id" \| "-id" \| "startedDateTime" \| "-startedDateTime"` | — | 通常は `"-startedDateTime"`（開始日の新しい順） |

レスポンス: `{ subscribers: [...], totalCount }`。各要素の読み方は [content-schema.md](content-schema.md)。

## 2 つのツールの使い分け

| | `postCreatorProductApplicants` | `postCreatorProductSubscribers` |
|---|---|---|
| 対象 | 単発・分割決済・銀行振込の申込 | サブスクリプションの購読 |
| 1 行の単位 | 申込 1 件 | 購読 1 件 |
| 期間の基準 | 申込日（`approvedDateTime`） | 購読開始日（`startDateTime` / `endDateTime`） |
| 状態の絞り込み | `paymentPattern`（決済方法 × 状態） | `statuses` |

商品にサブスクのプランと単発のプランが両方あるときは両方を呼ぶ。
