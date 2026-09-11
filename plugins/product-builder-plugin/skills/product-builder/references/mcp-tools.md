# 使用する MCP ツール

ツール名は OpenAPI の operationId で記す。実際のツール名にはサーバー識別子の接頭辞（例: `mcp__<server>__postCreatorProduct`）が付くが、その形は環境により異なるためここでは付けない。

## 書き込み系

### `postCreatorProduct` — 商品を作成

| パラメータ | 必須 | 制約 |
|---|---|---|
| `name` | ○ | 1〜100 文字 |
| `description` | ○ | 1〜100,000 文字（**表示テキストは 10,000 文字まで**） |
| `precautions` | ○ | 同上 |
| `cancellationPolicy` | ○ | 同上 |
| `imageIds` | — | 文字列の配列。**最大 7 件** |

作成された商品は**非公開**で始まる。作成時に公開状態は指定できない。

返るのは `productId` のみ。内容を確認するには `getCreatorProduct` で読み戻す。

### `patchCreatorProduct` — 商品を更新

`productId`（パス）で対象を指定する。body の項目はすべて任意で、**送ったものだけが更新される**。

| パラメータ | 制約 | 注意 |
|---|---|---|
| `name` | 1〜100 文字 | |
| `description` | 1〜100,000 文字（表示テキストは 10,000 文字まで） | |
| `precautions` | 同上 | |
| `cancellationPolicy` | 同上 | |
| `imageIds` | 文字列の配列。最大 7 件 | **配列まるごとの置き換え**。全件送る |
| `publishingStatus` | `PUBLIC` / `LIMITED` / `PRIVATE` | 指定した場合のみ更新 |
| `steps` | 下表 | **配列まるごとの置き換え**。全件送る |

`steps[]` の各要素は 4 項目すべてが必須:

| 項目 | 必須 | 制約 |
|---|---|---|
| `orderNum` | ○ | 0 以上の整数。**同じ値を 2 つ以上入れられない** |
| `title` | ○ | 1〜100 文字 |
| `description` | ○ | 1〜1,000 文字 |
| `imageId` | ○ | 文字列または `null`。**省略はできない**（画像が無いなら `null`） |

### `deleteCreatorProduct` — 商品を削除

`productId`（パス）で対象を指定する。**取り消せない。** 紐づくプランも同時に削除される。

サブスクを購読中・支払い延滞中のゲスト、あるいは購入済みのゲストがいるプランを含む場合は削除できない。事前に `getCreatorProduct` の `isDeletable` で判定できる。

### `postCreatorProductPlan` — プランを作成

`productId`（パス。数値文字列）で対象の商品を指定する。body の必須・任意は下表のとおり。値の意味と既定値は [content-schema.md](content-schema.md) の「プランの項目」を参照する。

| パラメータ | 必須 | 制約 |
|---|---|---|
| `name` | ○ | 1〜50 文字 |
| `description` | ○ | 10,000 文字まで |
| `orderNum` | ○ | 0 以上の整数。**作成時は自動採番されるため送った値は無視される** |
| `publishingStatus` | ○ | `PUBLIC` / `LIMITED` / `PRIVATE` |
| `capacity` | ○ | 0〜1,000,000。**`0` は定員なし** |
| `showRemaining` | ○ | 残席数を公開ページに出すか |
| `remainingThreshold` | ○ | 0 以上。残席がこの数以下になったら表示する |
| `price` | ○ | `{ amount, currency }`。`currency` は `JPY` |
| `discountPrice` | ○ | 割引前価格。無いなら `null`。1 円以上かつ**価格より大きい** |
| `paymentMethods` | ○ | `{ method, isEnabled }` の配列。`method` は `CARD` / `BANK_TRANSFER` |
| `billingCycle` | ○ | `ONE_TIME` / `MONTHLY` / `YEARLY` |
| `benefits` | ○ | 文字列の配列。**最大 10 件・各 50 文字**。買い切りでは `[]` |
| `installmentType` | ○ | `LUMP_SUM` / `INSTALLMENT_LUMP_DEPOSIT` / `INSTALLMENT` |
| `installmentIterations` | ○ | `{ count, price }` の配列。`count` は 2 以上、`price` は 1 円以上または `null`。**`LUMP_SUM` では空配列**、`INSTALLMENT` では 1 件以上必須。`INSTALLMENT_LUMP_DEPOSIT` は任意だが入れれば回数として使われる |
| `applicationStartDateTime` | ○ | 販売受付開始日時または `null`。**サブスクリプションでは常に `null`** |
| `applicationEndDateTime` | ○ | 販売受付終了日時または `null`。同上 |
| `initialPrice` | — | 初回価格（1 円以上）。fincode のクリエイターのサブスクリプションでのみ使う。`null` で通常価格と同額 |
| `iterations` | — | 課金回数（初回を含む合計）。**1〜12**。サブスクリプションかつ Stripe のクリエイターのみ。`null` で無制限 |
| `billingStartType` | — | `immediate` / `absolute` / `relative`。省略時は `immediate`。サブスクリプションのみ |
| `billingStartAbsoluteDate` | — | `absolute` のときの課金開始日。それ以外は `null` |
| `billingStartRelativeOffsetDays` | — | `relative` のときの日数（0〜365）。それ以外は `0` |
| `applicationPeriods` | — | 申込期間の配列。**買い切りでは使えない**。詳細は [content-schema.md](content-schema.md) |

返るのは `productPlanId` のみ。内容を確認するには `getCreatorProductPlan` で読み戻す。

**同じ body を再送すると別のプランがもう 1 つ作られる。** 再試行の前に一覧で重複を確認する。

### `patchCreatorProductPlan` — プランを更新

`productId` / `productPlanId`（ともにパス。数値文字列）で対象を指定する。

**部分更新には対応していない。** body は `initialPrice` / `iterations` / `discountPrice` / `applicationPeriods` 以外すべて必須で、作成時の項目に `isSuspended`（販売受付だけ止めるフラグ）が加わり、`billingStartType` / `billingStartAbsoluteDate` / `billingStartRelativeOffsetDays` は**項目自体が無い**（課金開始日は作成時のまま変わらない）。

送る前に `getCreatorProductPlan` で現在値を読み、変えない項目もそのまま含める。

拒否される変更:

| 変更 | 結果 |
|---|---|
| `billingCycle` / `iterations` / `initialPrice` を現在値と違う値にする | 拒否される。新しいプランを作る |
| 公開中または購入者がいるサブスクリプションプランの `name` / `price` を変える | 拒否される。`isNameEditable` / `isPriceEditable` で事前判定できる |
| `capacity` を現在の購読者数より小さくする | 拒否される |

### `patchCreatorProductProductPlansReorder` — プランの表示順を一括更新

| パラメータ | 必須 | 制約 |
|---|---|---|
| `productId`（パス） | ○ | **数値**（1 以上）。他のプラン操作は文字列なので注意 |
| `productPlanIds` | ○ | **数値**のプラン ID を並べたい順に入れた配列。1 件以上 |

非公開のプランも並び替えの対象。**リクエストに含まれないプランは、既存の相対順のまま末尾に回る**ため、`getCreatorProductPlans` で取得した全件を送る。

### `deleteCreatorProductPlan` — プランを削除

`productId` / `productPlanId`（ともにパス）で対象を指定する。**取り消せない。**

サブスクリプションを購読中のゲスト、購入済みのゲスト、分割払いの支払いが完了していないゲストがいる場合は削除できない。事前に `getCreatorProductPlan` の `isDeletable` で判定できる。

## 読み取り系（確認・読み戻しに使う）

### `getCreatorProducts` — 商品一覧を取得

| パラメータ | 既定 | 制約 |
|---|---|---|
| `limit` | 20 | 1〜100 |
| `offset` | 0 | 0 以上 |
| `status` | 省略時は全件 | `PUBLIC` / `LIMITED` / `PRIVATE` |

`totalCount` は**総件数**。取得済み件数より多ければ `offset` をずらして続きを取得する。

### `getCreatorProduct` — 商品詳細を取得

`productId`（パス。数値文字列）で指定する。自分が所有する商品のみ取得でき、他のユーザーの商品を指定すると「見つからない」旨のエラーになる。

### `getCreatorProductPlans` — 商品プラン一覧を取得

`productId`（パス）で指定する。**商品を公開できるかの判定に使う**（公開中のプランが 1 つ以上必要）。

この一覧は**ページングされない**。`limit` / `offset` は現状無視され、`totalCount` は返した件数を表す。

自分の商品でない・存在しない `productId` でもエラーにならず**空配列**が返る。0 件のときは「プランが無い」と断定せず、商品の特定が正しいかを先に確認する。

### `getCreatorProductPlan` — 商品プラン詳細を取得

`productId` / `productPlanId`（ともにパス。数値文字列）で指定する。自分が所有する商品のプランのみ取得でき、他のユーザーのプランを指定すると「見つからない」旨のエラーになる。

**更新の前には必ずこれを呼ぶ**（部分更新に対応していないため、現在値が要る）。`isNameEditable` / `isPriceEditable` / `isDeletable` もここで読める。

## このスキルで扱えない操作

| 操作 | 案内先 |
|---|---|
| 商品画像のアップロード・一覧取得 | 管理画面（画像 ID はユーザーから受け取る） |
| 事業者情報の登録・確認 | 管理画面 |
| クーポン・特典コンテンツ・収益分配の設定 | 管理画面 |
| 購入者・売上の確認 | `sales-reporter` スキル |
