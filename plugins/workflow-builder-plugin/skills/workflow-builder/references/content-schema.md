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

## 検査の二層構造: 構文解析 → 意味解析

ワークフローの JSON を点検するときは、コンパイラと同じ順序で**構文 → 意味**の二層に分けて見る。順序を守る理由は、構文が壊れている JSON に意味の検査を当てても結果が信用できないため。

二層の分岐点は「API が弾くか」ではなく、**判定結果が個々の対象者（コンタクト／ゲスト）のデータに依存するか**。API が通してしまう設定でも、対象者が誰であっても一律に失敗する・意図と異なる動作になるものは構文側に置く。同一テナント内の設定状態（参照リソースの実在・他ワークフローの公開状態）は対象者ごとに変わらないので、これだけで決まるものも構文側に含める。

| 層 | 見るもの | 弾かれるタイミング |
|---|---|---|
| **構文解析**（字句解析を含む） | ワークフロー単体の JSON 構造、または同一テナント内の設定状態だけから、**対象者が誰であっても一律に**判定できるもの。①フィールド型・enum 値・「対応する 1 サブフィールドだけ実オブジェクト、残りは `null`」の規律・ID の型・`messages[]` の形（本ファイル「trigger の構造」〜「ID の型一覧」）②木の整形性＝WAIT_TIME 終端禁止・両 branch 空禁止・CONDITION はそれを含む stages 配列（トップレベル／branch 内とも）の最後の要素であること＝合流構成の禁止（[best-practices.md](best-practices.md) の「ノード接続ルール」）③参照整合性＝参照先の実在、流入経路・リッチメニューが属する LINE 公式アカウントとワークフローの一致、`urlActions[].url` と本文 URL の一致、`urlActions` と `isTrackingEnabled` の組み合わせ、`messages` 空配列（本ファイル「TEXT の `urlActions`」「INACTIVE 状態の PATCH 検証」＋ best-practices「トリガー × アクションの相性」「メッセージ内容の参照整合性」）④埋め込み変数の使用可否（「埋め込み変数」対応表。使えるかどうかは trigger × action の組み合わせだけで決まる）⑤同一ワークフロー内の順序矛盾（タグ付与前にそのタグで判定している構成は、判定時点でタグが必ず未付与なので構造上100%falseになる） | ①と③の一部（PATCH 時の参照 ID 実在検証・公開時の流入経路チャンネル一致）、および②の CONDITION 後続ステージ禁止（合流構成）は API が 400 で弾く。それ以外は API は弾かず、実行時に**対象者を選ばず一律に**失敗する（`urlActions` + `isTrackingEnabled: false`）か、エラーにならず一律に意図と異なる動作になる（両 branch 空・順序矛盾・URL 不一致・埋め込み変数の空文字）。弾かれなくても対象者非依存なので構文側 |
| **意味解析** | 個々の対象者の属性・その時点の状態で**結果が変わる**もの。①トリガーが供給する識別子と action / condition が要求する識別子の適合＝コンタクト↔MOSH ID の紐付け有無（後述「コンテキスト適合表」の △）②`SEND_EMAIL` の宛先＝メールアドレス保有（同表 ※4）③LINE 系ステップの対象＝該当 LINE 公式アカウントの友だち状態・ブロック（同表 ※5）④**テナント横断の充足性**＝①の紐付けが成立する入口（トラッキング有効 URL の配信）がテナント内の他ワークフロー・一斉配信に存在するか、`CONDITION` の一時点判定では拾えない「直後・いつでも」の意図を担う専用トリガーのワークフローが存在するか（「CONDITION は一時点の判定」節） | **保存・公開では弾かれない**。対象者ごとに成否が分かれ、失敗した対象者は `FAILED`（後続ステップも打ち切り）になる。構造だけでは「失敗しうる」までしか言えず、対象者の状態（紐付け・メール保有・友だち状態）をユーザーに確認して初めて判定できる。④はワークフロー単体では判定できず、テナント内の他のワークフロー・一斉配信を横断して裏取りする（具体的な調べ方は [best-practices.md](best-practices.md) の「単一ワークフローで閉じない要件」参照） |

> **確認済みの事実（2026-09-02）**: API スキーマは `triggerType` と `actionType` を独立した enum として持ち、trigger 種別によって置ける action 種別を**構造的には制限していない**（両者を突き合わせる相互制約のバリデーションは無い）。公開時の相互チェックも「LINE 関連の trigger / action があるならワークフローに `creatorLineChannelId` が必須」「流入経路が属する LINE 公式アカウントとワークフローの `creatorLineChannelId` が一致」の 2 点だけ。したがってトリガー × ステップの組み合わせはどれもスキーマを通り、動くかどうかは対象者の紐付け状態に依存する＝**意味解析**の領域。「コンテキスト適合表」で判定する。

## trigger の構造

### triggerType と対応サブフィールド

| triggerType | 対応する非 null サブフィールド | サブフィールドの形 |
|---|---|---|
| `MARKETING_LEAD_BENEFIT_RECEIVED` | `marketingLeadBenefitReceivedTrigger` | `{ benefitId: string }`。特典提供終了により**新規設定は不可**（既存ワークフローの読み取り・維持のみ） |
| `LINE_CHANNEL_CONTACT_REGISTERED` | なし（サブフィールド不要） | LINE公式アカウントはリクエストボディ最上位の `creatorLineChannelId` で指定 |
| `SERVICE_APPLIED` | `serviceAppliedTrigger` | `{ serviceId: string, paymentMethod: "CASH"\|"CARD"\|"BANK_TRANSFER"\|null }` |
| `SERVICE_SCHEDULE_REMINDER` | `serviceScheduleReminder` | `{ serviceId: string, remindTimeType: "RELATIVE"\|"ABSOLUTE", beforeDays: 0-30, hours: 0-23\|null, minutes: 0-59\|null }`。**1件の予約につき、トリガーが指す時刻に到達した瞬間の1回だけ評価される**。予約された時点でその時刻がすでに過去なら、その予約に対して二度と発火しない（後から追いつかない）。複数タイミングでリマインドしたい場合に `WAIT_TIME` で1本のワークフローへ繋いではいけない理由は [best-practices.md](best-practices.md) の「予約リマインドは複数のトリガーに分ける」参照 |
| `CONTACT_TAG_ADDED` | `contactTagAdded` | `{ contactTagId: number }` |
| `INFLOW_ACTION_CONVERTED` | `inflowActionConverted` | `{ inflowActionId: number }`（流入経路は LINE 公式アカウントに属する。ボディ最上位の `creatorLineChannelId` を流入経路が属するアカウントと一致させること） |
| `INSTALLMENT_PAYMENT_FAILED` | なし（サブフィールド不要） | 詳細フィールドは不要。同一分割回の決済リトライ失敗では再発火しない |
| `SUBSCRIPTION_PAYMENT_FAILED` | なし（サブフィールド不要） | 詳細フィールドは不要。初回失敗のみ発火し、同一請求期間内の決済リトライ失敗では再発火しない |

### trigger テンプレート（必ず6フィールド全部指定）

`ScenarioTrigger` 型のサブフィールドは `triggerType` + 5つ。`lineChannelContactRegisteredTrigger` は型定義に存在しないため指定禁止。

```json
{
  "triggerType": "CONTACT_TAG_ADDED",
  "marketingLeadBenefitReceivedTrigger": null,
  "serviceAppliedTrigger": null,
  "serviceScheduleReminder": null,
  "contactTagAdded": { "contactTagId": 1 },
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
| `SEND_EMAIL` | `sendEmail` | `{ subject: string(最大50字), message: string, isTrackingEnabled: boolean }` |
| `SEND_LINE_MESSAGE` | `sendLineMessage` | `{ messages }`（トップレベルは `messages` のみ。`messageType` / `isTrackingEnabled` は各要素内に置く。後述） |
| `WAIT_TIME` | `waitTime` | `{ waitTimeType, actionDate: date\|null, actionAfterDays: 0-30, actionHours: 0-23\|null, actionMinutes: 0-59\|null }` |
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
トップレベルに置かない）。`messages` は **最大5件の配列**（スキーマ上は0件も許容されるが下書き専用。稼働させるには1件以上必要）で、各要素が自分自身の `messageType`
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

**TEXT の `urlActions`**（本文中のURLタップでコンタクトタグを操作したい場合のみ設定。不要なら `null`）:
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
- ⚠️ **`urlActions` を設定するメッセージは `isTrackingEnabled: true` が必須**。`isTrackingEnabled: false` のまま `urlActions` を 1 件以上入れると、保存・公開は通るが**実行時に「トラッキング無効ではアクションは使用できません」で `FAILED`** になり、そのコンタクトの後続ステップも打ち切られる（保存時・公開時のガードは無い）。`urlActions` 不要なら `null`、使うなら `isTrackingEnabled: true` の組み合わせだけが有効

CarouselItem の形（`postbackActions` はキー省略不可。設定しない場合は `null` を明示する）:
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
- `postbackActions` — ボタンタップ時に実行するアクション（1〜3件。キー自体の省略は不可で、設定しない場合は `null` を明示）。
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
  "actionDate": null,
  "actionAfterDays": 3,
  "actionHours": 10,
  "actionMinutes": 0
}
```

**起点は waitTimeType で変わる**。`RELATIVE` / `ABSOLUTE` は直前ステップの実行時刻が起点で、`actionHours` / `actionMinutes` の解釈だけが異なる。`FIXED` はカレンダー日付の絶対日時。

- `actionAfterDays`（0-30）は RELATIVE / ABSOLUTE の相対加算日数（起点に N 日を足す）。FIXED では 0。
- `actionDate` は FIXED のみ YYYY-MM-DD。RELATIVE / ABSOLUTE は null。
- `waitTimeType: "RELATIVE"` — `actionHours` / `actionMinutes` は起点の時刻に**加算する**時間・分（「起点から N日 H時間 M分 後」）。`null` のフィールドは加算なし。
- `waitTimeType: "ABSOLUTE"` — `actionAfterDays` 日後の日付に、時刻を `actionHours:actionMinutes` へ**セット**する（クロック時刻指定）。`actionHours` と `actionMinutes` のどちらかでも `null` なら時刻セットは行われず、起点の時刻がそのまま維持される。
- `waitTimeType: "FIXED"` — `actionDate` の日付に、時刻を `actionHours:actionMinutes` へセットする（Asia/Tokyo）。時分は必須。稼働時に指定日時が過去だと 400。
- `actionHours` / `actionMinutes` は RELATIVE / ABSOLUTE では `nullable: true`（時刻指定なしも可）。FIXED では必須。
- ⚠️ RELATIVE / ABSOLUTE で計算結果の実行時刻が現在より過去になると、後続ステップはスケジュールされずそこで停止する（ユーザーへの通知は無い）。「配信されない」という相談の典型パターンなので、待機日数・時刻の設定はユーザーの意図（起点からの相対か、特定の時刻に揃えたいか、カレンダー日付か）を確認してから `RELATIVE` / `ABSOLUTE` / `FIXED` を選ぶ。
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
| `AUTO_WEBINAR_PARTICIPATION` | `conditionAutoWebinarParticipation` | `{ autoWebinarId: number }`（オートウェビナーへの参加有無。`autoWebinarId` はユーザーから確認。MCP に検索ツール無し） |
| `AUTO_WEBINAR_WATCH_TIME` | `conditionAutoWebinarWatchTime` | `{ autoWebinarId: number, metricType: "WATCH_POSITION"\|"TOTAL_PLAY_TIME", thresholdSeconds: number(0-86400) }`（オートウェビナーの視聴時間条件。`WATCH_POSITION`=到達した最大視聴位置、`TOTAL_PLAY_TIME`=視聴した総再生時間。`thresholdSeconds`は「以上」で判定。`autoWebinarId` はユーザーから確認。MCP に検索ツール無し） |
| `BANK_TRANSFER_STATUS` | `conditionBankTransferStatus` | `{ serviceId: string, bankTransferStatus: "PENDING"\|"COMPLETED"\|"REJECTED"\|"CANCELED" }`（対象ゲストの指定プラン・サービスへの注文が、指定した銀行振込の振込状況に一致するかで分岐。`PENDING`=振込待ち・未入金、`COMPLETED`=振込完了・入金の反映待ちを含む、`REJECTED`=振込期限切れ、`CANCELED`=キャンセル。クレジットカード等、銀行振込以外の支払い方法の申し込みは常に「一致しない」側に進む） |

`branches` の各要素:
- `matchValue: boolean` — 条件にマッチしたら true 側、外れたら false 側を実行
- `stages: Stage[]` — その分岐で実行する一連のステージ（再帰構造）

#### CONDITION は「一時点の判定」— 常時待ち受けたいなら専用トリガーが要る

`CONDITION` は、実行がそのステージに**到達した瞬間に 1 回だけ**評価される（WAIT_TIME の待機明けなど）。到達より後に状態が変わっても再評価されず、到達より前に起きた出来事にも即時には反応しない。したがって「◯◯したら**すぐに**」「◯◯したら**いつでも**」という要件は、`CONDITION` を含むワークフローをどう直しても実現できず、その出来事を**起点にする別ワークフロー**（専用トリガー）が必要になる。

専用トリガーが存在する conditionType は次の 2 つだけ（trigger 種別一覧は本ファイル「triggerType と対応サブフィールド」）:

| conditionType（一時点の判定） | 対応する専用 triggerType（出来事の直後に起動） | 同一対象と見なす条件 |
|---|---|---|
| `SERVICE_APPLICATION_STATUS` | `SERVICE_APPLIED` | `conditionServiceApplicationStatus.serviceId` と `serviceAppliedTrigger.serviceId` が一致（文字列比較）。`paymentMethod` を絞っていると一部の申込では起動しない |
| `CONTACT_TAG` | `CONTACT_TAG_ADDED` | `conditionContactTag.contactTagId` と `contactTagAdded.contactTagId` が一致。`CONTACT_TAG_ADDED` は**新規付与のときだけ**発火する（同表） |
| `BANK_TRANSFER_STATUS` | **無し** | — |
| `AUTO_WEBINAR_PARTICIPATION` / `AUTO_WEBINAR_WATCH_TIME` | **無し** | — |

専用トリガーが無い 3 つの conditionType に「即時反応」の要望が出た場合は、トリガー一覧に該当するものが無い事実だけを伝える（同じ論法で別のトリガーを代用案として捏造しない）。ヒアリングでの確認文・既存ワークフローの探し方は [best-practices.md](best-practices.md) の「単一ワークフローで閉じない要件」参照。

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

## コンテキスト適合表（意味解析の中核）

「検査の二層構造」で言う意味解析の中核。**先頭 stage の `triggerType` が実行コンテキストに供給する識別子**と、**各 action / condition が実行時に要求する識別子**の適合を判定する。スキーマはどの組み合わせも通すので、ここで初めて「動くか」が決まる。

各トリガーが供給する識別子は 2 種類のどちらか一方だけ（両方を供給するトリガーも、どちらも供給しないトリガーも無い）:

- **コンタクト ID**: コンタクトリスト上の人物。LINE 友だち追加・タグ付与・流入経路 CV・特典取得はこれ
- **MOSH ID**: MOSH アカウント（ゲスト）。プラン・サービス申込・開催リマインダー・決済失敗系はこれ

| triggerType | 供給する識別子 | `SEND_EMAIL`<br>[要: どちらか]※4 | `SEND_LINE_MESSAGE`<br>[要: コンタクト]※5 | `ADD_CONTACT_TAG` / `REMOVE_CONTACT_TAG`<br>[要: コンタクト] | `LINK_LINE_RICH_MENU` / `UNLINK_LINE_RICH_MENU`<br>[要: コンタクト]※5 | `CONDITION(CONTACT_TAG)`<br>[要: コンタクト] | `CONDITION(SERVICE_APPLICATION_STATUS)`<br>[要: MOSH] | `CONDITION(AUTO_WEBINAR_*)`<br>[要: MOSH] | `CONDITION(BANK_TRANSFER_STATUS)`<br>[要: MOSH] |
|---|---|---|---|---|---|---|---|---|---|
| `LINE_CHANNEL_CONTACT_REGISTERED` | コンタクト ID のみ | ○※4 | ○ | ○ | ○ | ○ | △ | △ | △ |
| `CONTACT_TAG_ADDED` | コンタクト ID のみ | ○※4 | ○ | ○ | ○ | ○ | △ | △ | △ |
| `INFLOW_ACTION_CONVERTED` | コンタクト ID のみ | ○※4 | ○ | ○ | ○ | ○ | △ | △ | △ |
| `MARKETING_LEAD_BENEFIT_RECEIVED`（既存維持のみ） | コンタクト ID のみ | ○※4 | ○ | ○ | ○ | ○ | △ | △ | △ |
| `SERVICE_APPLIED` | MOSH ID のみ | ○ | △ | △ | △ | △ | ○ | ○ | ○ |
| `SERVICE_SCHEDULE_REMINDER` | MOSH ID のみ | ○ | △ | △ | △ | △ | ○ | ○ | ○ |
| `INSTALLMENT_PAYMENT_FAILED` | MOSH ID のみ | ○ | △ | △ | △ | △ | ○ | ○ | ○ |
| `SUBSCRIPTION_PAYMENT_FAILED` | MOSH ID のみ | ○ | △ | △ | △ | △ | ○ | ○ | ○ |

凡例:

- **○ そのまま可** — トリガーが供給する識別子をそのまま使える
- **△ 紐付け経由・実行時に失敗しうる（要注意）** — 供給された識別子から、もう一方の識別子を**コンタクト↔MOSH ID の紐付け**で引く。紐付けが無い対象では `FAILED` になり、そのコンタクトの後続ステップも打ち切られる（※2・※3）。公開前に「対象者は紐付け済みか」をユーザーに確認し、確認できなければトリガーかステップを変える
- **× 実質不可（ほぼ必ず失敗）** — 識別子の観点で該当する組み合わせは現状無い（不適合はすべて紐付けで救いうる）。ただし識別子とは別の軸で「ほぼ全滅」する組み合わせは実在する: LINE 友だち追加起点 × `SEND_EMAIL`（※4。母集団実績は [best-practices.md](best-practices.md) の「トリガー × アクションの相性」）

脚注:

- ※1 供給識別子: タグ付与・流入経路 CV・LINE 友だち追加・特典取得のトリガーは実行コンテキストに**コンタクト ID だけ**を積み、プラン・サービス申込・開催リマインダー・分割決済失敗・サブスク決済失敗のトリガーは**MOSH ID だけ**を積む（コンタクト ID は空）
- ※2 △ の失敗の出方: コンタクトを要求する action / condition は、「供給されたコンタクト ID」と「供給された MOSH ID に紐付くコンタクト」を集めた結果が空だと `TARGET_USER_NOT_FOUND / CONTACT_NOT_FOUND`。MOSH ID を要求する condition は、「供給された MOSH ID」も「供給されたコンタクトに紐付く MOSH ID」も無いと `TARGET_USER_NOT_FOUND / GUEST_NOT_FOUND`。この原因コードは実行履歴（`getCreatorScenariosExecutions`）には集計値としてしか出ず、対象者単位のドリルダウンは現状 MCP から取得できない
- ※3 紐付け（コンタクト↔MOSH ID）が成立する経路: ①トラッキング有効（`isTrackingEnabled: true`）なメッセージ本文の URL、またはリッチメニューのゲートウェイリンクを**コンタクトがタップ**し、②その遷移先で **MOSH にログイン状態で到達**する（未ログインなら Cookie に保持され、次回ログイン時に紐付く）。1 コンタクトに紐付く MOSH ID は 1 つで、別の MOSH ID と既に紐付いていれば上書きされない。紐付けは事後には遡らないので、**紐付け前に実行された △ のステップは失敗したまま**。①の「トラッキング有効な URL」を配信できる場所は**ワークフローに限らない**: コンタクト向けメッセージ配信（一斉配信）も同じ紐付け経路（トラッキング用の短縮 URL）を使う。トグルの有無はメッセージ種別で異なり、**TEXT（LINE）と メール本文は `isTrackingEnabled: true` のときだけ**、**CAROUSEL / IMAGE_CAROUSEL / RICH_MESSAGE のタップ遷移先 URL はトグル無しで常にトラッキング**される（ワークフロー・一斉配信とも）。したがってテナント内に紐付けの入口があるかは、他のワークフローと一斉配信の両方を横断して探す（具体的な調べ方は [best-practices.md](best-practices.md) の「単一ワークフローで閉じない要件」参照）
- ※4 `SEND_EMAIL` の宛先解決: MOSH ID があれば MOSH アカウントのメールアドレス、無ければコンタクトの**有効かつ購読中**のメールアドレス。コンタクト起点でメールアドレスを持たないコンタクト（LINE 友だち追加だけで作られたコンタクト等）は `CONTACT_NOT_FOUND` で失敗する。識別子は適合していても**メールアドレス保有**という別条件があるので ○※4 とした
- ※5 `SEND_LINE_MESSAGE` / `LINK_LINE_RICH_MENU` / `UNLINK_LINE_RICH_MENU` は、コンタクトが解決できても**該当する LINE 公式アカウントの友だちでなければ** `CONTACT_NOT_FOUND`。照合先のアカウントは `SEND_LINE_MESSAGE` / `UNLINK_LINE_RICH_MENU` が**ワークフロー**の `creatorLineChannelId`、`LINK_LINE_RICH_MENU` だけは**リッチメニューが属する**アカウント（ワークフローのアカウントは照合に使われない）。タグ付与起点でメール由来のコンタクトが対象になる場合、ワークフローと別アカウントのリッチメニューを `LINK_LINE_RICH_MENU` に指定した場合などに起きる

この表が意味解析の中核（△・※4・※5 はいずれも対象者の紐付け・メール保有・友だち状態で成否が分かれる）。次節「埋め込み変数」の対応表は、使えるかどうかが trigger × action の組み合わせだけで決まり対象者に依存しないため**構文側**に属する（保存・公開で弾かれない点は同じ）。[best-practices.md](best-practices.md) の「トリガー × アクションの相性」はこの表を前提に、△ の実務上の扱いと、LINE 公式アカウントの一致など**識別子以外の相性**（こちらは対象者非依存＝構文側）を扱う。

## 埋め込み変数（差し込み文字）

`SEND_EMAIL` / `SEND_LINE_MESSAGE` の本文には `{{variable_name}}` 形式（二重中括弧）の埋め込み変数を使える。変数が解決されるのは**メッセージ本文のみ**で、`SEND_EMAIL` の `subject`（件名）では解決されず `{{...}}` がそのまま件名に残る（件名には使わない）。**Excel等の外部ソースにある `%foo%` のようなプレースホルダ記法はMOSHでは解釈されない**（そのまま文字列として残るだけ）ので、本文を外部ドキュメントから転記する際は必ず `{{...}}` 形式に置き換えること。

### 対応変数一覧（この5つ以外は存在しない）

| 変数名 | 意味 / データソース | 使える条件 |
|---|---|---|
| `line_name` | コンタクトのLINEプロフィール表示名 | `actionType: SEND_LINE_MESSAGE` の時のみ（`SEND_EMAIL`では常に空文字） |
| `guest_name` | ゲスト（Moshユーザー）の名前 | `actionType: SEND_EMAIL` かつ `triggerType` が `SERVICE_APPLIED` / `SERVICE_SCHEDULE_REMINDER` / `INSTALLMENT_PAYMENT_FAILED` / `SUBSCRIPTION_PAYMENT_FAILED` の時のみ。それ以外は空文字 |
| `service_name` | トリガーに紐づくプラン・サービス名 | `triggerType` が `SERVICE_APPLIED` / `SERVICE_SCHEDULE_REMINDER` / `INSTALLMENT_PAYMENT_FAILED` / `SUBSCRIPTION_PAYMENT_FAILED` の時（決済失敗系も実行コンテキストに対象プラン・サービスの参照が積まれるため値が入る）。それ以外のトリガー（既存の `MARKETING_LEAD_BENEFIT_RECEIVED` / `LINE_CHANNEL_CONTACT_REGISTERED` / `CONTACT_TAG_ADDED` / `INFLOW_ACTION_CONVERTED`）では常に空文字 |
| `reservation_time_range` | 予約日時の範囲（`YYYY年M月D日 HH:mm〜HH:mm`, JST） | `SERVICE_SCHEDULE_REMINDER`は常に対応。`SERVICE_APPLIED`はプラン・サービスの`serviceType`が「予約(event)」または「個別(private)」の場合のみ（コンテンツ/サブスク/オンライン単体のプラン・サービスでは空文字） |
| `zoom_url` | ZoomのjoinURL | `reservation_time_range`と同条件に加えて、プラン・サービスの`locationType`がオンライン/ハイブリッドかつクリエイターがZoom連携済みの場合のみ。条件を満たさない場合は静かに空文字になる（保存・送信はブロックされない） |

存在しない変数名（上表の5つ以外のキー）は**エラーにならずプレースホルダ文字列がそのまま残る**（無言の失敗）。上表の5変数は、使える条件を満たさないトリガー・アクションの組み合わせでは**空文字**に置換される（実害例: メール本文の宛名に `{{line_name}}` を書くと、保存・公開はエラーにならないが宛名が空文字のまま配信される）。どちらも意図通りに差し込まれないため、使う前に上表の条件を必ず確認する。

## 流入経路 / トラッキング同意

- 流入経路 (`inflowAction`) は LINE公式アカウントに紐づく URL を発行する仕組み。`scenarioTriggerInflowActionConverted` の `inflowActionId` で参照
- トラッキング同意は API 利用前提として `postCreatorScenariosTrackingConsent` で記録。`getCreatorScenariosTrackingConsents` で同意状態取得

## INACTIVE 状態の PATCH 検証

`INACTIVE` のワークフローに対する `patchCreatorScenario` では、参照 ID の実在検証の有無がフィールドで異なる:

- **`benefitId` / `serviceId` の2つだけは実在検証されない** — 存在しない ID を入れても 200 OK で保存される（実在が検証されるのは公開＝`ACTIVE` 化時。`benefitId` は既存の特典取得ワークフローの維持時のみ）
- それ以外の参照 ID（`contactTagId` / `inflowActionId` / `creatorLineChannelId` / `autoWebinarId` / `muxAssetId` / `lineRichMenuId`）は **PATCH 時にも実在検証され、不正なら 400 で弾かれる**（保存できた時点で実在は保証される）
- 「保存できた」を「動く」と思い込まないこと。`serviceId`（および既存ワークフローの `benefitId`）は実在の管理画面リソースに対応しているかをユーザーに口頭で確認する
