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

**更新の前には必ずこれを呼ぶ**（部分更新に対応していないため、現在値が要る）。`isNameEditable` / `isPriceEditable` / `isDeletable` もここで読める。`membershipSites` にプランに紐づく会員サイトの名前と ID が入る。

## 提供コンテンツ（会員サイト・予約メニュー）

プランで提供する会員サイト・予約メニューの紐付け。`productId` / `productPlanId` はいずれもパスで、**数値文字列**。

### `getCreatorProductPlanOfferedContents` — プランに紐づく提供コンテンツ一覧を取得

会員サイトと予約メニューが種別の違う提供コンテンツとして**同じ配列**に並ぶ。ページングされない。

レスポンス: `{ offeredContents: [...], totalCount, isUsingLegacySite }`。各要素の項目は [content-schema.md](content-schema.md) の「提供コンテンツの項目」。`isUsingLegacySite` が `true` なら、いま申込者向けサイト（旧会員サイト）を使っている。

紐付けが無ければ空配列。書き込みの前後でこれを読み、**変更前と変更後を会員サイト名・予約メニューのタイトルで提示する**。

### `postCreatorProductPlanOfferedContent` — プランに会員サイトを紐付け

| パラメータ | 必須 | 制約 |
|---|---|---|
| `useLegacyMembershipSite` | ○ | `true` で申込者向けサイト（旧会員サイト）を使う |
| `membershipSiteId` | ○ | 紐付ける会員サイトの ID（**数値**）または `null`。**省略はできない** |

2 つの組み合わせで 3 通りの操作になる（詳細は [content-schema.md](content-schema.md) の「会員サイトの紐付けリクエスト」）。

| `useLegacyMembershipSite` | `membershipSiteId` | 操作 |
|---|---|---|
| `false` | 数値 | その会員サイトを紐付ける（変更も同じ）。**申込者向けサイトは非公開になる** |
| `true` | `null` | 申込者向けサイト（旧会員サイト）を使う。紐付いている会員サイトは外れない。非公開にした申込者向けサイトは自動では公開に戻らない |
| `false` | `null` | 会員サイトを使わない。会員サイトの紐付けをすべて外し、**申込者向けサイトを非公開にする** |

`true` と数値を同時に送ると拒否される。

拒否される操作:

| 操作 | 結果 |
|---|---|
| 購入者がいるプランで、別の会員サイトへ変える・申込者向けサイトへ切り替える・会員サイトを使わない設定にする | 拒否される。`getCreatorProductPlan` の `isDeletable: false` なら呼ばずに伝える |
| 購入者がいるプランで、申込者向けサイトから会員サイトへ変える | 同上 |
| 自分の会員サイトでない `membershipSiteId` | 「見つからない」旨のエラー |
| 申込者向けサイトが用意されていないプランで `useLegacyMembershipSite: true` | 「見つからない」旨のエラー |

購入者の有無で止まるのは**すでに紐付いているものの変更・解除**だけ。何も紐付いていないプランへの初回の紐付けは、購入者がいてもできる。

**同じ会員サイトを再指定すると「変更がないためスキップしました」で成功する。** 会員サイトを新しく紐付けたときは、そのプランのライセンスが「全フォルダ閲覧可・今後追加分も含む・無期限」で自動作成される。予約メニューの紐付けは、このツールでは変わらない。

### `deleteCreatorProductPlanOfferedContent` — プランから提供コンテンツの紐付けを解除

`productId` / `productPlanId` / `offeredContentId`（いずれもパス。数値文字列）で対象を指定する。`offeredContentId` は一覧の `id`。会員サイトも予約メニューも同じツールで外す。

拒否される操作:

| 操作 | 結果 |
|---|---|
| 申込者向けサイト（旧会員サイト）の紐付けを外す | 拒否される。`postCreatorProductPlanOfferedContent` で「会員サイトを使わない」を選ぶ |
| 購入者がいるプランの紐付けを外す | 拒否される（会員サイト・予約メニューとも） |
| 一度でも予約に使われた予約メニューを外す | 拒否される。キャンセル済みの予約も数える |

例外として、**削除済み（アーカイブ済み）の予約メニュー**の紐付けは、予約の有無・購入者の有無にかかわらず外せる。

会員サイトの紐付けをこのツールで外しても、申込者向けサイトの公開状態は変わらない。

### 参照に使う他ドメインのツール

| ツール | 使いどころ |
|---|---|
| `getCreatorMembershipSites` | 会員サイト名から `membershipSiteId` を特定する（`id` / `name` / `isPublished` が返る） |
| `getCreatorMembershipSite` | 会員サイトの閲覧順の固定（`isFixedViewingOrder`）を確認する |

## 購入後設定

購入完了後にゲストが見る画面・受け取るメールと、その後の導線。`productId` / `productPlanId` はいずれもパスで、数値文字列。3 つの設定は互いに独立している。サンクスページ／メールと感想レポートには取得ツールがあるが、**自動リダイレクトの現在値を取得するツールは無い**（作成／更新と削除のみ）。

### `getCreatorProductPlanSettingsThanks` — サンクスページ／メール設定を取得

レスポンス: `{ content, bannerMediaId, bannerLinkUrl }`。未設定の項目は `content` が空文字、バナーの 2 項目が `null`。

### `patchCreatorProductPlanSettingsThanks` — サンクスページ／メール設定を更新

| パラメータ | 必須 | 制約 |
|---|---|---|
| `content` | ○ | 本文。10,000 文字まで。**毎回必須**（変えないときも現在値を送る） |
| `bannerMediaId` | — | バナー画像の ID。**省略＝維持、`null`＝削除** |
| `bannerLinkUrl` | — | バナーをタップしたときの移動先。`http://` / `https://` で始まる URL のみ。**省略＝維持、`null`＝削除** |

拒否される操作:

| 操作 | 結果 |
|---|---|
| `bannerLinkUrl` に `http(s)` 以外の URL | 拒否される |
| 自分の画像でない・存在しない `bannerMediaId` | 拒否される |
| バナー画像が無い状態（保存済みも `null`）で `bannerLinkUrl` だけ指定する | 拒否される。先に画像を設定するか、リンク先も `null` にする |

### `patchCreatorProductPlanSettingsRedirect` — 自動リダイレクト設定を作成／更新

| パラメータ | 必須 | 制約 |
|---|---|---|
| `redirectUrl` | ○ | 移動先の URL |
| `pendingSecond` | ○ | 移動までの待機秒数。**0〜15** の整数 |

作成と更新は同じツール。2 項目とも毎回送る。**現在値を読むツールが無い**ので、設定済みかどうか・移動先・秒数はユーザーに確認し、設定済みなら上書きになることを伝えてから送る。送信後は読み戻せないため、送った値を提示する。

### `deleteCreatorProductPlanSettingsRedirect` — 自動リダイレクト設定を削除

パラメータはパスのみ。**未設定でも成功する**（2 回呼んでも成功する）ため、有無を確かめるためだけに呼ばない。「自動リダイレクトを解除します（あとから再設定できます）」と伝えて承認を取ってから呼ぶ。

### `getCreatorProductPlanSettingsReviewReport` — 感想レポート設定を取得

レスポンス: `{ isAutoRequestEnabled }`。**一度も設定していないプランは `true`（有効）で返る。**

### `patchCreatorProductPlanSettingsReviewReport` — 感想レポート設定を更新

| パラメータ | 必須 | 制約 |
|---|---|---|
| `isAutoRequestEnabled` | ○ | 購入後に感想レポートの投稿依頼メールを自動で送るか |

## ライセンス設定（会員サイトの閲覧権限）

会員サイトが紐付いたプランで、購入したゲストが閲覧できるフォルダと期間を決める。パスは `membershipSiteId`（**数値**）/ `productId` / `planId`（数値文字列）の 3 つで、**プラン ID のパラメータ名だけ `planId`**。`membershipSiteId` は `getCreatorProductPlanOfferedContents` の値を使う。

### `getCreatorMembershipSiteProductPlanLicenseSetting` — プランのライセンス設定を取得

レスポンス: `{ folders: [{ id, name, isViewable }], isNewFoldersIncluded, accessDurationDays }`。`folders` は会員サイトの**全フォルダ**にフォルダ名と閲覧可否が付いた一覧。**フォルダ名を別ツールで引く必要はない。**

会員サイトが紐付いていないプラン・自分の会員サイトでない `membershipSiteId` は「見つからない」旨のエラー。

### `putCreatorMembershipSiteProductPlanLicenseSetting` — プランのライセンス設定を更新

| パラメータ | 必須 | 制約 |
|---|---|---|
| `viewableFolderIds` | ○ | 閲覧可能にするフォルダの ID（数値）の配列。**送った内容に置き換わる**。重複不可 |
| `isNewFoldersIncluded` | ○ | 今後追加されるフォルダも自動で閲覧可能にするか |
| `accessDurationDays` | ○ | 閲覧期限（日数）。**1〜1000**。無期限は `null`（省略不可） |

拒否される操作:

| 操作 | 結果 |
|---|---|
| 閲覧順の固定が有効な会員サイトで、一部のフォルダだけ閲覧可にする・`isNewFoldersIncluded: false` にする | 拒否される。全フォルダ + `true` にするか、先に閲覧順の固定を外す |
| `viewableFolderIds` に重複がある・その会員サイトに無いフォルダ ID を含む | 拒否される |

公開中・限定公開のプラン、または `isDeletable: false`（購入者がいる可能性のある）プランで、いま閲覧可能なフォルダを外すと、**そのフォルダを購入済みのゲストも見られなくなる**。送る前に承認を取る。

## このスキルで扱えない操作

| 操作 | 案内先 |
|---|---|
| 商品画像・サンクスページのバナー画像のアップロード・一覧取得 | 管理画面（画像 ID はユーザーから受け取る） |
| 事業者情報の登録・確認 | 管理画面 |
| 予約メニューをプランに紐付ける | 管理画面（解除はこのスキルで可能） |
| 自動リダイレクトの現在値の確認 | 管理画面（作成／更新と削除のみ可） |
| クーポン・収益分配・決済リンクの設定 | 管理画面 |
| 会員サイトそのもの（フォルダ・コンテンツ・タグ・閲覧順の固定などのサイト設定） | `membership-site-builder` スキル |
| 購入者・売上の確認 | `sales-reporter` スキル |
