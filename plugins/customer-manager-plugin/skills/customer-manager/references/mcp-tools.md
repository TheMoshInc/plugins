# postCustomersSearch ツール仕様（正本）

顧客検索スキルが使う MCP ツールは `postCustomersSearch` の1本だけ。ここがパラメータ・enum・返り値の正本。

> 本ファイルのパラメータ・enum・レスポンス・画面ラベル対応は、MOSH の API スキーマ定義とツール定義で確認したもの（2026-09 時点）。
> 値を書き換えるときは、記憶や推測ではなく MOSH の API スキーマ定義で裏取りする。

## 呼び出しの形

- ツール名: `postCustomersSearch`（HTTP では `POST /creator/customers/search`）
- 引数は `bodyParams` でラップする: `{ "bodyParams": { ...検索条件... } }`
- `annotations.destructiveHint: false`（参照専用。データを変更しない）
- 対象クリエイターはサーバー側の認証情報から解決される（`hostId` はクライアントから指定できない）
- 上流には常に `type=client` が渡る。取得できるのは**購入者（顧客）だけ**で、フォロワー・順番待ち・配信停止者などの他区分は取得できない

## リクエストボディ（14フィールドすべて必須）

省略したフィールドがあるとサーバーに届く前に MCP クライアント側のスキーマ検証で弾かれる。使わない条件も既定値で必ず埋める。

| フィールド | 型 | 使わないときの値 | 画面での呼び名 | 補足 |
|---|---|---|---|---|
| `serviceIds` | integer[] | `[]` | プラン・サービス | ID を得る MCP ツールが無い（後述） |
| `subscriptionIds` | integer[] | `[]` | プラン・サービス（サブスク種別） | 上流の `subscriptionServiceIds`。個々の契約IDではなく**サブスク型サービスのID** |
| `eventDateTime` | ISO8601 or null | `null` | 開催日時 | 分単位の完全一致。範囲指定不可（下記の注意） |
| `searchQuery` | string | `""` | 名前・メールアドレス | 顧客名 / メールアドレス / MOSH ID の部分一致 |
| `subscriptionState` | enum or null | `null` | 継続状態 | `active`=継続中 / `canceled`=解約済み |
| `paymentMethod` | enum or null | `null` | 支払い方法 | `card`=カード / `cash`=現金 / `bank-transfer`=銀行振込 |
| `paymentType` | enum or null | `null` | 支払いタイプ | `one_time`=一括 / `installment`=分割払い |
| `paymentStatus` | enum or null | `null` | 決済ステータス | `approved`=成功 / `unapproved`=反映待ち / `pending`=振込待ち / `canceled`=キャンセル / `rejected`=振込期限切れ |
| `isEmailReceivingAllowed` | boolean or null | `null` | メール受信ステータス | `true`=受信中 / `false`=停止中。受信設定が未登録の顧客は**受信中（true）扱い** |
| `tagIds` | string[] | `[]` | 顧客タグ | OR条件（いずれか1つでも持つ顧客を含める）。**コンタクトタグとは別体系** |
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
- `id` は**ユーザーID**（`userIds` に渡せるのと同じ体系。ただし `userIds` は文字列）
- `email` は null になり得る
- `avatarUrl` は未設定でもデフォルト画像のURLが必ず入る（アバターの有無の判定には使えない）
- `tags` は**顧客タグ**（id / name のみ）。返るのはその顧客に付いているタグだけで、クリエイターの顧客タグ全一覧ではない

## MCP に無いもの（このスキルで代用できないこと）

以下は MOSH の API には存在するが MCP には公開されておらず、ツールカタログに出てこない（2026-09 時点）。

| できないこと | 該当 operationId | 代替 |
|---|---|---|
| 顧客タグの一覧取得・作成 | `getCreatorCustomerTags` / `postCreatorCustomerTags` | タグ名からIDを引けない。ユーザーに確認するか管理画面を案内 |
| 顧客へのタグ付け・タグ剥がし | `putCreatorCustomerTags` / `postCreatorCustomerTagCustomers` / `deleteCreatorCustomerTagCustomer` | 管理画面「顧客一覧」 |
| 絞り込み用サービス一覧の取得 | `getCreatorCustomersFiltersServices` | `serviceIds` は空配列。サービス名で絞りたい依頼は管理画面を案内 |
| 顧客のプロフィール詳細・予約履歴・カルテ | `getCreatorCustomer` / `getCreatorCustomerReservations` / `getCreatorCustomerNote` / `putCreatorCustomerNote` | 管理画面「顧客一覧」の顧客詳細 |

**他ドメインの MCP ツールで代用しない。** `getCreatorContactTags`（コンタクトタグ一覧）は取得できるが、返る id は顧客タグとは**別体系**なので `tagIds` / `excludeTagIds` に渡してはならない（[customer-vs-contact.md](customer-vs-contact.md)）。

## エラー応答

- 成功応答は 200 のみ。HTTP 400 以上のときは `isError: true` でレスポンスボディがそのまま返る
- 401: 認証エラー（MCP のアクセストークン失効など）
- 500: 上流の顧客検索APIの失敗またはタイムアウト（サーバー側で 5 秒でタイムアウトする）
- 必須フィールド欠落・enum 違反・`limit` 範囲外は、サーバーに届く前にスキーマ検証で弾かれる
