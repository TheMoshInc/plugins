---
name: workflow-builder
description: MOSH のワークフローを MCP 経由で作成・編集・運用するスキル。要件ヒアリング → 空ワークフロー作成 → ステージ（trigger + action）構築 → レビュー → ユーザー承認のうえ公開、という段階で対話的に進める。「ワークフロー作って」「ワークフロー組んで」「ステップ配信を作って」「ワークフローを有効化」「ワークフローの分岐を追加」「workflow-builder」など、MOSH のワークフロー作成・編集・運用リクエストで使用する。流入アクション・トラッキング同意・ワークフロー複製も含む。
---

# MOSH ワークフロービルダー

## Overview

MCP ツール (`*Scenario*` および `*InflowAction*` / `*TrackingConsent*`) を使って、MOSH クリエイターのワークフローを作成・編集・運用する。空のワークフローを作成し、`stages` 配列で「いつ起動するか（trigger）」と「何を実行するか（action）」を組み立て、ユーザーの承認を得て公開（`ACTIVE` 化）する。

> **ツール名表記について**: 本スキル内では MCP ツールを OpenAPI の `operationId`（例: `postCreatorScenario`）で参照する。MCP クライアントが実際に提示するツール名にはサーバー識別子のプレフィックス（例: `mcp__<server>__postCreatorScenario`）が付くが、その形はユーザー環境により異なるため、本スキルでは付けない。

> **呼称について**: プロダクト上の呼称は「ワークフロー」、API / JSON のキー名は `scenario*`（例: `scenarioAction`, `scenarioTrigger`）。MCP ツール名 (`*Scenario*`) や JSON のキー名はスキーマ通りに使うこと。

`stages[].trigger` と `stages[].action` の構造には**強い制約**があるため、構築前に [references/content-schema.md](references/content-schema.md) を必ず参照する。`actionType` / `triggerType` ごとに必須となるサブオブジェクトが切り替わる仕様で、ナイーブに JSON を組むとほぼ確実に失敗する。

## 前提条件: 参照リソース

ワークフローは「**何かが起きたら → 何かをする**」を組み立てる仕組みなので、**起点となる既存リソースが最低1つ必要**。真っさらなテナント（まだ何も作っていない状態）ではどの trigger も構築できない。

最初に必ずユーザーへ確認する: 以下のうち**最低1つは既に作成済み**ですか？

| triggerType | 必要な既存リソース |
|---|---|
| `MARKETING_LEAD_BENEFIT_RECEIVED`（特典取得） | **特典** が1つ以上（`benefitId` を確認） |
| `LINE_CHANNEL_CONTACT_REGISTERED`（LINE友達追加） | **LINE チャンネル** が連携済み（`creatorLineChannelId` を確認。trigger サブフィールド不要、ボディ最上位で指定） |
| `SERVICE_APPLIED`（サービス申込） | **サービス** が1つ以上（`serviceId` を確認） |
| `SERVICE_SCHEDULE_REMINDER`（開催リマインダー） | **サービス** + 開催スケジュール |
| `CONTACT_TAG_ADDED`（タグ付与） | **顧客タグ** が1つ以上（`contactTagId` を確認） |
| `INFLOW_ACTION_CONVERTED`（流入アクション CV） | **流入アクション** が1つ以上（`inflowActionId` を確認、`getCreatorScenariosInflowActions` で取得可） |

action 側にも参照リソースが必要なケースがある:

| action | 必要な既存リソース |
|---|---|
| `SEND_LINE_MESSAGE` | **LINE チャンネル**（`creatorLineChannelId` がワークフロー側に必要） |
| `ADD_CONTACT_TAG` / `REMOVE_CONTACT_TAG` | **顧客タグ** |
| `CONDITION` (`CONTACT_TAG`) | **顧客タグ** |
| `CONDITION` (`SERVICE_APPLICATION_STATUS`) | **サービス** |
| `CONDITION` (`AUTO_WEBINAR_PARTICIPATION`) | **オートウェビナー** |

参照したいリソースが無い場合は、ユーザーに**先に管理画面で作成**してもらってからワークフロー構築に着手する。

## When to use

- ユーザーが新しいワークフローを作りたいと依頼したとき
- 既存ワークフローの stage（trigger / action）を更新したいとき
- 既存ワークフローを複製したいとき
- 流入アクション（INFLOW_ACTION）を作成・参照したいとき
- `*Scenario*` 系ツールが必要な文脈

## When NOT to use

- LP（ランディングページ）の作成・編集 → `lp-builder` スキル
- 単発のメール・LINE 送信（ワークフロー化しないもの）
- 受講者プロフィール・決済・受講者管理など、ワークフロー以外のドメイン

## Workflow（推奨フロー）

このスキルは **下書き作成 → ユーザー確認 → 公開（`ACTIVE` 化）** を基本フローとする。公開前に必ずユーザーの承認を取る。

### 1. 要件ヒアリング

ユーザーに以下を確認する:

- **起動条件（trigger）**: 何が起きたら走らせるか
- **実行内容（action）**: 何を実行するか、何ステップ並べるか
- **参照リソース**: 上の「前提条件」表の該当リソースが既にあるか、その ID
- **メッセージ内容**: メール件名/本文、LINE メッセージ等
- **クリエイター名・トーン**: メッセージ案を組み立てるときに使う

### 2. 空ワークフローの作成（INACTIVE で誕生）

`postCreatorScenario` で名前だけ指定して作成する。

```json
{ "bodyParams": { "name": "ウェルカムメッセージワークフロー" } }
```

返却された `id` を以降のステップで使用する。**作成直後のライフサイクルは `INACTIVE`（下書き）で固定**。

### 3. stages の構築（trigger + action の組み立て）

[references/content-schema.md](references/content-schema.md) を読み込み、構造ルールに沿った JSON を組み立てる。設計指針は [references/best-practices.md](references/best-practices.md) を参照。

`patchCreatorScenario` で `stages` を更新する:

```json
{
  "pathParams": { "id": <作成したワークフローID> },
  "bodyParams": {
    "creatorLineChannelId": <LINEチャンネルID または null>,
    "stages": [
      { "trigger": { ... }, "action": { ... } },
      {                     "action": { ... } }
    ]
  }
}
```

`stages` は**配列まるごと置き換え**になる想定で組む。1ステージ目の `trigger` は必ず指定し、2ステージ目以降は `trigger` フィールド自体を省略する（先頭ステージの trigger を契機に連鎖実行される）。`trigger: null` は Zod バリデーションで弾かれるため使用禁止。

PATCH 時は `action` 配下の構造が **`scenarioActionForUpdate`** 系（`condition` だけ `scenarioActionConditionForUpdate`）になる点に注意。詳細は content-schema 参照。

> ⚠️ **PATCH 200 OK ≠ 動作保証**: INACTIVE 状態の PATCH は検証が緩く、**存在しないリソース ID を入れても 200 OK が返る**（例: 存在しない `benefitId` でも保存できてしまう）。「保存できた」を「正しく動く」と勘違いしない。ユーザーに参照リソース ID が実在のものかを口頭で確認すること。

### 4. レビュー（公開前の最終確認）

MOSHのAPIは `ACTIVE` 化時に埋め込み変数の可否を検証しない（フロントエンドの編集画面だけがこのチェックを行う）。MCP経由で操作する場合はこのガードが効かないため、**埋め込み変数のチェックもこのレビューの一部として自分で行う**。

1. `getCreatorScenario` で現在の状態を取得する
2. **JSON を貼らずに自然言語の箇条書きで要約**してユーザーに提示する（「ユーザーへの提示・コミュニケーション規約」§1 参照）。最低限の項目: ワークフロー名 / 公開状態 / トリガー（と参照リソース ID） / アクション（メール件名・本文の要旨、トラッキング有無等）
3. 参照リソース ID が全て実在のものか口頭で確認する（INACTIVE の PATCH は ID 実在検証が行われないため）
4. **埋め込み変数チェック**: 各 stage の `action` 内の `sendEmail.message` / `sendLineMessage.messages[].text` から `{{...}}` パターンを全て抽出し、そのステージが従う trigger（先頭 stage の `triggerType`）と `actionType` の組み合わせで [content-schema.md](references/content-schema.md) の対応表に照らして使用可能か確認する。使用不可の変数が1つでもあれば、ユーザーに「この文言は現在のトリガーでは差し込まれず空欄配信になります」と日本語で具体的に指摘し、修正（変数を外す／使える変数に変える／トリガーを変える）してもらう
5. 「この内容で公開しますか？」と必ずユーザーに確認する（埋め込み変数に問題がある間は公開に進まない）
6. 修正が必要なら Step 3 に戻る

### 5. 公開（`ACTIVE` 化）

ユーザーが公開に合意したら、`patchCreatorScenariosLifecycle` で `ACTIVE` 化する:

```json
{ "pathParams": { "id": <ワークフローID> }, "bodyParams": { "lifecycle": "ACTIVE" } }
```

`ACTIVE` 化時には参照リソースの実在や同時稼働数の上限が確認され、満たさない場合は理由を示すエラーが返る（例:「指定された特典が見つかりません。」「LINE公式アカウントを設定してください。」）。内容をユーザーへ平易に伝え、原因を解消してもらってから再度公開する。公開後は `getCreatorScenario` で稼働中（`ACTIVE`）になったことを確認する。

## ユーザーへの提示・コミュニケーション規約

このスキルを使うエンドユーザーは MCP / JSON / HTTP の専門知識を持たない前提で会話する。Claude が「内部で何をしているか」と「ユーザーに見せるもの」を切り分けること。

### 1. JSON 構造を直接ユーザーにレビューさせない

- 下書き作成前のドラフトも、PATCH 後のレビュー要約も、**JSON ブロックを並べてユーザーに「これでよいですか？」と聞かない**。
- 提示するのは、自然言語の箇条書きで「トリガー / アクション / メッセージ内容 / 公開状態」を要約したものだけ。
- 例: 「特典取得をきっかけに、件名◯◯・本文◯◯のメールを1通送る構成です。トラッキングは OFF、下書き状態です。」
- ユーザーが「中身の JSON が見たい」と明示的に求めた時だけ、参考情報として JSON を出す。

### 2. ステータスコード / HTTP 用語をユーザー向け文面に出さない

- `200 OK`, `500`, `status: ...`, `PATCH`, `GET`, `endpoint` などの語彙は**ユーザー向けの文面に書かない**。
- 内部判定（例: 業務エラーの種別で分岐する）には使ってよいが、ユーザーには「保存できました」「公開しました」「特典が見つからないため公開できませんでした」など平易な日本語で伝える。
- 失敗時も「うまくいきませんでした。原因は◯◯のため、〜してください」のように原因と対処を日本語で示す。コードは出さない。

### 3. 参照 ID が未確定のときは PATCH しない

- `benefitId` 等の参照リソース ID が未確定の場合、**INACTIVE 状態でも PATCH はしない**。INACTIVE の PATCH は参照 ID の実在検証が行われないため、空文字や不正な ID のまま保存してもエラーにならず、ユーザーが気付かず公開した場合にトリガー・アクションが実リソースに紐づかない壊れたワークフローになる。
- 未確定 ID がある場合は、**会話内に下書き案を保持したまま**「`benefitId` が必要です。管理画面で特典 ID を確認してから教えてください」とユーザーに確認を促す。
- 実 ID が揃ってから `patchCreatorScenario` で保存する。

## 必須ルール

### A. `action` は `actionType` + 7サブオブジェクト全て指定

`scenarioAction` / `scenarioActionForUpdate` は型定義上、以下 8 フィールドが **全て required**:

- `actionType`（enum）
- `sendEmail` / `sendLineMessage` / `waitTime` / `condition` / `addContactTag` / `removeContactTag` / `linkLineRichMenu`

実態は `actionType` に対応する 1 フィールドだけを実オブジェクトで埋め、**残り 6 つは `null` を明示**する。`undefined` ではなく `null`。`actionType` には `LINK_LINE_RICH_MENU`（`linkLineRichMenu` を設定）と `UNLINK_LINE_RICH_MENU`（詳細フィールドを持たず全て `null`）も追加されている。

```json
{
  "actionType": "SEND_EMAIL",
  "sendEmail": { "subject": "...", "message": "...", "isTrackingEnabled": true },
  "sendLineMessage": null,
  "waitTime": null,
  "condition": null,
  "addContactTag": null,
  "removeContactTag": null,
  "linkLineRichMenu": null
}
```

### B. `trigger` も `triggerType` + 5サブオブジェクト全て指定

`ScenarioTrigger` 型のサブフィールドは `triggerType` 以外に 5 つ（`marketingLeadBenefitReceivedTrigger` / `serviceAppliedTrigger` / `serviceScheduleReminder` / `contactTagAdded` / `inflowActionConverted`）。対応する 1 フィールドだけ実オブジェクト、他は `null`。`lineChannelContactRegisteredTrigger` は型定義に存在しないため指定禁止。`LINE_CHANNEL_CONTACT_REGISTERED` の場合はサブフィールドが全て `null` で、LINE チャンネルはボディ最上位の `creatorLineChannelId` で指定する。詳細は content-schema 参照。

### C. PATCH 時は `*ForUpdate` 系を使う

- `patchCreatorScenario` の `stages[].action` は `scenarioActionForUpdate` 構造
- 内部の `condition` は `scenarioActionConditionForUpdate`
- `branches[].stages[].action` は再帰参照を避けるため `additionalProperties: true` の汎用オブジェクト扱い（型定義は緩いが、サーバー側で再帰的に再検証される）

### D. ID は数値 / 文字列の使い分けに注意

| 種別 | 型 | 例 |
|---|---|---|
| `serviceId` / `benefitId` | **文字列**（数値文字列） | `"<service_id>"` |
| `contactTagId` / `inflowActionId` / `creatorLineChannelId` / `autoWebinarId` / `muxAssetId` | **整数** | `<contact_tag_id>` |
| `previewMoshImageId` | 文字列 | `"<preview_mosh_image_id>"` |

### E. LINE メッセージは `type` と `messages` の形が連動

`scenarioActionSendLineMessage.messages` は `type` に応じて形が変わる（仕様上は oneOf）:

- `TEXT` → `string[]`（1〜5件、各1〜5000文字）
- `CAROUSEL` → `{ altText, carousels[] }`（1〜4件）
- `IMAGE_CAROUSEL` → `{ altText, imageCarousels[] }`（1〜4件）
- `VIDEO` → `{ muxAssetId, previewMoshImageId }`

### F. 埋め込み変数は次の5つ「だけ」（名前は完全一致・それ以外は無言で失敗）

メール本文・LINE 本文で使える `{{...}}` 埋め込み変数は**以下の5つだけ**。この名前を**一字一句そのまま**使うこと。ここに無い変数名（例: `{{reservation_datetime}}`, `{{customer_name}}`, `{{date}}` 等）は**存在せず、エラーにもならずそのまま文字列として配信される（無言の失敗）**ため、絶対に創作しない。

| 変数（この綴りで固定） | 意味 | 使える条件 |
|---|---|---|
| `{{guest_name}}` | ゲスト名 | `SEND_EMAIL` かつ trigger が `SERVICE_APPLIED` / `SERVICE_SCHEDULE_REMINDER` |
| `{{line_name}}` | LINE プロフィール名 | `SEND_LINE_MESSAGE` のみ |
| `{{service_name}}` | サービス名 | trigger が `SERVICE_APPLIED` / `SERVICE_SCHEDULE_REMINDER` |
| `{{reservation_time_range}}` | 予約日時の範囲（**`reservation_datetime` ではない**） | `SERVICE_SCHEDULE_REMINDER`、および予約型 `SERVICE_APPLIED` |
| `{{zoom_url}}` | Zoom 参加 URL | 上記予約系＋オンライン/Zoom 連携時 |

本文を書く前に、使いたい概念が上表にあるか必ず確認する。無ければ変数を使わず固定文言にする。詳細な可否は [references/content-schema.md](references/content-schema.md) の対応表を参照（本文の5変数が唯一の真実）。

## よくあるミス

| NG | OK |
|---|---|
| `actionType` を指定して該当フィールドだけ埋め、他を省略 | 他 6 フィールドを `null` で明示する |
| 他フィールドを `undefined` / 空オブジェクト `{}` で埋める | 必ず `null` を明示する |
| `triggerType` だけ書いて trigger サブオブジェクトを省略 | 対応する 1 フィールド以外は全て `null` を明示 |
| PATCH で `scenarioAction`（POST用）を使う | PATCH では `scenarioActionForUpdate` 構造を使う |
| `serviceId: <service_id>`（クォート無しの整数として書く） | `serviceId: "<service_id>"`（クォート付きの文字列として書く） |
| `getCreatorScenarios` の `lifecycle` に `"ACTIVE"` を指定 | クエリ上の enum は `"INACTIVE_ACTIVE"` か `"ARCHIVED"` の2値のみ |
| 2ステージ目以降にも `trigger` を入れる | 通常は先頭 stage のみ trigger を持ち、以降は `trigger: null` |
| LINE メッセージで `type: "TEXT"` なのに `messages: { altText: ... }` | `type: "TEXT"` なら `messages` は `string[]` |
| 真っさらなテナントでワークフロー構築を始める | 先に「前提条件: 参照リソース」表のリソースを1つ以上用意してもらう |
| PATCH 200 OK を「動く」と即断する | INACTIVE 中は検証が緩く、不正な ID でも保存できる。実在を口頭で再確認 |
| 参照リソースの実在を確認せず `ACTIVE` 化する | 下書き中は ID が未検証。公開時に弾かれるので、公開前に特典/サービス/LINE 等の実在を確認する |
| 作成直後にすぐ `ACTIVE` 化を試す | まず `INACTIVE` のまま stages を構築・確認し、ユーザーの承認を得てから公開する |
| JSON ブロックをユーザーに貼って「この内容で OK ですか？」と聞く | 自然言語の箇条書きで要約してから確認する（ユーザーは JSON を読めない前提） |
| `200 OK` / `500` / `PATCH` 等のコード・HTTP 用語をユーザー向け文面に出す | 「保存できました」「公開しました」など平易な日本語で伝える |
| 参照 ID が未確定のまま `patchCreatorScenario` を呼ぶ（空文字・ゼロ・ダミー値を入れる） | 実 ID が揃うまで PATCH しない。会話内で下書き案を保持し、ユーザーに ID 確認を促す |
| 2ステージ目以降に `trigger: null` を指定する | `trigger` フィールド自体を省略する（null は Zod バリデーションで弾かれる） |
| `lineChannelContactRegisteredTrigger` を trigger に含める | 型定義に存在しない。LINE チャンネルはボディ最上位の `creatorLineChannelId` で指定し、trigger サブフィールドは全て `null` |
| 埋め込み変数チェックをせず `ACTIVE` 化する | `ACTIVE` 化前に自分で `{{...}}` を抽出し、trigger/actionType の組み合わせで使用可能か確認する（Step 4 レビューの一部） |
| 存在しない埋め込み変数を創作する（例: `{{reservation_datetime}}` / `{{customer_name}}` / `{{date}}`） | 必須ルール F の5つ（`guest_name` / `line_name` / `service_name` / `reservation_time_range` / `zoom_url`）だけを綴りそのまま使う。無ければ固定文言にする |

## References

| ファイル | 内容 |
|---|---|
| [references/content-schema.md](references/content-schema.md) | `stages[].trigger` / `stages[].action` の JSON 構造、enum 一覧、actionType 別テンプレ、`*ForUpdate` 差分 |
| [references/best-practices.md](references/best-practices.md) | ID 参照の確認手順、ライフサイクル運用、動作仕様、命名規約 |
| [references/mcp-tools.md](references/mcp-tools.md) | 各 MCP ツールのパラメータ仕様 |
| [examples/welcome-benefit-email.json](examples/welcome-benefit-email.json) | 特典取得（`MARKETING_LEAD_BENEFIT_RECEIVED`）をトリガーに、ウェルカムメール（`SEND_EMAIL`）を1通送る最小例。`patchCreatorScenario` の bodyParams としてそのまま渡せる形 |
