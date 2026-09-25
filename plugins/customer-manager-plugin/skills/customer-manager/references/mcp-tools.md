# 顧客管理 MCP ツール仕様（正本）

本スキルが使う MCP ツールは7本。ここがパラメータ・型・enum・返り値の正本。

| ツール | 用途 | 書き込み |
|---|---|---|
| `postCustomersSearch` | 顧客の検索・件数 | なし |
| `getCreatorCustomer` | 顧客1人のプロフィール＋付いている顧客タグ | なし |
| `getCreatorCustomerReservations` | 顧客1人の申込・予約履歴 | なし |
| `getCreatorCustomerTags` | 顧客タグの全一覧 | なし |
| `postCreatorCustomerTags` | 顧客タグを作成して顧客1人に付与 | あり |
| `postCreatorCustomerTagCustomers` | 既存の顧客タグ1つを最大100人に付与 | あり |
| `deleteCreatorCustomerTagCustomer` | 顧客1人から顧客タグ1つを解除 | あり |

> 本ファイルのパラメータ・型・enum・レスポンス・画面ラベル対応は、MOSH の API スキーマ定義・ツール定義・サーバー側の実挙動で確認したもの（2026-09 時点）。
> 値を書き換えるときは、記憶や推測ではなく MOSH の API スキーマ定義で裏取りする。

---

# postCustomersSearch（検索）

## 呼び出しの形

- ツール名: `postCustomersSearch`（HTTP では `POST /creator/customers/search`）
- 引数は `bodyParams` でラップする: `{ "bodyParams": { ...検索条件... } }`
- 参照専用。データを変更しない
- 対象クリエイターはサーバー側の認証情報から解決される（`hostId` はクライアントから指定できない）
- 上流には常に `type=client` が渡る。取得できるのは**購入者（顧客）だけ**で、フォロワー・順番待ち・配信停止者などの他区分は取得できない

## リクエストボディ（14フィールドすべて必須）

省略したフィールドがあるとサーバーに届く前に MCP クライアント側のスキーマ検証で弾かれる。使わない条件も既定値で必ず埋める。

| フィールド | 型 | 使わないときの値 | 画面での呼び名 | 補足 |
|---|---|---|---|---|
| `serviceIds` | integer[] | `[]` | プラン・サービス | 本スキル単体では ID を得る手段が無いが、`product-navigator` の `moshServiceId` 経由で得られる（後述） |
| `subscriptionIds` | integer[] | `[]` | プラン・サービス（サブスク種別） | 上流の `subscriptionServiceIds`。個々の契約IDではなく**サブスク型サービスのID**（`serviceIds` と同じ経路で取得可） |
| `eventDateTime` | ISO8601 or null | `null` | 開催日時 | 分単位の完全一致。範囲指定不可（下記の注意） |
| `searchQuery` | string | `""` | 名前・メールアドレス | 顧客名 / メールアドレス / MOSH ID の部分一致 |
| `subscriptionState` | enum or null | `null` | 継続状態 | `active`=継続中 / `canceled`=解約済み |
| `paymentMethod` | enum or null | `null` | 支払い方法 | `card`=カード / `cash`=現金 / `bank-transfer`=銀行振込 |
| `paymentType` | enum or null | `null` | 支払いタイプ | `one_time`=一括 / `installment`=分割払い |
| `paymentStatus` | enum or null | `null` | 決済ステータス | `approved`=成功 / `unapproved`=反映待ち / `pending`=振込待ち / `canceled`=キャンセル / `rejected`=振込期限切れ |
| `isEmailReceivingAllowed` | boolean or null | `null` | メール受信ステータス | `true`=受信中 / `false`=停止中。受信設定が未登録の顧客は**受信中（true）扱い** |
| `tagIds` | string[] | `[]` | 顧客タグ | OR条件（いずれか1つでも持つ顧客を含める）。id は `getCreatorCustomerTags` で名前から引く。**コンタクトタグとは別体系** |
| `excludeTagIds` | string[] | `[]` | 除外タグ | OR条件（いずれか1つでも持つ顧客を除外） |
| `userIds` | string[] | `[]` | （画面に無い） | OR条件。最大100件。レスポンスの `customers[].id` と同じ体系 |
| `page` | integer ≥1 | `1` | — | 1始まり |
| `limit` | integer 1〜100 | `10` | — | 上流には `offset = limit × (page - 1)` に変換して渡される |

完成形のボディは [../examples/search-body.json](../examples/search-body.json) を参照。

### `eventDateTime` の注意（範囲検索ではない）

サーバー側は受け取った日時を年・月・日・時・分に分解し、開催日時の**完全一致**として扱う。

- 「今月申し込んだ人」のような**期間での絞り込みには使えない**
- 分解は実行環境のタイムゾーン解釈に依存するため、時刻まで指定した絞り込みは意図とずれることがある
- 画面ではイベント／プライベート型のサービスを選択したときだけ使えるフィルターで、単独では使わない
- **原則 `null` を渡す。** 開催日で絞りたい依頼は管理画面「顧客一覧」のフィルターを案内する

## レスポンス

```json
{
  "customers": [
    {
      "id": 1,
      "name": "MOSH太郎",
      "email": "taro.mosh@example.com",
      "avatarUrl": "https://.../ic_profile_default_new.png",
      "isEmailReceivingAllowed": true,
      "tags": [{ "id": "1", "name": "優良顧客" }]
    }
  ],
  "totalCount": 2
}
```

- `totalCount` は**ページネーション前の総件数**。1ページに収まらない場合は `page` を増やして続きを取る
- 該当者ゼロは `customers: []` / `totalCount: 0`。エラーではない
- `id` は**顧客ID**（`userIds` に渡せるのと同じ体系。ただし `userIds` は文字列）。`getCreatorCustomer` / `getCreatorCustomerReservations` / タグ付与・解除の顧客ID（整数）にそのまま使う
- `email` は未登録なら `null`（顧客詳細では空文字になる点が違う）
- `avatarUrl` は未設定でもデフォルト画像のURLが必ず入る（アバターの有無の判定には使えない）
- `tags` は**顧客タグ**（id / name のみ）。返るのはその顧客に付いているタグだけ。全一覧は `getCreatorCustomerTags`

## serviceIds / subscriptionIds は `product-navigator` 経由で取得できる

`getCreatorCustomersFiltersServices` は MCP に無いが、**商品プランとして販売している商品の serviceId は `product-navigator` スキルから取得できる。**

- `product-navigator` の `getCreatorProductPlan` / `getCreatorProductPlans` のレスポンスに含まれる `moshServiceId`（文字列、例: `"123"`）が、`postCustomersSearch` の `serviceIds` / `subscriptionIds`（整数）と**同一のID**（顧客絞り込み用サービス一覧の `id`（整数）が商品プランの `moshServiceId` と直接突き合わせで解決されることを確認済み。2026-09 時点）
- 使い方: `moshServiceId` を数値に変換して `serviceIds` に入れる。「サブスク型」のプランなら `subscriptionIds` に入れる（画面の見分け方は `product-navigator` 側のプランのサブスク種別表示に従う）
- **商品プランに紐づいていない古い形式のサービス**（イベント・プライベートセッション等を商品プラン化せず単体で販売している場合）にはこの経路が無い。その場合のみ、ID不明のまま空配列でユーザーに確認する（必須ルールB）
- 「サービス名で絞りたい」だけで商品名の手がかりが無い依頼は、まず商品名・プラン名を聞くか `product-navigator` で検索する

## エラー応答（全ツール共通）

- ツールハンドラは HTTP 400 以上のとき `isError: true` でレスポンスボディをそのまま返す
- 401: 認証エラー（MCP のアクセストークン失効など）
- 400: MOSH の内部 API が入力を受け付けなかった（タグ作成で内部 API が 400/422 を返したものをそのまま返す）。タグ名の長さ（1〜50文字）はこれより前のスキーマ検証で弾かれる
- 403: 認可エラー（顧客詳細・予約で OpenAPI に定義あり）
- 404: 顧客・タグが見つからない（他クリエイターの顧客も同じ扱い）、解除時はそのタグが付いていない。検索は 404 を返さない
- 500: 上流 API の失敗またはタイムアウト（サーバー側で 5 秒でタイムアウトする）
- 必須フィールド欠落・enum 違反・`limit` 範囲外・型違い（整数と文字列）は、サーバーに届く前にスキーマ検証で弾かれる

---

# getCreatorCustomer（顧客1人のプロフィール）

- 引数: `{ "pathParams": { "id": 1001 } }`。`id` は**整数**（`postCustomersSearch` の `customers[].id` と同じ値）
- 他クリエイターの顧客・存在しない顧客は **404**（存在の有無を秘匿する。「見つからない」と伝える）

## レスポンス

```json
{
  "id": 856842,
  "name": "有村真希",
  "email": "maki@example.com",
  "avatarUrl": "https://.../default.png",
  "phone": "090-0000-0000",
  "gender": "female",
  "isEmailReceivingAllowed": true,
  "tags": [{ "id": "1", "name": "優良顧客" }]
}
```

| フィールド | 画面での表示 | 補足 |
|---|---|---|
| `id` | MOSH ID | 検索の `customers[].id` と同じ |
| `name` | 名前 | 空文字あり |
| `email` | メールアドレス | **未登録は空文字**（`null` ではない。検索結果では未登録が `null` なので扱いが違う）→「未設定」 |
| `phone` | 電話番号 | 未登録は空文字 →「未設定」 |
| `gender` | 性別 | `male`=男性 / `female`=女性 / `none`=**無回答** |
| `isEmailReceivingAllowed` | メール受信ステータス | `true`=受信中 / `false`=停止中 |
| `tags` | 顧客タグ | その顧客に付いている顧客タグ（id は文字列） |
| `avatarUrl` | — | 未設定でもデフォルト画像のURLが入る |

---

# getCreatorCustomerReservations（顧客1人の申込・予約履歴）

- 引数: `{ "pathParams": { "customerId": 1001 }, "queryParams": { ... } }`。**`queryParams` は必須キー**（条件が無ければ `{}`）
- `customerId` は**整数**。他クリエイターの顧客・存在しない顧客は **404**
- 並び順は**申込日時の新しい順**

| クエリ | 型 | 省略時 | 補足 |
|---|---|---|---|
| `reservedAtFrom` | ISO 8601（タイムゾーン付き） | 下限なし | 申込日時がこの日時**以降** |
| `reservedAtTo` | ISO 8601（タイムゾーン付き） | 上限なし | 申込日時がこの日時**以前** |
| `limit` | integer 1〜100 | **offset 以降の全件** | 件数が多いときは指定して区切る |
| `offset` | integer ≥0 | `0` | 取得開始位置（ページ番号ではない） |

期間は日本時間で組み立てる。例:「2026年9月」→ `reservedAtFrom: "2026-09-01T00:00:00+09:00"`, `reservedAtTo: "2026-09-30T23:59:59+09:00"`。

**顧客1人ごとのツール。** 顧客全体を期間で絞る用途（「今月申し込んだ顧客一覧」）には使わない（SKILL.md 必須ルールF）。

## レスポンス

```json
{
  "reservations": [
    {
      "id": "rsv_01",
      "serviceId": "svc_01",
      "serviceTitle": "60分パーソナルレッスン",
      "startAt": "2025-02-01T10:00:00+09:00",
      "endAt": "2025-02-01T11:00:00+09:00",
      "reservedAt": "2025-01-15T18:30:00+09:00",
      "serviceImageUrl": "https://...",
      "message": "今回はよろしくお願いします",
      "participantCount": 1,
      "options": [{ "name": "レンタルシューズ", "price": 500 }]
    }
  ],
  "totalCount": 1
}
```

- `totalCount` は**期間で絞った後の総件数**。該当なしは `reservations: []` / `totalCount: 0`（エラーではない）
- `reservedAt`=申込日時、`startAt` / `endAt`=開催日時。「いつ申し込んだか」と「いつ参加するか」を取り違えない
- **古い予約では申込日時が記録されていないことがある。** その場合 `reservedAt` には予約した開催枠の日時（無ければ開催開始日時）が入り、`reservedAtFrom` / `reservedAtTo` の絞り込みもその値で行われる。`reservedAt` と `startAt` が一致する予約は、申込日時として断定しない
- `message`（予約時の連絡事項）は個人情報。聞かれたときだけ出す
- `serviceImageUrl` は取得に失敗すると空文字になる（画像が無いとは限らない）
- `serviceId` は文字列で、`postCustomersSearch` の `serviceIds`（整数）に流用できるかは未確認。流用しない

---

# 顧客タグ

## getCreatorCustomerTags（一覧）

- 引数なし（`{}`）
- レスポンス: `{ "customerTags": [{ "id": "12", "name": "VIP" }, ...] }`。クリエイターの**全顧客タグ**が1回で返る（ページングなし）
- タグ名から id を引くときはこれを使い、**名前の完全一致**で探す（入手経路の全体は [customer-vs-contact.md](customer-vs-contact.md)）

## postCreatorCustomerTags（作成＋1人に付与）

- 引数: `{ "bodyParams": { "name": "VIP", "customerId": 1001 } }`。`name` は1〜50文字、`customerId` は整数
- **タグだけを作ることはできない。** 必ず顧客1人への付与とセット
- 同名タグが既にあれば新規作成せず、それを付与する（同名の重複は作られない）。同名かどうかの判定は MOSH の内部 API 側で行われ、全角半角・大文字小文字を同一視するかは**未確認**。表記ゆれのある既存タグがあれば、実行前にユーザーに確認する
- レスポンス（201）: 作成（または再利用）されたタグ `{ "id": "34", "name": "VIP" }`。この id を続く一括付与に使う
- 顧客が見つからない（他クリエイターの顧客を含む）→ 404。MOSH の内部 API が名前を受け付けなかった → 400（空・51文字以上はその前のスキーマ検証で弾かれる）

## postCreatorCustomerTagCustomers（既存タグを複数人に付与）

- 引数: `{ "pathParams": { "id": "34" }, "bodyParams": { "customerIds": [1001, 1002] } }`
- タグ `id` は**文字列**（英数字・`_`・`-` のみ）、`customerIds` は**整数**の配列で1〜100件。101人以上は分けて呼ぶ
- 既に付いている顧客はそのまま（何度実行しても結果は同じ）
- サーバーは先にタグの存在を確認し（無ければ 404、誰にも付与されない）、その後**顧客を先頭から1人ずつ**付与する。途中の顧客で 404 になると、それより前の顧客には付与済みのまま 404 が返る
- 成功時は `{ "message": "OK" }` のみ（誰に付いたかの一覧は返らない）

## deleteCreatorCustomerTagCustomer（1人から解除）

- 引数: `{ "pathParams": { "id": "34", "customerId": 1001 } }`（タグ id は文字列、顧客ID は整数）
- 外れるのは指定したタグだけ。他のタグは残る
- タグが付いていない・タグが存在しない・顧客が見つからない → いずれも 404
- 複数人から外すときは1人ずつ呼ぶ（一括解除のツールは無い）。取り消したいときは `postCreatorCustomerTagCustomers` で付け直せる

---

# MCP に無いもの（このスキルで代用できないこと）

以下は MOSH の API には存在するが MCP には公開されておらず、ツールカタログに出てこない（2026-09 時点）。

| できないこと | 該当 operationId | 代替 |
|---|---|---|
| 顧客カルテ（メモ・画像）の閲覧・編集 | `getCreatorCustomerNote` / `putCreatorCustomerNote` / `*CustomerNoteImage*` | 管理画面「顧客一覧」の顧客詳細 |
| 顧客のタグを丸ごと置き換える | `putCreatorCustomerTags` | 付与・解除を組み合わせる（差分を承認のうえで） |
| 顧客タグ自体の名前変更・削除 | （MCP にも OpenAPI にも無い） | 管理画面 |
| 顧客のプロフィール編集・削除 | （同上） | 管理画面 |
| 顧客とのチャット履歴 | `getCreatorCustomerChats` | 管理画面 |
| 絞り込み用サービス一覧の直接取得 | `getCreatorCustomersFiltersServices` | `product-navigator` 経由（上記「serviceIds / subscriptionIds」） |

**他ドメインの MCP ツールで代用しない。** `getCreatorContactTags` / `postCreatorContactTags`（コンタクトタグ）は呼べるが、顧客タグとは**別体系**（[customer-vs-contact.md](customer-vs-contact.md)）。
