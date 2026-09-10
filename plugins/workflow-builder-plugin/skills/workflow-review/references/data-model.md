# データモデル（構造と実行実績の読み方）

レイヤー0に着手する前に必ず読む。**誤読が誤診断に直結する**箇所だけを集めている。

裏取り:

- **構造**: `getCreatorScenario` の実レスポンス（テストアカウント上のワークフローで確認。確認日 2026-08-28）と、[`workflow-builder` の references/content-schema.md](../../workflow-builder/references/content-schema.md)
- **実行実績**: `getCreatorScenariosExecutions` の OpenAPI レスポンススキーマと実際の挙動。確認日 2026-08-28

## 構造（`getCreatorScenario`）の読み方

**本スキルの主軸。実行データが1件も無くてもここまでは確定できる。**

```json
{
  "stages": [
    { "handle": "<uuid>", "trigger": { "triggerType": "...", ... }, "action": { ... } },
    { "handle": "<uuid>", "action": {
        "actionType": "CONDITION",
        "condition": {
          "conditionType": "SERVICE_APPLICATION_STATUS",
          "conditionServiceApplicationStatus": { "serviceId": "<service_id>", "serviceApplicationStatus": "APPLIED" },
          "branches": [
            { "matchValue": true,  "stages": [] },
            { "matchValue": false, "stages": [
                { "handle": "<uuid>", "action": { "actionType": "SEND_LINE_MESSAGE", "sendLineMessage": { ... } } }
            ] }
          ]
        }
    } }
  ]
}
```

- `stages[]` は上から順に実行される直列の並び。先頭 stage だけが `trigger` を持つ
- **`branches[]` の各要素が `matchValue`（true / false）と、その分岐で実行する子ステージの木（`stages[]`）を丸ごと埋め込んでいる**。ジャンプ先の参照ではなく実体なので、**どちらの側で何が起きるかを構造だけで 100% 確定できる**（trueなら◯◯、falseなら△△、と日本語で書き下せる）
- **`stages: []`（空配列）は「その側は何もせず終わり」**。もう片方の branch に合流するのでも、後続に進むのでもない
- 分岐は再帰する（branch の中にさらに CONDITION を置ける）
- `conditionType` ごとのサブフィールド、`actionType` ごとのサブフィールド、ID の型といった**構造スキーマの網羅表は builder の content-schema.md が正本**。本ファイルでは再掲しない

### 親子・前後関係の読み方（S-2 の順序判定に使う）

「あるステージが別のステージより前か」は、埋め込み構造をたどれば機械的に決まる。

- **前（上流）**: 同じ `stages[]` 配列内で添字が小さい／その配列を含む branch の CONDITION ステージ、およびそのさらに祖先
- **到達しない**: 別の branch サブツリーの中にあるステージ同士は、**互いに実行されない**（true 側の人は false 側のステージを一切通らない）。「上に書いてあるから先に実行される」は誤り
- **合流**: CONDITION ステージの**次の要素（同じ配列内の後続ステージ）**には、両 branch を通った人が合流して到達する。片側 branch を空にして「そこで終わり」にしたつもりでも、後続ステージがあれば全員そこへ進む（S-3）。**この構成は現在 `patchCreatorScenario` が新規保存を拒否する**（400エラー）ため、既存データで見つかった場合はこの制約が入る前に保存されたものに限られる

## 実行モデルとスナップショット

```
ワークフロー本体（編集するとここが変わる）
  └─ スナップショット        ← 公開（稼働ON）のたびに1件作られる「凍結コピー」
       └─ ステージスナップショット
            └─ ステージ実行（1コンタクト × 1ステージ = 1行）
```

実行時に参照されるのはスナップショットであって編集中の構造ではない。**構造（`getCreatorScenario`）と実績（`getCreatorScenariosExecutions`）は別世代を見ている可能性がある** — 公開後に編集していれば、画面上の構造と実績のステージが対応しない。

### 最新スナップショット制約【最重要】

`getCreatorScenariosExecutions` は **最新のスナップショット1件分の実行しか集計しない**（作成日時が最新のスナップショットだけを集計対象にする）。

- 編集して再公開すると新しいスナップショットが作られ、**カウントは実質リセットされる**
- したがって「実績0件」は「壊れている」と「昨日再公開したばかり」を区別できない
- 対策: レイヤー1に入る前に**最後に公開（稼働ON）した日**をユーザーに確認する。MCP から公開日を直接取る手段はない

## `getCreatorScenariosExecutions` レスポンスの読み方

パラメータは `{ id: <ワークフローID・整数> }` のみ。

```
{
  trigger: { type, <type別サブオブジェクト> },
  steps: [
    { stageSnapshotId, type, waitTime, totalCount, completedCount, failedCount, steps: [...] },
    ...
  ]
}
```

**`trigger` の type別サブオブジェクトには表示名が入っている**ので、参照リソースを別途引き直す必要はない:

| type | サブオブジェクト | 表示に使う値 |
|---|---|---|
| `LINE_CHANNEL_CONTACT_REGISTERED` | `lineChannelContactRegisteredTrigger` | `displayName`（LINE公式アカウント名） |
| `SERVICE_APPLIED` | `serviceAppliedTrigger` | `title` |
| `SERVICE_SCHEDULE_REMINDER` | `serviceScheduleReminderTrigger` | `title`, `remindTimeType`, `beforeDays` |
| `CONTACT_TAG_ADDED` | `contactTagAddedTrigger` | `name`（タグ名） |
| `INFLOW_ACTION_CONVERTED` | `inflowActionConvertedTrigger` | `name`（流入経路名） |
| `MARKETING_LEAD_BENEFIT_RECEIVED` | `marketingLeadBenefitReceivedTrigger` | `title` |
| `INSTALLMENT_PAYMENT_FAILED` / `SUBSCRIPTION_PAYMENT_FAILED` | なし | — |

| 項目 | 意味 | 落とし穴 |
|---|---|---|
| `steps[0]` | トリガーステージ（先頭）。以降 `steps` は上流→下流の順 | — |
| `totalCount` | **`completedCount` + `failedCount` のみ** | 待機中（SCHEDULED）とキャンセル（CANCELED）は含まれない。WAIT_TIME 直後の件数減は「待機中の人が見えていないだけ」で正常 |
| `completedCount` | 正常に実行できた件数 | CONDITION では「条件が false だった人」も COMPLETED に含まれる |
| `failedCount` | 実行できなかった件数 | 「条件に合わなかった」ではない（下の原因コード辞書） |
| `waitTime` | WAIT_TIME ステージのみ非 null。`{waitTimeType, actionDate, actionAfterDays(0-30), actionHours, actionMinutes}` | 待機の長さは `actionAfterDays` で読む |
| `steps[].steps` | CONDITION の分岐先サブツリー（CONDITION 以外では空配列） | **どちらの `matchValue` に対応するかは下記の通り未検証** |
| 404 | **一度も稼働していない**（スナップショット不在）／トリガーステージが特定できない | エラー扱いしない |

### 実績と分岐の対応づけ【未検証・フェイルセーフに扱う】

実績側の `steps[].steps`（CONDITION の子）が、構造側の `branches[]` のどちらに対応するかは**実クリエイターの生レスポンスで未確認**（テストアカウントに実行履歴のあるワークフローが無いため）。次の順で扱う:

1. 実績のステージエントリに構造側と同じ `handle` が**含まれていれば**、それで対応づけて確定してよい
2. 含まれていなければ**対応づけを推測しない**。レイヤー0で確定している「true 側／false 側で何が起きるか」に、実績の合計値を突き合わせて言える範囲（例:「この分岐を通った◯人のうち、配信が実行されたのは△人」）にとどめ、`要確認` として報告する

この穴は `getCreatorScenariosExecutions` に `handle` を含める拡張（SKILL.md §未解決の制約）で根本解決する。

## 失敗の原因コード辞書

**MCP では取得できない**（原因コード別の内訳を返すツールが無い。SKILL.md §未解決の制約の1件目）。したがって通常のレビューでは**原因を断定せず、下表から最も確からしい仮説を1つ提示するだけ**にとどめる。ユーザーが管理画面の実行履歴で見た文言を伝えてきたとき、および PM が BigQuery で深掘りするときの読み替え表として使う。

| errorCode / errorDetailCode | 主に出るアクション | 意味 | 疑うべき設定ミス |
|---|---|---|---|
| `TARGET_USER_NOT_FOUND` / `CONTACT_NOT_FOUND` | SEND_EMAIL, SEND_LINE_MESSAGE, ADD_CONTACT_TAG, REMOVE_CONTACT_TAG, LINK_LINE_RICH_MENU, UNLINK_LINE_RICH_MENU, CONDITION(CONTACT_TAG) | 実行時点で対象コンタクトが解決できない。SEND_EMAIL では**メールアドレスを持たないコンタクトが対象**のケース、LINE配信・タグ操作では**MOSH ID からコンタクトを引けない**ケースが支配的 | 2方向ある。①LINE友だち追加・タグ付与など**メールアドレスを持たない起点**で `SEND_EMAIL` を使っている（S-5）。②`SERVICE_APPLIED` / `SERVICE_SCHEDULE_REMINDER` / 決済失敗系の**MOSH ゲスト起点**で `SEND_LINE_MESSAGE` / タグ操作 / リッチメニュー操作 / `CONDITION(CONTACT_TAG)` を使っていて、対象ゲストにコンタクト↔MOSH ID の紐付けが無い（S-5 の逆方向。紐付けの成立条件は [builder の content-schema.md](../../workflow-builder/references/content-schema.md) の「コンテキスト適合表」※3）。また LINE 系はコンタクトが解決できても、ワークフローの LINE 公式アカウントの友だちでなければ同じコードになる。**いずれもこのコンタクトの後続ステップも打ち切られる** |
| `TARGET_USER_NOT_FOUND` / `GUEST_NOT_FOUND` | CONDITION(SERVICE_APPLICATION_STATUS / AUTO_WEBINAR_* / BANK_TRANSFER_STATUS), SEND_EMAIL | 対象の MOSH アカウントが解決できない。CONDITION ではコンタクトに MOSH アカウントが紐付いていない、SEND_EMAIL では MOSH アカウントのメールアドレスが取得できない | コンタクト起点のワークフローで、MOSH アカウント前提の条件（`SERVICE_APPLICATION_STATUS` / `BANK_TRANSFER_STATUS` / オートウェビナー系）を使っている（S-5 の同型。適合表では △） |
| `REJECTED` / `LINE_BLOCKED` | SEND_LINE_MESSAGE | 相手が LINE 公式アカウントをブロック済み | 設定ミスではない。一定割合は正常。率が突出して高い場合は配信頻度・内容の問題 |
| `QUOTA_LIMIT_ERROR` / `LINE_QUOTA_LIMIT` | SEND_LINE_MESSAGE | LINE 公式アカウントの月間配信通数上限に到達 | 設定ミスではなく運用課題。LINE 側プランの見直しを案内する |
| `TARGET_RESOURCE_NOT_FOUND` / `CREATOR_LINE_CHANNEL_NOT_FOUND` | SEND_LINE_MESSAGE | ワークフローに紐づく LINE 公式アカウントが解決できない | LINE 連携が解除された、または流入経路が属するアカウントとワークフローの LINE 公式アカウントが不一致（S-4） |
| `UNKNOWN_ERROR` | SEND_LINE_MESSAGE | 分類不能 | 件数が少なければ様子見。多発するならエンジニアにエスカレーション |
