# リンクタブのコンテンツ（ツール仕様）

## 提供されているもの / いないもの（ツールカタログ基準）

| 種別（画面の呼び方） | `type` | 追加 | 詳細取得 | 更新 | 削除 |
|---|---|---|---|---|---|
| 見出し | `heading` | ✅ | ✅ | ✅ | ✅ |
| 画像 | `image` | ✅ | ✅ | ✅ | ✅ |
| ウェブリンク | `link` | ✅ | ✅ | ✅ | ✅ |
| 地図 | `map` | ✅ | ✅ | ✅ | ✅ |
| SNSリンク | `sns` | ✅ | ✅ | ✅ | ✅ |
| 動画 | `video` | ✅ | ✅ | ✅ | ✅ |
| サービス購入導線 | `service` | ❌ | ✅ | ❌ | ✅ |
| 特典 | `benefit` | ❌ | ❌ | ❌ | ✅ |

- 削除は `type` を問わず `deleteProfileLinkContent` 1本。共通のサイズ変更は `patchProfileLinkContent`（全 type 可。本文のキーは `layout`）
- type 別のツール名（`{Type}` は Heading / Image / Link / Map / Sns / Video）: 詳細取得 `getCreatorProfileLink{Type}Content`、追加 `postCreatorProfileLink{Type}Contents`（複数形）、更新 `patchCreatorProfileLink{Type}Content`。サービス購入導線の詳細取得は `getCreatorProfileLinkServiceContent`
- ❌ は MCP に手段が無い。**サービス購入導線の追加・編集と特典の追加・編集**は管理画面を案内する。サイズ（`layout`）の変更と並べ替え・削除は上記の共通ツールでできる

## 共通ツール

### `getProfileLinkContents`（一覧）
クエリ: `limit`（1〜100、既定20）/ `offset`。返り値: `contents[]` と `totalCount`。**`limit: 100` を指定し、`totalCount` が取得件数より多ければ offset を進めて取り切る。** 並び順は表示順（未設定は末尾）。

各要素: `id` / `type` / `layout`（サイズ。本文のキーと同じ）/ `title` / `description` / `imageUrl` / `iconUrl` / `linkUrl` / `styleConfig`（見出しなど。無ければ null）/ `details`（`serviceId`・`price`・`snsType`・動画情報・ボタン設定・`contentStatus`）。`details.contentStatus` が `unavailable` のものは、紐づく対象が公開状態でなく、タイトル等の表示値は意味を持たない。

type 固有の項目（見出しの色、動画のソースなど）は一覧には出ない。**更新前は type 別の取得で現在値を取る。**

### `putProfileLinkContentsLayout`（並べ替え）
ボディ: `items: Array<{ contentId; displayOrder }>`。サーバーは `displayOrder` の昇順に並べた ID 列で保存する（表示順のキーであり、サービスのような絶対値の保存ではない）。0始まりの連番で渡す。

- **実行前に一覧を全件取り切り、`items` には並べ替え後の全コンテンツを `displayOrder` 付きで渡す**。全件を取得できているときは、1件だけを送らない（入れなかったコンテンツが、並べ替えたものの間に入り込む場合がある。ツール説明の記載）。1件だけを送る方法は、全件を取得できない場合の保険
- 所有外または存在しない ID が1件でも含まれると、**1件も更新せず 404**（確認済み）
- 実機確認（7件）: 存在しない ID を含めると **404 で、並びは一切変わらない**。`items` に1件だけ（先頭に）入れて送ると、**その1件が先頭に動き、残りは相対順を保つ**（1件だけ動かす用途は成立するが、上のとおり全件を取得できているときは全件を渡す）
- アーカイブ済みの ID の扱いは、ツール説明の記載のみで実機未確認。迷う場合は全件を渡す・取得結果の ID だけを使う

### `patchProfileLinkContent`（共通のサイズ）
パス `id`、ボディ `{ layout }`。`layout`: `small` / `horizontal` / `vertical` / `large`。画面の名称との対応は [ui-model.md](ui-model.md)（ユーザーには画面の名称で話す）。サービスカードの `cardSize` とは別物（SKILL.md 必須ルールC）。

### `deleteProfileLinkContent`
パス `id`。復元不可。対象のタイトルと種別を示して確認を取る。

## type 別の項目（追加 = post、更新 = patch）

**追加（post）も更新（patch）も全項目必須**。更新は「type 別の取得で現在値を取り、変える項目だけ差し替える」。取得結果の URL 項目（`imageUrl` 等）は更新には渡さず、ID 項目を渡す（SKILL.md 必須ルールE）。

**post と patch で `null`・空文字の意味が違う**（**実機確認が正**。ツール定義の説明と挙動が異なる行がある）:

| 値 | post（追加） | patch（更新） |
|---|---|---|
| 画像系の ID（`imageId` / `iconId` / `thumbnailImageId`）= `null` | 未設定（ウェブリンクは OGP / favicon を自動取得） | **画像・アイコンが外れる**（実機確認: ウェブリンクの `imageId: null` で `imageId` / `imageUrl` が空になった）。**画像を残すなら、取得した ID をそのまま渡す** |
| 動画・SNS のサムネイル（`thumbnailImageId`）= `null` | 未設定 | **サムネイルが外れる**（実機確認） |
| `videoPlaybackMode` = `null`（YouTube 動画） | 未設定 | **変更しない**（実機確認: `external` のまま残った。画像系 ID とは逆なので注意） |
| `videoMediaId` = `null` | 未設定 | **動画だけが外れる**（タイトル・サムネイル・サイズは残る。ツール定義の記載で、実機は未確認）。YouTube 動画で変えないなら取得値（`null`）のまま渡す。**アップロード動画の `videoMediaId` は、外す依頼でない限り取得値を渡す** |
| 画像の `linkUrl` = `""` | 遷移先なし | **遷移先が消える**（実機確認: 画像カードの `linkUrl: ""` で空になった） |
| `title` = `""` | 既定タイトルで保存される（ウェブリンク「リンク」／画像「画像コンテンツ」／動画「動画」／SNS「SNS リンク」） | 同じ（空にはならず既定タイトルに置き換わる） |

「画像を外して」「遷移先を消して」は、patch で `null` / `""` を送れば実現できる（承認を取ってから）。「タイトルを空にして」は既定タイトルに置き換わる（実機確認: ウェブリンク「リンク」・画像「画像コンテンツ」）。**画像・遷移先を残したいときは、取得した値をそのまま渡す（`null` / `""` を使わない）。**

### 見出し `Heading`
`layout` / `title`（100文字以内）/ `styleConfig`:
- `color`: `#RRGGBB` または `""`（既定色）
- `alignment`: `left` / `center` / `right`
- `fontSize`: `xlarge` / `large` / `medium` / `small` / `xsmall`
- `isBold` / `isUnderlined`: boolean

追加時、`styleConfig` は必須でサーバー任せの既定は無い。ユーザーが指定しない項目は、管理画面の追加フォームの初期値（画面で確認）に揃える: `color: "#121212"` / `alignment: left` / `fontSize: medium` / **`isBold: true`** / `isUnderlined: false`。**初期値で埋めたことを承認時に伝える。**（`color: ""` は「既定色」を表すが、画面の初期値は `#121212`）

### 画像 `Image`
`layout` / `title` / `description`（空文字可）/ `imageId`（string | null）/ `linkUrl`（遷移先。追加時に無ければ `""`。**更新で `""` を送ると遷移先が消える**。残すなら取得した値を渡す。実機確認）。

追加時にユーザーが指定しない項目は、管理画面の追加フォームの初期値に揃える（画面で確認）: `layout: large`（画面にサイズの設定は無い）/ `description: ""` / `linkUrl: ""`。**初期値で埋めたことを承認時に伝える。**

### ウェブリンク `Link`
`layout` / `title` / `description` / `linkUrl` / `imageId`（サムネ上書き）/ `iconId`（アイコン上書き）。追加時の `null` は OGP・favicon の自動取得。**更新時の `null` は画像が外れる**（上の表）。

### 地図 `Map`
`layout` / `address`（住所）/ `gmbPlaceId`（Google マップの Place ID）/ `location`（`latitude` / `longitude`）。**Place ID と緯度経度を調べるツールは無い。ユーザーが示した値だけを使い、住所から座標を創作しない。** 分からなければ管理画面での追加を案内する。実機確認: 追加・取得・削除は動くが、**Place ID の実在は検証されず、でたらめな値でも保存される**（無言の失敗になる）。創作値で追加しない。

### SNSリンク `Sns`
`layout` / `snsType`（`line` / `instagram` / `threads` / `facebook` / `x` / `tiktok` / `youtube` / `note`）/ `accountIdentifier`（プロフィール URL。http/https）/ `title`（空文字可）/ `description`（150文字以内。空文字可）/ `thumbnailImageId`（string | null）/ `iconStyle`（`original` = ブランドカラー、`simple` = モノクロ）。

### 動画 `Video`
`layout` / `videoSource`（`youtube` | `upload`）/ `title` / `thumbnailImageId`（string | null）/ `youtubeUrl` / `videoPlaybackMode`（`inline` = ページ内再生、`external` = タップで YouTube を開く、または null）/ `videoMediaId`（string | null）。

- `youtube` のとき: `youtubeUrl` 必須。`videoMediaId` は `null`
- `upload` のとき: `videoMediaId` 必須（既存の動画の更新用）。**アップロード動画の新規登録は、このスキルでは扱わない**ので、追加は YouTube のみ案内する。既存の upload 動画のタイトル等の更新は、取得した `videoMediaId` / `thumbnailImageId` / `videoSource` をそのまま渡す（実機未確認）

### サービス購入導線 `Service`（取得のみ。実機確認済み）
`getCreatorProfileLinkServiceContent`: サイズ / `title`（紐づくサービス名）/ `serviceId` / `showServiceRating` / `showServicePrice` / `isButtonVisible` / `buttonText` / `buttonColor`。更新ツールは無い。
