# ワークフローコンテンツ構造リファレンス

`patchCreatorScenario` の `stages` フィールドに渡す JSON の詳細仕様（プロダクト呼称は「ワークフロー」、API / JSON のキー名は `scenario*`）。

## ライフサイクル

```
INACTIVE  : 停止中（作成直後 / 一時停止）
ACTIVE    : 有効（稼働中）
ARCHIVED  : アーカイブ済（事実上の廃止）
```

`getCreatorScenarios` のクエリ enum だけ別物で、`"INACTIVE_ACTIVE"` / `"ARCHIVED"` の2値のみ。

### ライフサイクルの遷移

作成直後は必ず `INACTIVE`。`INACTIVE` / `ACTIVE` / `ARCHIVED` 間の遷移は `patchCreatorScenariosLifecycle` で行う。`ACTIVE` 化（公開）時のみ、参照リソースの実在と同時稼働数の上限が確認され、満たさない場合は理由を示すエラーが返る。

## stages の全体構造

```json
{
  "stages": [
    {
      "trigger": { /* triggerType と1つの実フィールド + 残り全て null */ },
      "action":  { /* actionType と1つの実フィールド + 残り全て null */ }
    },
    {
      "action":  { ... }
    }
  ]
}
```

通常パターンは「先頭 stage のみ `trigger` を持ち、以降は `trigger` フィールド自体を省略する」。`trigger: null` は Zod バリデーションで弾かれるため使用禁止。

## trigger の構造

### triggerType と対応サブフィールド

| triggerType | 対応する非 null サブフィールド | サブフィールドの形 |
|---|---|---|
| `MARKETING_LEAD_BENEFIT_RECEIVED` | `marketingLeadBenefitReceivedTrigger` | `{ benefitId: string }` |
| `LINE_CHANNEL_CONTACT_REGISTERED` | なし（サブフィールド不要） | LINE公式アカウントはリクエストボディ最上位の `creatorLineChannelId` で指定 |
| `SERVICE_APPLIED` | `serviceAppliedTrigger` | `{ serviceId: string, paymentMethod: "CASH"\|"CARD"\|"BANK_TRANSFER"\|null }` |
| `SERVICE_SCHEDULE_REMINDER` | `serviceScheduleReminder` | `{ serviceId: string, remindTimeType: "RELATIVE"\|"ABSOLUTE", beforeDays: 0-30, hours: 0-23\|null, minutes: 0-59\|null }` |
| `CONTACT_TAG_ADDED` | `contactTagAdded` | `{ contactTagId: number }` |
| `INFLOW_ACTION_CONVERTED` | `inflowActionConverted` | `{ inflowActionId: number }` |
| `INSTALLMENT_PAYMENT_FAILED` | なし（サブフィールド不要） | 詳細フィールドは不要。同一分割回の決済リトライ失敗では再発火しない |
| `SUBSCRIPTION_PAYMENT_FAILED` | なし（サブフィールド不要） | 詳細フィールドは不要。初回失敗のみ発火し、同一請求期間内の決済リトライ失敗では再発火しない |

### trigger テンプレート（必ず6フィールド全部指定）

`ScenarioTrigger` 型のサブフィールドは `triggerType` + 5つ。`lineChannelContactRegisteredTrigger` は型定義に存在しないため指定禁止。

```json
{
  "triggerType": "MARKETING_LEAD_BENEFIT_RECEIVED",
  "marketingLeadBenefitReceivedTrigger": { "benefitId": "<benefit_id>" },
  "serviceAppliedTrigger": null,
  "serviceScheduleReminder": null,
  "contactTagAdded": null,
  "inflowActionConverted": null
}
```

`LINE_CHANNEL_CONTACT_REGISTERED` の場合はサブフィールドが全て `null`。LINE公式アカウントはボディ最上位の `creatorLineChannelId` で指定する:

```json
{
  "triggerType": "LINE_CHANNEL_CONTACT_REGISTERED",
  "marketingLeadBenefitReceivedTrigger": null,
  "serviceAppliedTrigger": null,
  "serviceScheduleReminder": null,
  "contactTagAdded": null,
  "inflowActionConverted": null
}
```

`INSTALLMENT_PAYMENT_FAILED` / `SUBSCRIPTION_PAYMENT_FAILED` も同様にサブフィールドが全て `null`（前提となる既存リソースの確認も不要。トリガー自体に ID を持たない）。

## action の構造（POST 用 = `scenarioAction`）

### actionType と対応サブフィールド

| actionType | 対応する非 null サブフィールド | サブフィールドの形 |
|---|---|---|
| `SEND_EMAIL` | `sendEmail` | `{ subject: string(1-50), message: string, isTrackingEnabled: boolean }` |
| `SEND_LINE_MESSAGE` | `sendLineMessage` | `{ type, messages, isTrackingEnabled }`（後述） |
| `WAIT_TIME` | `waitTime` | `{ waitTimeType, actionAfterDays: 0-30, actionHours: 0-23\|null, actionMinutes: 0-59\|null }` |
| `CONDITION` | `condition` | `{ conditionType, branches[], conditionXxx }`（後述） |
| `ADD_CONTACT_TAG` | `addContactTag` | `{ contactTagId: number }` |
| `REMOVE_CONTACT_TAG` | `removeContactTag` | `{ contactTagId: number }` |
| `LINK_LINE_RICH_MENU` | `linkLineRichMenu` | `{ lineRichMenuId: number }` |
| `UNLINK_LINE_RICH_MENU` | （なし） | 全詳細フィールド `null` |

### action テンプレート（必ず8フィールド全部指定）

```json
{
  "actionType": "SEND_EMAIL",
  "sendEmail": {
    "subject": "ウェルカムメッセージ",
    "message": "ご登録ありがとうございます。",
    "isTrackingEnabled": true
  },
  "sendLineMessage": null,
  "waitTime": null,
  "condition": null,
  "addContactTag": null,
  "removeContactTag": null,
  "linkLineRichMenu": null
}
```

## actionType 別: 個別フィールド詳細

### SEND_LINE_MESSAGE: `sendLineMessage`

`sendLineMessage` はトップレベルに `messages` の**1フィールドのみ**（`type` や `isTrackingEnabled` を
トップレベルに置かない）。`messages` は **1〜5件の配列**で、各要素が自分自身の `messageType`
（大文字・種別を判別するリテラル）を持つオブジェクト。**`messages: ["文字列", ...]` のような
プレーン文字列配列は誤り**（TEXTタイプであっても、必ず `{ messageType: "TEXT", text, ... }` の
オブジェクトを1件ずつ並べる）。

```json
{
  "messages": [
    {
      "messageType": "TEXT",
      "text": "こんにちは！",
      "isTrackingEnabled": false,
      "urlActions": null
    },
    {
      "messageType": "TEXT",
      "text": "もう一通テキストを送ります",
      "isTrackingEnabled": false,
      "urlActions": null
    }
  ]
}
```

`messages[]` 各要素の形（`messageType` で判別。5種類）:

| messageType | 対応フィールド | 制約 |
|---|---|---|
| `TEXT` | `text: string(1-5000)`, `isTrackingEnabled: boolean`, `urlActions: UrlAction[] \| null` | 1メッセージにつき本文1件（配列ではない） |
| `CAROUSEL` | `altText: string(1-50)`, `carousels: CarouselItem[]` | carousels は1〜4件 |
| `IMAGE_CAROUSEL` | `altText: string(1-50)`, `imageCarousels: ImageCarouselItem[]` | imageCarousels は1〜4件 |
| `VIDEO` | `muxAssetId: number`, `previewMoshImageId: string` | — |
| `RICH_MESSAGE` | `altText`, `imageUrl`, `imageWidth(300-1024)`, `imageHeight(300-1024)`, `splitPattern`, `cells: RichMessageCell[]` | 1枚画像を分割しマスごとにタップ設定。cells の要素数は splitPattern に対応（後述） |

いずれのタイプも `messages` 配列の**要素数は最大5件**（同じ `messageType` に限らず、TEXTとCAROUSEL等を混在させてもよい）。

**TEXT の `urlActions`**（本文中のURLタップで顧客タグを操作したい場合のみ設定。不要なら `null`）:
```json
"urlActions": [
  {
    "url": "https://example.com/guide",
    "postbackActions": [
      { "postbackActionType": "addContactTag", "contactTagId": 1 }
    ]
  }
]
```
- `url` — 本文中に含まれるURLと完全一致でキーにする
- `postbackActions` — 1〜3件。`postbackActionType` は `"addContactTag"` | `"removeContactTag"`（+ `contactTagId`）

CarouselItem の形（`postbackActions` は任意。設定しない場合は `null`）:
```json
{
  "imageUrl": "https://...",
  "title": "タイトル(1-40)",
  "text": "本文(1-60)",
  "buttonText": "ボタン(1-20)",
  "buttonUrl": "https://...（設定しない場合は null）",
  "postbackActions": null
}
```
- `postbackActions` — ボタンタップ時に実行するアクション（1〜3件、任意。設定しない場合は `null`）。
  `postbackActionType` は `"sendMessage"`（+ `messages: [{ messageType: "text"（小文字）, message: string }]` で自動返信）
  | `"addContactTag"` | `"removeContactTag"`（+ `contactTagId`）。
  ⚠ `sendMessage` の内側の `messageType` は**小文字 `"text"`**（トップレベルの `messages[].messageType` の
  大文字 `"TEXT"` とは綴りが異なるので混同しない）。

ImageCarouselItem の形（`postbackActions` の仕様は CarouselItem と同じ）:
```json
{
  "image": "https://...",
  "url": "https://...（設定しない場合は null）",
  "postbackActions": null
}
```

RichMessageCell の形（`postbackActions` の仕様は CarouselItem と同じ。`cells` の要素数は
`splitPattern` に対応: `ONE_BLOCK`=1 / `LEFT_RIGHT_SPLIT`=2 / `TOP_BOTTOM_SPLIT`=2 / `GRID_FOUR`=4。
マスは左上から右下への行優先順）:
```json
{
  "url": "https://...（設定しない場合は null）",
  "postbackActions": null
}
```

### WAIT_TIME: `waitTime`

```json
{
  "waitTimeType": "RELATIVE",
  "actionAfterDays": 3,
  "actionHours": 10,
  "actionMinutes": 0
}
```

**起点は常に「直前ステップの実行時刻」**（トリガー発火時刻ではない）。`RELATIVE` / `ABSOLUTE` いずれも起点は同じで、`actionHours` / `actionMinutes` の解釈だけが異なる。

- `actionAfterDays`（0-30）は両タイプ共通の相対加算日数（起点に N 日を足す）。
- `waitTimeType: "RELATIVE"` — `actionHours` / `actionMinutes` は起点の時刻に**加算する**時間・分（「起点から N日 H時間 M分 後」）。`null` のフィールドは加算なし。
- `waitTimeType: "ABSOLUTE"` — `actionAfterDays` 日後の日付に、時刻を `actionHours:actionMinutes` へ**セット**する（クロック時刻指定）。`actionHours` と `actionMinutes` のどちらかでも `null` なら時刻セットは行われず、起点の時刻がそのまま維持される。
- `actionHours` / `actionMinutes` は `nullable: true`（時刻指定なしも可）。
- ⚠️ 計算結果の実行時刻が現在より過去になると、後続ステップはスケジュールされずそこで停止する（ユーザーへの通知は無い）。「配信されない」という相談の典型パターンなので、待機日数・時刻の設定はユーザーの意図（起点からの相対か、特定の時刻に揃えたいか）を確認してから `RELATIVE` / `ABSOLUTE` を選ぶ。
- 待機0（結果が起点と同時刻になる設定）は即時実行される。

### CONDITION: `condition`

`conditionType` に応じてサブフィールドが切り替わるネスト構造。

```json
{
  "conditionType": "CONTACT_TAG",
  "branches": [
    { "matchValue": true,  "stages": [ { "action": { ... } } ] },
    { "matchValue": false, "stages": [ { "action": { ... } } ] }
  ],
  "conditionServiceApplicationStatus": null,
  "conditionContactTag": { "contactTagId": 1 },
  "conditionAutoWebinarParticipation": null,
  "conditionAutoWebinarWatchTime": null,
  "conditionBankTransferStatus": null
}
```

`condition` のサブフィールドは `conditionType` + 5つ（全て指定必須。対応する1つだけ実オブジェクト、残り4つは `null`）。

| conditionType | 対応する非 null サブフィールド | 形 |
|---|---|---|
| `SERVICE_APPLICATION_STATUS` | `conditionServiceApplicationStatus` | `{ serviceId: string, serviceApplicationStatus: "APPLIED" }` |
| `CONTACT_TAG` | `conditionContactTag` | `{ contactTagId: number }` |
| `AUTO_WEBINAR_PARTICIPATION` | `conditionAutoWebinarParticipation` | `{ autoWebinarId: number }`（オートウェビナーへの参加有無。ID は `getCreatorAutoWebinars` で取得可） |
| `AUTO_WEBINAR_WATCH_TIME` | `conditionAutoWebinarWatchTime` | `{ autoWebinarId: number, metricType: "WATCH_POSITION"\|"TOTAL_PLAY_TIME", thresholdSeconds: number(0-86400) }`（オートウェビナーの視聴時間条件。`WATCH_POSITION`=到達した最大視聴位置、`TOTAL_PLAY_TIME`=視聴した総再生時間。`thresholdSeconds`は「以上」で判定。ID は `getCreatorAutoWebinars` で取得可） |
| `BANK_TRANSFER_STATUS` | `conditionBankTransferStatus` | `{ serviceId: string, bankTransferStatus: "PENDING"\|"COMPLETED"\|"REJECTED"\|"CANCELED" }`（対象ゲストの指定サービスへの注文が、指定した銀行振込の振込状況に一致するかで分岐。`PENDING`=振込待ち・未入金、`COMPLETED`=振込完了・入金の反映待ちを含む、`REJECTED`=振込期限切れ、`CANCELED`=キャンセル。クレジットカード等、銀行振込以外の支払い方法の申し込みは常に「一致しない」側に進む） |

`branches` の各要素:
- `matchValue: boolean` — 条件にマッチしたら true 側、外れたら false 側を実行
- `stages: Stage[]` — その分岐で実行する一連のステージ（再帰構造）

### ADD_CONTACT_TAG / REMOVE_CONTACT_TAG

```json
{ "contactTagId": 1 }
```

## PATCH 専用: `scenarioActionForUpdate` 差分

`patchCreatorScenario` の `stages[].action` は **`scenarioActionForUpdate`** 型を使う。差分は **`condition` フィールドだけ**:

- POST 用: `condition` は `scenarioActionCondition`
- PATCH 用: `condition` は `scenarioActionConditionForUpdate`

両者の違いは `branches[].stages[].action` の型:

- POST 側 (`scenarioActionConditionBranch`): `action` も `scenarioAction` で完全な型定義
- PATCH 側 (`scenarioActionConditionBranchForUpdate`): `action` は `additionalProperties: true` の汎用オブジェクト扱い（循環参照を避けるため。サーバー側で再帰 zod 検証）

PATCH で分岐内アクションを組むときは、型エラーが出にくいぶん**仕様逸脱の検知が遅れる**ことに留意。content-schema の制約は同等に満たす必要がある。

## ID の型一覧

| フィールド | 型 | 例 |
|---|---|---|
| `id`（ワークフロー・ステージ・トリガー・アクション） | integer | `<scenario_id>` |
| `serviceId` | **string**（数値文字列） | `"<service_id>"` |
| `benefitId` | **string**（数値文字列） | `"<benefit_id>"` |
| `contactTagId` | integer | `<contact_tag_id>` |
| `inflowActionId` | integer | `<inflow_action_id>` |
| `creatorLineChannelId` | integer | `<creator_line_channel_id>` |
| `autoWebinarId` | integer | `<auto_webinar_id>` |
| `muxAssetId` | integer | `<mux_asset_id>` |
| `previewMoshImageId` | string | `"<preview_mosh_image_id>"` |
| `handle`（stage） | string (uuid) | `"<stage_handle_uuid>"` |

## 埋め込み変数（差し込み文字）

`SEND_EMAIL` / `SEND_LINE_MESSAGE` の本文には `{{variable_name}}` 形式（二重中括弧）の埋め込み変数を使える。**Excel等の外部ソースにある `%foo%` のようなプレースホルダ記法はMOSHでは解釈されない**（そのまま文字列として残るだけ）ので、本文を外部ドキュメントから転記する際は必ず `{{...}}` 形式に置き換えること。

### 対応変数一覧（この5つ以外は存在しない）

| 変数名 | 意味 / データソース | 使える条件 |
|---|---|---|
| `line_name` | コンタクトのLINEプロフィール表示名 | `actionType: SEND_LINE_MESSAGE` の時のみ（`SEND_EMAIL`では常に空文字） |
| `guest_name` | ゲスト（Moshユーザー）の名前 | `actionType: SEND_EMAIL` かつ `triggerType` が `SERVICE_APPLIED` / `SERVICE_SCHEDULE_REMINDER` / `INSTALLMENT_PAYMENT_FAILED` / `SUBSCRIPTION_PAYMENT_FAILED` の時のみ。それ以外は空文字 |
| `service_name` | トリガーに紐づくプラン・サービス名 | `triggerType` が `SERVICE_APPLIED` / `SERVICE_SCHEDULE_REMINDER` の時のみ。`INSTALLMENT_PAYMENT_FAILED` / `SUBSCRIPTION_PAYMENT_FAILED` はトリガー自体にプラン・サービス参照を持たないため対象外。それ以外のトリガー（`MARKETING_LEAD_BENEFIT_RECEIVED` / `LINE_CHANNEL_CONTACT_REGISTERED` / `CONTACT_TAG_ADDED` / `INFLOW_ACTION_CONVERTED` / `INSTALLMENT_PAYMENT_FAILED` / `SUBSCRIPTION_PAYMENT_FAILED`）では常に空文字 |
| `reservation_time_range` | 予約日時の範囲（`YYYY年M月D日 HH:mm〜HH:mm`, JST） | `SERVICE_SCHEDULE_REMINDER`は常に対応。`SERVICE_APPLIED`はプラン・サービスの`serviceType`が「予約(event)」または「個別(private)」の場合のみ（コンテンツ/サブスク/オンライン単体のプラン・サービスでは空文字） |
| `zoom_url` | ZoomのjoinURL | `reservation_time_range`と同条件に加えて、プラン・サービスの`locationType`がオンライン/ハイブリッドかつクリエイターがZoom連携済みの場合のみ。条件を満たさない場合は静かに空文字になる（保存・送信はブロックされない） |

未対応のキー（例: 存在しない変数名や`SERVICE_APPLIED`以外での`service_name`利用）は**空文字ではなくプレースホルダ文字列がそのまま残る**か、状況によっては空文字になる（trigger種別で`context`自体が無い場合）。いずれにせよ意図通りに差し込まれるとは限らないため、使う前に上表の条件を必ず確認する。

## 流入経路 / トラッキング同意

- 流入経路 (`inflowAction`) は LINE公式アカウントに紐づく URL を発行する仕組み。`scenarioTriggerInflowActionConverted` の `inflowActionId` で参照
- トラッキング同意は API 利用前提として `postCreatorScenariosTrackingConsent` で記録。`getCreatorScenariosTrackingConsents` で同意状態取得

## INACTIVE 状態の PATCH 検証

`INACTIVE` のワークフローに対する `patchCreatorScenario` の検証は緩い:

- **参照リソース ID の実在検証は行われない** — 例えば存在しない `benefitId` を入れても 200 OK で保存される
- 「保存できた」を「動く」と思い込まないこと。ユーザーに `benefitId` / `serviceId` / `contactTagId` 等が実在の管理画面リソースに対応しているかを口頭で確認する
