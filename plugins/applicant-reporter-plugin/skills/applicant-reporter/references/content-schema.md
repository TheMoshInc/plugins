# 申込者・購読者レスポンスの読み方

`postCreatorProductApplicants` / `postCreatorProductSubscribers` と、1 件の詳細の `getCreatorProductApplicant` / `getCreatorProductSubscriber` が返す JSON の読み方と、ユーザーに提示するときの表示名。生の値をそのまま出さず、ここにある日本語の状態名で示す。

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
| `billingDurationMonths` | 購読開始から現在（解約済みなら解約日）までの継続月数（切り捨て）。「満 N ヶ月」と示し、「N ヶ月目」とは書かない |
| `billingCycleType` | `monthly`（月額）/ `yearly`（年額） |

### `status` の表示名変換表（4 値）

| 値 | 表示名 | 意味 |
|---|---|---|
| `active` | 継続中 | 購読が続いていて決済も正常 |
| `pastDue` | 決済エラー（購読は継続） | 直近の決済に失敗しているが購読は続いている。「継続中」に含めて扱う |
| `canceled` | 解約済み | `canceledDateTime` が解約日時。`scheduleEndDateTime` があれば期間末で終了する予定 |
| `trialing` | トライアル中 | 現状どのレコードにも付かない。絞り込みに指定しても 0 件になる |

## 申込 1 件の詳細（`getCreatorProductApplicant`）

一覧と同じ `id` / `guest` / `planId` / `planName` / `createdDateTime` / `viewingRestrictionDateTime` / `paymentType` / `status` に加えて次が返る。一覧の `order`（単数）は返らず、`orders`（配列）になる。

| 項目 | 意味 |
|---|---|
| `message` | 申込時のゲストからのメッセージ。無ければ `null` |
| `cardInstallmentCount` | 分割（一括入金型）のカード分割回数。それ以外は `null` |
| `orders[]` | 支払い明細。分割（分割入金型）は回ごとに複数件。各要素は `status`（注文の生の状態）、`paymentMethod`、`amount`（金額）、`discount`（割引額）、`additionalFee`（追加手数料）、`paidAmount`（支払済み金額）、`differenceAmount`（差額）、`currency`、`createdDateTime`、`approvedDateTime`（決済日時）、`canceledDateTime`、`bankTransferExpirationDateTime`（振込期限） |
| `installmentsOrder` | 分割（分割入金型）のときだけ入る。一覧の項目に加え、`interval`（`month` 月ごと / `hour`）、`nextApprovedDateTime`（次回決済予定日）、`amount`（1 回あたりの金額） |
| `coupon` | 使ったクーポン。`code`、`name`、`amountOff`（定額割引）か `percentOff`（割合割引 %）。未使用なら `null` |
| `preCouponAmount` | クーポン適用前の金額。一括と分割（一括入金型）で入る。分割（分割入金型）と、明細の無い古いデータでは `null`（算出できない） |
| `surveyAnswers[]` | アンケート回答。`question` と `answer`（チェックボックスは `,` 区切り、未回答は空文字）。無ければ空配列。電話番号・性別などの個人情報を含みうる |

`orders` の金額・手数料・差額の内訳は MOSH 内部の計算に由来する。ユーザーには `amount`（金額）と `paidAmount`（支払済み）を中心に示し、`differenceAmount` の意味を推測して説明しない。

## 購読 1 件の詳細（`getCreatorProductSubscriber`）

一覧の項目すべてに加えて次が返る。

| 項目 | 意味 |
|---|---|
| `plan` | プランの価格情報（取得できないと `null`）。`price`（プラン価格）、`initialAmount`（初回金額。無ければ `null`）、`iterations`（支払い回数。無期限なら `null`）、`currency`、`paymentInterval`（`month` 月額 / `year` 年額 / `once`） |
| `currentPeriodEnd` | 現在の請求期間の終了日時（次回請求の目安） |
| `billingStartDate` | 課金開始日の指定があるときの初回請求日。指定が無ければ `null` |
| `coupon` | 申込時に使ったクーポン。申込の詳細と同じ形 |
| `surveyAnswers[]` | アンケート回答。申込の詳細と同じ形 |

過去の毎月の請求明細は返らない。

## 表示名への変換表（生 ID → 何で引くか）

| 生の値 | 表示 | 変換元 |
|---|---|---|
| `productId` | 商品名 | `getCreatorProducts` の `name` |
| `planId` | プラン名 | レスポンスの `planName`（空文字なら `getCreatorProductPlans`） |
| `guest.id` | MOSH ID | 変換しない。ユーザーが求めたときだけ「MOSH ID」として出す |
| `status` | 状態名 | 上の変換表 |
| `paymentType` | 一括 / 分割（分割入金型）/ 分割（一括入金型） | 上の説明 |
| `order.paymentMethod` / `orders[].paymentMethod` | カード / 現金 / チケット / 銀行振込 | `card` / `cash` / `ticket` / `bank-transfer` |
| 申込 ID / 購読 ID / 注文 ID / クーポン ID | 出さない | 詳細ツールの呼び出しにだけ使う |
| 金額 | 円（3 桁区切り） | `currency` が `JPY` 以外のときは通貨を添える |
| 日時 | 日本時間の年月日 | ISO 8601 のオフセット付き値を変換する |

## 取得できないもの

- 一覧ツールの金額（金額は 1 件の詳細でだけ返る。期間・商品全体の売上集計は `sales-reporter` スキル）
- サブスクの過去の請求明細
- 返金できるかどうか
- 申込のキャンセル・返金・購読の解約などの操作
- ゲストのプロフィール項目（電話番号・住所など）。アンケートで聞いている場合だけ `surveyAnswers` に入る

## 絶対に避けること

- 依頼された用途以外に `guest.name` / `guest.email` / `surveyAnswers` / `message` を出力・転記・要約する
- `status` や `paymentType`、絞り込みに使った `paymentPattern` の生の値をそのままユーザーに見せる
- 「決済済み」と「反映待ち（`unapproved`）」を混同する
- 詳細ツールで取っていない金額や支払い明細を推測して答える
