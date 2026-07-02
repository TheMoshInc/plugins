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
| `LINE_CHANNEL_CONTACT_REGISTERED` | なし（サブフィールド不要） | LINE チャンネルはリクエストボディ最上位の `creatorLineChannelId` で指定 |
| `SERVICE_APPLIED` | `serviceAppliedTrigger` | `{ serviceId: string, paymentMethod: "CASH"\|"CARD"\|"BANK_TRANSFER"\|null }` |
| `SERVICE_SCHEDULE_REMINDER` | `serviceScheduleReminder` | `{ serviceId: string, remindTimeType: "RELATIVE"\|"ABSOLUTE", beforeDays: 0-30, hours: 0-23\|null, minutes: 0-59\|null }` |
| `CONTACT_TAG_ADDED` | `contactTagAdded` | `{ contactTagId: number }` |
| `INFLOW_ACTION_CONVERTED` | `inflowActionConverted` | `{ inflowActionId: number }` |

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

`LINE_CHANNEL_CONTACT_REGISTERED` の場合はサブフィールドが全て `null`。LINE チャンネルはボディ最上位の `creatorLineChannelId` で指定する:

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

`type` に応じて `messages` の形が変わる。

```json
{
  "type": "TEXT",
  "messages": ["こんにちは！", "もう一通テキストを送ります"],
  "isTrackingEnabled": false
}
```

| type | messages の形 | 制約 |
|---|---|---|
| `TEXT` | `string[]` | 1〜5要素、各1〜5000文字 |
| `CAROUSEL` | `{ altText: string(1-50), carousels: CarouselItem[] }` | carousels は1〜4件 |
| `IMAGE_CAROUSEL` | `{ altText: string(1-50), imageCarousels: { image: uri, url: uri }[] }` | imageCarousels は1〜4件 |
| `VIDEO` | `{ muxAssetId: number, previewMoshImageId: string }` | — |

CarouselItem の形:
```json
{
  "imageUrl": "https://...",
  "title": "タイトル(1-40)",
  "text": "本文(1-60)",
  "buttonText": "ボタン(1-20)",
  "buttonUrl": "https://...",
  "openExternalBrowser": false
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

- `RELATIVE`: 前ステップ実行時から `actionAfterDays` 日後の `actionHours:actionMinutes` に次へ進む
- `ABSOLUTE`: 絶対指定（仕様は実作業で確認・追記する）
- `actionHours` / `actionMinutes` は `nullable: true`（時刻指定なしも可）

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
  "conditionAutoWebinarParticipation": null
}
```

| conditionType | 対応する非 null サブフィールド | 形 |
|---|---|---|
| `SERVICE_APPLICATION_STATUS` | `conditionServiceApplicationStatus` | `{ serviceId: string, serviceApplicationStatus: "APPLIED" }` |
| `CONTACT_TAG` | `conditionContactTag` | `{ contactTagId: number }` |
| `AUTO_WEBINAR_PARTICIPATION` | `conditionAutoWebinarParticipation` | `{ autoWebinarId: number }` |

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

## 流入アクション / トラッキング同意

- 流入アクション (`inflowAction`) は LINE チャンネルに紐づく URL を発行する仕組み。`scenarioTriggerInflowActionConverted` の `inflowActionId` で参照
- トラッキング同意は API 利用前提として `postCreatorScenariosTrackingConsent` で記録。`getCreatorScenariosTrackingConsents` で同意状態取得

## INACTIVE 状態の PATCH 検証

`INACTIVE` のワークフローに対する `patchCreatorScenario` の検証は緩い:

- **参照リソース ID の実在検証は行われない** — 例えば存在しない `benefitId` を入れても 200 OK で保存される
- 「保存できた」を「動く」と思い込まないこと。ユーザーに `benefitId` / `serviceId` / `contactTagId` 等が実在の管理画面リソースに対応しているかを口頭で確認する
