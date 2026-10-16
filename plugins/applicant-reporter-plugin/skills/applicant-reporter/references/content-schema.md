# 申込者・購読者レスポンスの読み方

`postCreatorProductApplicants` / `postCreatorProductSubscribers` が返す JSON の読み方と、ユーザーに提示するときの表示名。生の値をそのまま出さず、ここにある日本語の状態名で示す。

## 申込者一覧（`applicants[]`）

### 各項目

| 項目 | 意味 |
|---|---|
| `id` | 申込 ID。ユーザーには出さない |
| `guest` | 申込者。`id`（MOSH ID）、`name`、`email`、`iconPhotoId`。`name` / `email` は未設定なら `null`。ゲストを解決できなかったときは `guest` 自体が `null` |
| `planId` / `planName` | 申込んだプラン。対応するプランが無い（削除済みなど）と空文字 |
| `createdDateTime` | 申込日時（ISO 8601、オフセット付き） |
| `viewingRestrictionDateTime` | 視聴制限がかかった日時。無ければ `null` |
| `paymentType` | `one-time`（一括）/ `mosh-installments`（分割（分割入金型））/ `card-installments`（分割（一括入金型）） |
| `status` | 申込の状態（下表） |
| `order` | 注文。`paymentMethod`（`card` / `cash` / `ticket` / `bank-transfer`）、`contentTitle`（購入時のプラン名）、`status`（注文の生の状態）、`bankTransferExpirationDateTime`（振込期限）、`canceledDateTime`。注文を解決できないと `null` |
| `installmentsOrder` | 分割（分割入金型）のときだけ入る。`iteration`（分割回数）、`count`（支払済み回数）、`status`（`active` / `past_due` / `fulfilled` / `canceled`）。それ以外は `null` |

### `status` の表示名変換表（8 値）

| 値 | 表示名 | 意味 | 対象 |
|---|---|---|---|
| `approved` | 決済済み | 支払いが完了している | 一括決済 / 分割決済（一括入金）/ 銀行振込の入金反映後 |
| `canceled` | キャンセル済み | 申込が取り消された。分割決済（分割入金）のキャンセルもここに入る | 全て |
| `pending` | 振込待ち | 振込期限内で、まだ振込が確認されていない。`order.bankTransferExpirationDateTime` が期限 | 銀行振込 |
| `unapproved` | 反映待ち | 振込は届いたが、反映処理が済んでいない | 銀行振込 |
| `rejected` | 振込期限切れ | 期限までに振込が確認されなかった | 銀行振込 |
| `installmentActive` | 分割決済中 | 分割決済（分割入金）の途中。`installmentsOrder.count` / `iteration` で「何回目まで支払済みか」を添える | 分割決済（分割入金） |
| `installmentFulfilled` | 分割決済完了 | 全回の支払いが完了した | 分割決済（分割入金） |
| `installmentPastDue` | 決済エラー | 分割決済（分割入金）の途中で決済に失敗し、再決済待ち | 分割決済（分割入金） |

分割決済は 2 種類ある。**分割（分割入金型）**（`paymentType: mosh-installments`。毎回請求され、途中経過が `status` に出る）と**分割（一括入金型）**（`paymentType: card-installments`。カード会社の分割で、状態は `approved` / `canceled` の 2 つだけ）。

## 購読者一覧（`subscribers[]`）

### 各項目

| 項目 | 意味 |
|---|---|
| `id` | 購読 ID。ユーザーには出さない |
| `guest` | 購読者。申込者一覧と同じ形 |
| `planId` / `planName` | 購読しているプラン |
| `startedDateTime` | 購読開始日時 |
| `canceledDateTime` | 解約日時。未解約なら `null` |
| `scheduleEndDateTime` | 自動解約予定日時。予定が無ければ `null` |
| `status` | 購読の状態（下表） |
| `message` | 申込時のメッセージ。無ければ `null` |
| `billingDurationMonths` | 購読開始から現在（解約済みなら解約日）までの継続月数（切り捨て） |
| `billingCycleType` | `monthly`（月額）/ `yearly`（年額） |

### `status` の表示名変換表（4 値）

| 値 | 表示名 | 意味 |
|---|---|---|
| `active` | 継続中 | 購読が続いていて決済も正常 |
| `pastDue` | 決済エラー（購読は継続） | 直近の決済に失敗しているが購読は続いている。「継続中」に含めて扱う |
| `canceled` | 解約済み | `canceledDateTime` が解約日時。`scheduleEndDateTime` があれば期間末で終了する予定 |
| `trialing` | トライアル中 | 現状どのレコードにも付かない。絞り込みに指定しても 0 件になる |

## 表示名への変換表（生 ID → 何で引くか）

| 生の値 | 表示 | 変換元 |
|---|---|---|
| `productId` | 商品名 | `getCreatorProducts` の `name` |
| `planId` | プラン名 | レスポンスの `planName`（空文字なら `getCreatorProductPlans`） |
| `guest.id` | MOSH ID | 変換しない。ユーザーが求めたときだけ「MOSH ID」として出す |
| `status` | 状態名 | 上の変換表 |
| `paymentType` | 一括 / 分割（分割入金型）/ 分割（一括入金型） | 上の説明 |
| `order.paymentMethod` | カード / 現金 / チケット / 銀行振込 | `card` / `cash` / `ticket` / `bank-transfer` |
| 日時 | 日本時間の年月日 | ISO 8601 のオフセット付き値を変換する |

## 取得できないもの

- 金額・支払い回数ごとの明細（売上は `sales-reporter` スキル）
- 申込 1 件の詳細（アンケート回答・クーポン・返金可否）
- 申込のキャンセル・返金・購読の解約などの操作
- ゲストの電話番号・住所などのプロフィール項目

## 絶対に避けること

- 依頼された用途以外に `guest.name` / `guest.email` を出力・転記・要約する
- `status` や `paymentType` の生の値をそのままユーザーに見せる
- 「決済済み」と「反映待ち（`unapproved`）」を混同する
- 金額や支払い明細を推測して答える
