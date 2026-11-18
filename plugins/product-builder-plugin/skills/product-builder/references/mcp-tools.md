# 使用する MCP ツール

**いつ読むか**: 商品・プラン本体（作成・更新・並び替え・削除・取得）と画像アップロードのツールを呼ぶ前に、パラメータ・必須項目・返却値を確かめるとき。プランの設定・複製・クーポンのツールは各領域ファイルに置いてあり、このファイルを読む必要はない（末尾の対応表）。

ツール名は OpenAPI の operationId で記す。実際のツール名にはサーバー識別子の接頭辞（例: `mcp__<server>__postCreatorProduct`）が付くが、その形は環境により異なるためここでは付けない。

## 書き込み系

項目の範囲・組み合わせ・既定値・**拒否されたときのメッセージ**の正本は [content-schema.md](content-schema.md)。この節は送るパラメータの一覧で、拒否条件は再掲しない。旧サービスの扱いは [content-schema.md](content-schema.md) の「旧サービス（isLegacyService: true）」。

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

**同じ body を再送すると別の商品がもう 1 つ作られる。** 再試行の前に一覧で重複を確認する。

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
| `discountPrice` | ○ | 割引前価格。無いなら `null`。1 円以上かつ**価格より大きい**。**買い切り（`ONE_TIME`）のみ**。サブスクリプションでは `null` |
| `paymentMethods` | ○ | `{ method, isEnabled }` の配列。`method` は `CARD` / `BANK_TRANSFER`。サブスクリプションでは `BANK_TRANSFER` を有効にできない |
| `bankTransferDeadlineDays` | ○ | 銀行振込の振込期限（申込からの日数）。1〜99 または `null`。**`null` は標準の 15 日**。銀行振込が無効なプラン（サブスクリプションは常に無効）では `null` を送る。日数を送ると拒否される |
| `billingCycle` | ○ | `ONE_TIME` / `MONTHLY` / `YEARLY` |
| `benefits` | ○ | 文字列の配列。**最大 10 件・各 50 文字**。買い切りでは `[]` |
| `installmentType` | ○ | `LUMP_SUM` / `INSTALLMENT_LUMP_DEPOSIT` / `INSTALLMENT` |
| `installmentIterations` | ○ | `{ count, price }` の配列。**`LUMP_SUM` では空配列**、`INSTALLMENT` では 1 件以上必須。`INSTALLMENT_LUMP_DEPOSIT` では送っても保存されない（[content-schema.md](content-schema.md) の「分割払い」） |
| `applicationStartDateTime` | ○ | 販売受付開始日時または `null`。**サブスクリプションでは常に `null`** |
| `applicationEndDateTime` | ○ | 販売受付終了日時または `null`。同上 |
| `initialPrice` | — | 初回価格（1 円以上）。サブスクリプション × fincode のクリエイター限定。`null` で通常価格と同額。Stripe では無視されるので読み戻して確かめる（[content-schema.md](content-schema.md) の「初回価格」） |
| `iterations` | — | 課金回数（初回を含む合計）。**1〜12**。サブスクリプションかつ Stripe のクリエイターのみ。`null` で無制限 |
| `billingStartType` | — | `immediate` / `absolute` / `relative`。省略時は `immediate`。サブスクリプションのみ |
| `billingStartAbsoluteDate` | — | `absolute` のときの課金開始日。それ以外は `null` |
| `billingStartRelativeOffsetDays` | — | `relative` のときの日数（1〜365）。それ以外は `0` |
| `applicationPeriods` | — | 申込期間の配列。**買い切りでは使えない**。詳細は [content-schema.md](content-schema.md) |

返るのは `productPlanId` のみ。内容を確認するには `getCreatorProductPlan` で読み戻す。

**同じ body を再送すると別のプランがもう 1 つ作られる。** 再試行の前に一覧で重複を確認する。

### `patchCreatorProductPlan` — プランを更新

`productId` / `productPlanId`（ともにパス。数値文字列）で対象を指定する。

**部分更新には対応していない。** body は `initialPrice` / `iterations` / `discountPrice` / `applicationPeriods` / `bankTransferDeadlineDays` / 課金開始日の 3 項目以外すべて必須で、作成時の項目に `isSuspended`（販売受付だけ止めるフラグ）が加わる。

- **課金開始日の 3 項目は、変えないなら省略する**（省略で現在値を維持）。取得値は日数が `null` で返ることがあり、そのまま送り返すと形式エラーになる。変更できるのは `isBillingStartEditable` が true（非公開かつ購読者ゼロ）のプランだけ（[content-schema.md](content-schema.md) の「課金開始日」）
- **`discountPrice` は省略・現在値の送り返しでも `null` に置き換わったり、価格を上げると拒否されたりする**（[content-schema.md](content-schema.md) の「割引前の価格と価格の更新」）

送る前に `getCreatorProductPlan`（詳細）か `getCreatorProductPlans`（一覧）で同じプランの現在値を読み、変えない項目もそのまま含める（一覧と詳細は同じ項目を返す）。

`bankTransferDeadlineDays` は省略すると現在値のまま、`null` を送ると標準の 15 日に戻る。銀行振込を無効にする更新では、残っていた期限も消える。

**エラーで終わっても途中まで反映されていることがある**（価格は元に戻るが、プラン名・公開状態・定員は新しい値で残りうる）。エラーのあとは読み戻して確かめる。拒否される変更（変更不可の項目・名前と価格のロック・定員・振込期限など）は [content-schema.md](content-schema.md) の「拒否されたときのメッセージ」。

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
| `searchQuery` | 省略時は絞らない | 商品名・プラン名の部分一致。最大 200 文字。商品が多いときの対象の特定に使う |

`totalCount` は**総件数**。取得済み件数より多ければ `offset` をずらして続きを取得する。

### `getCreatorProduct` — 商品詳細を取得

`productId`（パス。数値文字列）で指定する。自分が所有する商品のみ取得でき、他のユーザーの商品を指定すると「見つからない」旨のエラーになる。

### `getCreatorProductPlans` — 商品プラン一覧を取得

`productId`（パス）で指定する。**商品を公開できるかの判定に使う**（公開中のプランが 1 つ以上必要）。

この一覧は**ページングされない**。`limit` / `offset` は現状無視され、`totalCount` は返した件数を表す。

自分の商品でない・存在しない `productId` でもエラーにならず**空配列**が返る。0 件のときは「プランが無い」と断定せず、商品の特定が正しいかを先に確認する。

### `getCreatorProductPlan` — 商品プラン詳細を取得

`productId` / `productPlanId`（ともにパス。数値文字列）で指定する。自分が所有する商品のプランのみ取得でき、他のユーザーのプランを指定すると「見つからない」旨のエラーになる。

**更新の前には必ずこれを呼ぶ**（部分更新に対応していないため、現在値が要る）。`isNameEditable` / `isPriceEditable` / `isDeletable` もここで読める。`membershipSites` にプランに紐づく会員サイトの名前と ID が入る。

## 画像のアップロード

### `postMeMediaUploadUrls` — 画像のアップロード先を発行

商品画像（`imageIds` / `steps[].imageId`）とサンクスページのバナー（`bannerMediaId`）に使う画像を、手元のファイルからアップロードするときに使う。返る `moshMediaId` がそのまま画像 ID になる。パラメータ・PUT の方法・失敗時の対処は [media-upload.md](media-upload.md) にまとめてある。

## プランの設定・複製・クーポンのツール

次のツールのパラメータ・応答・注意は、手順と同じ領域ファイルの「使用するツール」（複製は「リクエストと返却」）と各「項目」節にまとめてある（このファイルには置かない）。

| ツール | 正本 |
|---|---|
| `getCreatorProductPlanOfferedContents` / `postCreatorProductPlanOfferedContent` / `deleteCreatorProductPlanOfferedContent` / `getCreatorMembershipSites` / `getCreatorMembershipSite` | [membership-link.md](membership-link.md) |
| `getCreatorMembershipSiteProductPlanLicenseSetting` / `putCreatorMembershipSiteProductPlanLicenseSetting` | [membership-link.md](membership-link.md) |
| `getCreatorProductPlanSettingsThanks` / `patchCreatorProductPlanSettingsThanks` / `getCreatorProductPlanSettingsRedirect` / `patchCreatorProductPlanSettingsRedirect` / `deleteCreatorProductPlanSettingsRedirect` / `getCreatorProductPlanSettingsReviewReport` / `patchCreatorProductPlanSettingsReviewReport` | [post-purchase-settings.md](post-purchase-settings.md) |
| `postCreatorProductPlanDuplicate` | [plan-duplicate.md](plan-duplicate.md) |
| `getCreatorProductPlanCoupons` / `postCreatorProductPlanCoupon` / `getCreatorProductPlanCoupon` / `patchCreatorProductPlanCoupon` / `deleteCreatorProductPlanCoupon` | [coupons.md](coupons.md) |

## このスキルで扱えない操作

| 操作 | 案内先 |
|---|---|
| アップロード済みの画像の一覧・検索 | できない。いま付いている画像は `getCreatorProduct` / `getCreatorProductPlanSettingsThanks` で読める。それ以外はユーザーから ID を受け取るか、手元のファイルをアップロードし直す（[media-upload.md](media-upload.md)） |
| 事業者情報の登録・確認 | 管理画面 |
| 予約メニューをプランに紐付ける | 管理画面（解除はこのスキルで可能） |
| 商品ごとの複製（配下のプランをまとめて別の商品として複製する） | 管理画面（プラン 1 件の複製はこのスキルで可能） |
| クーポンの内容（コード・割引・枚数・期限・回数設定）の変更 | できない。新しいクーポンを作成し、古いクーポンを無効化する |
| 収益分配・決済リンクの設定 | 管理画面 |
| 会員サイトそのもの（フォルダ・コンテンツ・タグ・閲覧順の固定などのサイト設定） | `membership-site-builder` スキル |
| 購入者・売上の確認 | `sales-reporter` スキル |
