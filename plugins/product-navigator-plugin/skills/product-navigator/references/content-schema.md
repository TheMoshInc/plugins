# 商品・プラン レスポンス構造リファレンス

各ツールが返す JSON の読み方。提示はここの変換表を通してから行う。

## 商品（`product`）の主な項目

| 項目 | 意味・提示時の扱い |
|---|---|
| `id` | 商品 ID。プラン取得の `productId` に使う。**提示しない** |
| `slug` | 公開 URL 用の識別子。**提示しない**（URL は `publicUrl` を使う） |
| `name` | 商品名（1〜100 文字）。提示の主キーにする |
| `description` | 商品説明。**リッチテキストの内部形式（Tiptap JSON）**。本文テキストを取り出して要約する |
| `imageIds` | 商品画像の ID 一覧（最大 7 件）。**提示しない**（枚数の言及は可） |
| `precautions` | 注意事項。**編集画面ではリッチテキストで保存されるため、Tiptap JSON 形式ならば本文テキストを取り出して提示する** |
| `cancellationPolicy` | キャンセルポリシー。同上（Tiptap JSON 形式ならば本文テキストへ変換） |
| `publishingStatus` | 公開ステータス（下の変換表） |
| `steps` | 申し込み後の流れ。`title` / `description` を提示する。`steps[].id` / `steps[].imageId` は**提示しない**（画像は枚数の言及のみ可） |
| `isDeletable` | 削除可能か（配下プランにサブスク購読中またはアクティブな購入者がいると `false`） |
| `publicUrl` | ゲスト向け商品ページの URL。**非公開（`PRIVATE`）ではページが存在しないため `null`（正常）**。ただし旧サービス扱いの商品（`isLegacyService: true`）は公開ステータスに関係なく**サービスページの URL** が入り、サービスが非公開なら `null`（正常）。つまり旧サービスで URL が入っていれば、サービスページは公開中で閲覧できる。商品が `PRIVATE` でも「非公開なので閲覧できない」と言わない |
| `planCount` | 紐づくプラン件数 |
| `isLegacyService` | 旧サービス扱いの商品か。`true` の商品（イベント・プライベート・Online のプランを持つもの）は商品ページではなくサービスページで設定・販売されている。提示では「旧サービス（サービスページで販売中）」と添え、英字の項目名は出さない |
| `createdAt` / `updatedAt` | 作成・更新日時 |

## 商品プラン（`productPlan`）の主な項目

| 項目 | 意味・提示時の扱い |
|---|---|
| `id` | プラン ID。プラン詳細の `productPlanId` に使う。**提示しない** |
| `slug` | ゲスト向けプラン詳細 URL 用の識別子（UUID）。**提示しない** |
| `productId` | 親商品の ID。**提示しない** |
| `moshServiceId` | 内部のサービス ID。**常に提示しない** |
| `name` | プラン名（1〜50 文字）。提示の主キーにする |
| `description` | プラン説明（最大 10,000 文字） |
| `orderNum` | 表示順序 |
| `publishingStatus` | 公開設定（下の変換表） |
| `capacity` / `remainingCapacity` | 定員 / 残り枠。**`capacity: 0` は「定員なし（無制限）」を表し、「定員 0 人」ではない**（このとき `remainingCapacity` は `null`）。管理画面と同じく「定員なし」と表示する。申込人数は返らないため、定員と残り枠の差から「◯人申し込み済み」と逆算しない |
| `showRemaining` / `remainingThreshold` | 残席数表示の有無 / 表示閾値 |
| `price` | `{ amount, currency }`。`JPY` なら「◯◯円」と表記 |
| `paymentMethods` | 支払い方法と有効/無効（下の変換表）。`isEnabled: false` のものは「利用不可」として扱う |
| `bankTransferDeadlineDays` | 銀行振込の振込期限（申込からの日数）。`null` は標準の 15 日。銀行振込が無効なプランでは使われないので提示しない |
| `billingCycle` | 課金サイクル（下の変換表） |
| `iterations` | 課金回数（サブスクリプションのみ。`null` は無制限） |
| `benefits` | 特典内容（サブスクリプションのみ） |
| `installmentType` / `installmentIterations` | 分割払いタイプ / 分割プラン設定 |
| `initialPrice` | 初回価格（サブスクリプションの一部のプランだけで使われる）。`null` は「初回も通常価格と同額」として提示する。理由として決済の仕組みの名前は出さない |
| `discountPrice` | 割引前の価格（`null` は割引なし） |
| `isSuspended` | 販売受付停止フラグ（`true` は公開のまま受付のみ停止） |
| `isDeletable` | プランを削除できるか（サブスク購読中のゲストがいると `false`） |
| `isNameEditable` / `isPriceEditable` / `isBillingStartEditable` | プラン名・価格・課金開始日を変更できるか。サブスクは「非公開かつ購読者（解約済み含む）がいない」ときだけ `true`。買い切りは名前・価格が常に `true`、課金開始日は常に `false`。変更の依頼を引き継ぐときに「このプランは購読者がいるため価格を変えられない」等を添える |
| `billingStartType` | 課金開始日の指定方法（サブスクのみ。下の変換表） |
| `billingStartAbsoluteDate` | 課金開始日（`billingStartType` が `absolute` のときのみ。それ以外は `null`）。プランの設定値であり、各ゲストの実際の初回課金日ではない（「◯◯さんの初回課金日は」と言い切らない） |
| `billingStartRelativeOffsetDays` | 購入から課金開始までの日数（`relative` のときのみ。それ以外は `null`） |
| `applicationStartDateTime` / `applicationEndDateTime` | 販売受付の開始・終了日時（買い切りプラン用。`null` は制限なし） |
| `applicationPeriods` | 申込期間設定（複数・繰り返しあり）。繰り返しの読み方は下記 |
| `salesStatus` | 販売状態（下の変換表） |
| `membershipSites` | 紐づく会員サイトの要約（`id` と `name`）。**`name` で提示し、`id` は出さない** |

### 申込期間（`applicationPeriods`）の読み方

- `recurringFrequency`: `NONE`=繰り返しなし / `DAILY`=日毎 / `WEEKLY`=週ごと / `MONTHLY`=月ごと
- `recurringInterval`: 繰り返し間隔。**必ず提示に反映する**（例: `WEEKLY` + `2` = 「2 週ごと」。1 でも「毎週」等と明示）
- `recurringWeekOfMonth`: 第何週（1〜5、`MONTHLY` 時のみ）。**必ず提示に反映する**（例: `[1, 3]` = 「第 1・第 3」）
- `recurringDaysOfWeek` は 1=月曜〜7=日曜
- 繰り返しあり時、`startDateTime` / `endDateTime` は 1 回分の受付時間帯を表し、繰り返しの範囲は `recurringEndDateTime`（`null` は無期限）
- 組み合わせ例: `MONTHLY` + `recurringInterval: 1` + `recurringWeekOfMonth: [1,3]` + `recurringDaysOfWeek: [1]` = 「毎月第 1・第 3 月曜」

## 販売状況（`getCreatorProductSalesStatus`）の項目

管理画面の商品詳細に出るカードと同じ値。ラベルも管理画面に合わせる（どの項目を出すかは SKILL.md「4. 販売状況の取得」）。

| 項目 | 表示ラベル | 意味 |
|---|---|---|
| `paymentAmount` | 決済金額（今月分） | 今月決済された金額（円）。全プラン合算。分割払いの 2 回目以降やサブスクの月次課金も含む |
| `applicantsCount` | 申し込み者（今月分） | 今月の買い切りプランのユニーク申込者数。**サブスクの申込は含まない** |
| `subscribersCount` | サブスク購読者（現時点） | 今この時点で購読中（お試し中を含む）の人数 |

- **決済ベース**である。管理画面の「売上一覧」や `sales-reporter` スキルの売上（申込ベース：分割払いは申込月に満額を計上し、カード決済のキャンセル・期限切れも計上に残る）とは一致しない。例: 12 回分割の申込は、こちらでは毎月その回の支払い分だけが入る
- 値はそのまま提示し、再計算・合算・他の数値との足し引きをしない
- 誰が申し込んだか・購読しているかの一覧は返らない（`applicant-reporter` スキルの領域）

## enum の日本語変換表

| 項目 | 値 → 表示 |
|---|---|
| `publishingStatus` | `PUBLIC`=公開 / `LIMITED`=限定公開 / `PRIVATE`=非公開 |
| `billingCycle` | `ONE_TIME`=買い切り / `MONTHLY`=月額 / `YEARLY`=年額 |
| `paymentMethods[].method` | `CARD`=クレジットカード / `BANK_TRANSFER`=銀行振込 |
| `salesStatus` | `forSale`=受付中 / `notForSale`=受付停止中 / `soldOut`=満員 / `beforeApplicationPeriod`=受付開始前 / `afterApplicationPeriod`=受付終了 / `outOfApplicationPeriod`=受付期間外 |
| `billingStartType` | `immediate`=購入後すぐに初回課金 / `absolute`=指定日（`billingStartAbsoluteDate`）に課金開始 / `relative`=購入から `billingStartRelativeOffsetDays` 日後に課金開始 |
| `installmentType` | `LUMP_SUM`=一括払い / `INSTALLMENT_LUMP_DEPOSIT`=一括払い または 分割払い（一括入金） / `INSTALLMENT`=一括払い または 分割払い（分割入金） |

## 絶対に避けること（提示禁止）

本ドキュメント各所の制約のうち、ユーザーへの提示で必ず守るものを 1 箇所に集約する。

- 生の内部 ID・識別子（`id` / `slug` / `productId` / `productPlanId` / `moshServiceId` / `imageIds` / `steps[].id` / `steps[].imageId` / `membershipSites[].id`）を提示しない
- `description` の内部形式（Tiptap JSON）をそのまま表示しない
- enum の英字値（`PUBLIC` / `forSale` 等）をそのまま表示しない
- `publicUrl: null` を「エラー」「取得失敗」と説明しない（非公開商品、または非公開の旧サービスの正常な値）
- 購入者数・売上など、レスポンスに無い実績データを推測で補わない（引き継ぎ先は SKILL.md「When NOT to use」）
- 販売状況の「決済金額」を「売上」と言い換えない（理由は「販売状況」節）
