# 顧客宛メッセージ MCP ツールのパラメータ・状態条件（正本）

本スキルが使う5ツールのパラメータ・型・状態条件・レスポンスの正本。ツールを呼ぶ直前に、該当する操作の節だけ読む。

> 本ファイルのパラメータ・型・状態条件・レスポンス・実際に届く形は、MOSH の API スキーマ定義・ツール定義・配信処理の実挙動で確認したもの（2026-09 時点）。
> 値を書き換えるときは、記憶や推測ではなく MOSH の API スキーマ定義で裏取りする。

## 共通

- 引数のラッパー: 作成は `{ "bodyParams": {...} }`、編集は `{ "pathParams": { "id": "..." }, "bodyParams": {...} }`、単体取得・削除は `{ "pathParams": { "id": "..." } }`、一覧は `{ "queryParams": { "page": "1" } }`
- **`id` と `page` は文字列。** 数値のまま渡すとスキーマ検証で弾かれる（`zod.string().min(1)`）
- 対象クリエイターはサーバー側の認証情報から解決される
- MCP 経由の操作はサーバーからは常に**APIトークン認証**として扱われる。下記の「APIトークン認証では〜」はすべて MCP 経由の操作に常に適用される

## 作成（`postCustomersMessages`）

### リクエストボディ（15フィールドすべて必須）

省略したフィールドがあるとサーバーに届く前にスキーマ検証で弾かれる。使わない検索条件も既定値で必ず埋める。

| フィールド | 型 | 使わないときの値 | 画面での呼び名 | 補足 |
|---|---|---|---|---|
| `title` | string（1文字以上） | — | メッセージ名 | クリエイターの管理用。**ゲストには届かない** |
| `message` | string（1〜4500文字） | — | メッセージ本文 | 実際に届くのはこれだけ |
| `scheduledDateTime` | ISO8601（**オフセット必須**） | — | 予約送信 | 下記「配信日時」 |
| `customerIds` | **number[]** | `[]` | （個別選択） | `isAllCustomers: false` のときの宛先 |
| `isAllCustomers` | boolean | — | — | 宛先モードの切替。下記「宛先の2モード」 |
| `serviceIds` | integer[] | `[]` | プラン・サービス | 本ツール単体では ID を得る手段が無いが、`product-navigator` 経由で取得可（`customer-manager` の [`references/mcp-tools.md`](../../customer-manager/references/mcp-tools.md) が正本） |
| `subscriptionIds` | integer[] | `[]` | プラン・サービス（サブスク種別） | 同上（`serviceIds` と同じ経路） |
| `eventDateTime` | ISO8601 or null | `null` | 開催日時 | 分単位の完全一致。範囲検索ではない |
| `searchQuery` | string | `""` | 名前・メールアドレス | 顧客名 / メールアドレス / MOSH ID の部分一致 |
| `tagIds` | **string[]** | `[]` | 顧客タグ | OR条件。**コンタクトタグとは別体系** |
| `excludeTagIds` | **string[]** | `[]` | 除外タグ | OR条件 |
| `subscriptionState` | `active` / `canceled` / null | `null` | 継続状態 | — |
| `paymentMethod` | `card` / `cash` / `bank-transfer` / null | `null` | 支払い方法 | 銀行振込は**ハイフン区切り** |
| `paymentType` | `one_time` / `installment` / null | `null` | 支払いタイプ | — |
| `paymentStatus` | `approved` / `unapproved` / `pending` / `canceled` / `rejected` / null | `null` | 決済ステータス | — |

- **`customerIds` は数値配列、`tagIds` / `excludeTagIds` は文字列配列。** 同じボディの中で型が混在しているので、顧客IDを `["1001"]`、タグIDを `[42]` と取り違えない
- enum の意味・画面ラベルとの対応、`eventDateTime` が範囲検索でないことの詳細は、`customer-manager` の [`references/mcp-tools.md`](../../customer-manager/references/mcp-tools.md) が正本（同じ検索条件を共有している）
- **`postCustomersSearch` にあって、ここに無い条件**: `isEmailReceivingAllowed`（メール受信可否）・`userIds`・`page` / `limit`。メール受信可否での絞り込みはこの配信ではできない
- 完成形のボディは [../examples/message-bodies.json](../examples/message-bodies.json)

### 宛先の2モード

| `isAllCustomers` | 宛先 | `customerIds` |
|---|---|---|
| `true` | 検索条件に該当する**全顧客** | `[]`（空配列） |
| `false` | `customerIds` に指定した顧客 | 顧客ID（数値）の配列 |

**`isAllCustomers: false` でも検索条件は無効化されない。** サーバーは `customerIds` を検索条件の `userIds` として渡し、他の条件（`tagIds`・`subscriptionState` など）と **AND** で評価する。個別指定するときは他の条件をすべて既定値にしておかないと、指定した顧客の一部が条件から外れて届かない。

**【最重要・無言の失敗】`isAllCustomers: false` のまま `customerIds: []`（空配列）で作成すると、エラーにならず`isAllCustomers: true`と完全に同じ「全顧客」宛になる。** サーバーは `customerIds` を検索条件の `userIds` に変換して渡すが、空配列は上流で「絞り込み条件なし」と解釈されるため。**「個別指定のIDがまだ分からないので、とりあえず条件は空のまま作っておいて」という依頼を額面通りに実行しない。** IDが特定できるまで作成を待つか、`postCustomersSearch` の `searchQuery`（名前等）で実際に特定してから `customerIds` を埋める。0件エラー（400）にはならないので、「まず空で作ってみて後で直す」という進め方は成立しない。

### 配信日時（`scheduledDateTime`）

- 形式は**タイムゾーンオフセット付き** ISO8601（例: `2026-09-20T10:00:00+09:00`）。オフセットの無い `2026-09-20T10:00:00` はスキーマ検証で弾かれる
- **APIトークン認証（＝MCP 経由）では、現在時刻の 10 分後より前の日時はすべて 422 で拒否される**（エラー文言は「API トークン認証では即時配信できません。」）。過去日時を指定しても即時配信にはならず 422 になる
- したがって **MCP からは即時配信できない**。作れるのは「10 分後以降の予約配信」だけ
- 画面の注意書きも同じ（「送信予定日は現在時刻の10分後以降の日時を選択してください。」）

### レスポンスと、作成後にできること

- 返るのは `{ "message": "メッセージを送信しました。" }` の1行だけで、**作成されたメッセージの ID は返らない**
- 作成直後の ID が必要なら `getCustomersMessages`（1ページ目が作成日時の新しい順）で `title` と `sendDateTime` を突き合わせて特定する。同名の `title` が複数あると特定できないので、その場合はユーザーに確認する

### 実際に届く形（本文を組むときの制約）

配信処理の実挙動として確認済み:

- **メールと MOSH公式LINE の両方に届く**。メールアドレス登録済みのゲストにはメール、LINE連携済みのゲストには MOSH公式LINE アカウントから送られ、両方に当てはまるゲストには両方届く。**クリエイター自身の LINE公式アカウントは使わない**
- メールの件名は `{クリエイター名}さんからメッセージが届きました` で**自動生成される**。件名を指定する手段は無い（`title` は件名ではない）
- 本文は**プレーンテキストのみ**。テンプレートに `{{message_body}}` としてそのまま差し込まれる。画像・ボタン・カルーセル・HTML は使えず、リンクは URL の文字列として出る
- **埋め込み変数は無い。** テンプレートが置換するのはクリエイター名と本文だけで、`{{line_name}}` 等を本文に書いても置換されずそのまま受信者に届く（無言の失敗）
- メールにはMOSHのフッター（ヘルプ・サポート窓口・SNS）と**配信停止リンク**が自動で付く
- 送信完了後、クリエイター本人宛に「「{title}」の送信が完了しました」という完了メールが届く
- この配信は**プランのメール・LINE配信可能数にカウントされない**（使用率アラートも付かない）

### 宛先は「条件」であって「承認時点の名簿」ではない

作成時に一度宛先を解決して保存するが、**送信時にもう一度同じ検索条件で解決し直し、その時点の対象者に送る**（保存済みの名簿は送信時のリストで置き換えられる）。承認から配信までの間にタグの付け外し・新規購入・解約があれば、実際の届き先は変わる。

## 宛先の件数確認（`postCustomersSearch` での事前プレビュー）

作成前に `postCustomersSearch` を**同じ検索条件**で呼んで件数を確認する。管理画面も同じ検索の `totalCount` を「◯名予定」として表示しているので、これが宛先件数の正しい見積もり方法。

`postCustomersSearch` のボディへの移し替え:

| `postCustomersMessages` | `postCustomersSearch` | 備考 |
|---|---|---|
| `serviceIds` / `subscriptionIds` / `eventDateTime` / `searchQuery` / `tagIds` / `excludeTagIds` / `subscriptionState` / `paymentMethod` / `paymentType` / `paymentStatus` | 同名フィールドへそのまま | 値を変えない |
| `customerIds`（number[]） | `userIds`（**string[]**） | `isAllCustomers: false` のときのみ。**数値を文字列に変換する**（`[1001]` → `["1001"]`） |
| `isAllCustomers: true` | `userIds: []` | — |
| （対応無し） | `isEmailReceivingAllowed` | **必ず `null`。** ここに値を入れると配信の宛先より狭い件数を見せてしまう |
| （対応無し） | `page: 1` / `limit: 10` | 件数だけ見るので1ページ目で十分 |

`totalCount` が宛先件数。**`totalCount: 0` のまま作成すると、サーバー側で「送信対象の顧客が見つかりませんでした。」の 400 になる**ので、作成せず条件を見直す。

`postCustomersSearch` の呼び方・ページング・個人情報の提示ルールは `customer-manager` スキルが正本。

## 一覧（`getCustomersMessages`）

- 引数: `{ "queryParams": { "page": "1" } }`。**`page` は文字列**（既定 `"1"`）
- **1ページ 10 件固定。** `limit` は指定できない。作成日時の新しい順
- 返り値: `messages[]`（`id` / `title` / `sendDateTime` / `isSent` / `failureReason`）と `totalCount`
- **本文は含まれない。** 本文が要るものだけ `getCustomersMessage` で個別取得する
- `sendDateTime` は予約日時。即時配信されたメッセージは予約日時を持たないため、代わりに作成日時が入る
- `failureReason`: `null`=失敗なし / `ERROR_BUDGET_LIMIT`=配信可能数の上限超過 / `ERROR_NOT_RETRYABLE`=再試行できないエラー。**コンタクト配信と共通の enum** で、顧客宛メッセージはプランの配信可能数にカウントされないため `ERROR_BUDGET_LIMIT` の意味は自明ではない。値が入っていたら「配信に失敗している」ことだけを伝え、原因の断定はしない

## 詳細（`getCustomersMessage`）

- 引数: `{ "pathParams": { "id": "123" } }`（**文字列**）
- 返り値: `id` / `title` / `message`（本文）/ `sendDateTime` / `customerNames` / `isSent` / `failureReason`
- **`customerNames` は MCP 経由では必ず空配列。** 顧客名の解決にはセッション認証の署名が必要で、APIトークン認証では取得しない。**空配列を「宛先0名」と解釈しない。** 宛先を知りたい場合は `postCustomersSearch` で条件から件数を出すか、管理画面を案内する
- 他のクリエイターのメッセージ・存在しない ID は 404

## 編集（`patchCreatorCustomersMessage`）

- 引数: `{ "pathParams": { "id": "123" }, "bodyParams": { "title": ..., "message": ..., "scheduledDateTime": ... } }`
- **3フィールドすべて必須の全置換。** 部分更新ではないので、変えない項目も `getCustomersMessage` で取得した現在値をそのまま入れる
- **変更できるのは `title` / `message` / `scheduledDateTime` だけ。宛先（検索条件・`customerIds`）は変更できない。** 宛先を変えたい場合は削除して作り直す
- `scheduledDateTime` の制約は作成と同じ（オフセット付き ISO8601、MCP からは現在時刻の 10 分後以降のみ）。**この検証は全置換のたびに毎回実行される。** `title` / `message` だけ変えて日時は現状維持のつもりでも、送信する `scheduledDateTime`（＝ `getCustomersMessage` で取得した現在値）が現在時刻の10分後より前なら 422 になる（編集可否のチェックの直後に無条件で実行される）
- **編集できるのは予約日時前の予約配信のみ、かつ実質的には予約日時の10分前まで。** サーバーの編集可否の判定自体は「未送信 かつ 予約日時あり かつ 予約日時が現在時刻より後」だが、上記の `scheduledDateTime` 再検証がそれより厳しく効くため、**残り10分を切った予約は編集そのものができない**。送信済み・即時配信・予約日時を過ぎたものは 422「すでに配信済みのため、更新することができません。」

## 削除（`deleteCustomersMessage`）

- 引数: `{ "pathParams": { "id": "123" } }`（**文字列**）
- 可否条件は編集可否の判定（未送信 かつ 予約日時が現在時刻より後）だけで、`scheduledDateTime` の再検証は行われない。**編集と違い、予約日時の直前まで削除できる。** 送信済み・即時配信・予約日時を過ぎたものは 422「すでに配信済みのため、削除することができません。」
- メッセージ本体と宛先レコードを物理削除する。**復元できない**（下書きに戻す・予約だけ解除するといった中間状態は無い）

## エラー応答

| 状況 | 返り値 |
|---|---|
| `scheduledDateTime` が現在時刻の 10 分後より前（作成・**編集とも**。編集は日時を変えないつもりで現在値をそのまま送った場合も対象） | 422「API トークン認証では即時配信できません。scheduledDateTime は現在時刻の 10 分後以降を指定してください。」 |
| 条件に該当する顧客が0名 | 400「送信対象の顧客が見つかりませんでした。」 |
| 送信済み・予約日時経過後の編集／削除 | 422「すでに配信済みのため、更新（削除）することができません。」 |
| 存在しない ID・他クリエイターのメッセージ | 404 |
| アクセストークン失効など | 401 |
| 必須フィールド欠落・型違い・enum 違反・4500文字超過 | サーバーに届く前にスキーマ検証で弾かれる |
