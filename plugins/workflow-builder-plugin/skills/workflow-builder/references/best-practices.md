# ワークフロー設計ベストプラクティス

## ID 参照の確認手順

ワークフローに埋め込む各種 ID のうち、`serviceId` / `benefitId` は MCP に検索・一覧取得ツールが無い。それ以外（`inflowActionId` / `creatorLineChannelId` / `contactTagId` / `autoWebinarId` / `lineRichMenuId`）は対応する一覧取得ツールで取得・実在確認できる。

| ID | MCP で取得できるか | 取得経路 |
|---|---|---|
| `inflowActionId` | ✅ | `getCreatorScenariosInflowActions` |
| `creatorLineChannelId` | ✅ | `getCreatorLineChannels` |
| `contactTagId` | ✅ | `getCreatorContactTags` |
| `autoWebinarId` | ✅ | `getCreatorAutoWebinars` |
| `lineRichMenuId` | ✅ | `getCreatorLineRichMenus`（`creatorLineChannelId` で絞り込み可） |
| `serviceId` | ❌ | ユーザーから受け取る（管理画面のサービス管理） |
| `benefitId` | ❌ | ユーザーから受け取る（管理画面の特典管理） |

`serviceId` / `benefitId` が必要な trigger/condition（`SERVICE_APPLIED` / `SERVICE_SCHEDULE_REMINDER` / `MARKETING_LEAD_BENEFIT_RECEIVED` / `CONDITION(SERVICE_APPLICATION_STATUS)`）では、要件ヒアリングの段階で「**該当リソースの ID を教えてください**」を必ず聞き、MCP からは検索・実在確認ができないことを踏まえてユーザー申告のまま扱う（下書き保存時は実在検証されないため、公開前レビューで改めて口頭確認する）。それ以外の ID は対応する一覧取得ツールで実在するものを選んでもらう（一覧を提示する際は名前で選んでもらい、生の ID を会話の主役にしない）。

## ライフサイクル運用

基本フロー: **`INACTIVE` で作る → ユーザー確認 → 公開（`ACTIVE` 化）**

- **作成直後**: 必ず `INACTIVE`
- **stages 構築中**: `INACTIVE` のまま何度でも更新する。下書き段階では参照 ID の実在はチェックされないので、公開前にユーザーへ口頭で確認する
- **公開（`ACTIVE` 化）**: 公開時に参照リソースの実在・同時稼働数の上限が確認され、満たさない場合は理由を示すエラーが返る
- **停止（`INACTIVE` 化）・廃止（`ARCHIVED`）**: `patchCreatorScenariosLifecycle` で行う

## 動作仕様

### 公開（`ACTIVE` 化）時の検証

- `ACTIVE` 化（公開）時に、参照リソースの実在と同時稼働数の上限が確認される。満たさない場合は理由を示すエラーが返る
  - 同時に稼働できるワークフロー数の上限に達している場合
  - 「指定された特典が見つかりません。特典が削除されていないか確認してください。」（トリガーの特典が削除済み等）
  - 「LINE関連の開始条件またはステップを使用するには、ワークフローにLINE公式アカウントを設定してください。」（LINE チャンネル未設定）
- 対処: エラーの原因（参照リソース不足・稼働数上限）を解消してから再度 `ACTIVE` 化する。ユーザーには平易な日本語で原因と対処を伝える

### 下書き（`INACTIVE`）中は検証が緩い

- 存在しない `benefitId` 等を入れても保存できてしまう
- 「保存できた」＝「動く」ではないので、参照 ID の実在をユーザーに口頭確認する
- 下書き中は不正な ID でも保存できてしまうので、公開前のレビュー段階で必ず ID 内容をユーザーに見せて確認する（公開時に検証され弾かれるが、手戻りを避けるため事前確認する）

### レスポンス JSON は送信時と形が違う

- `getCreatorScenario` のレスポンスでは、triggerType に該当しない trigger サブフィールドは `null` で返る
- ただし **`lineChannelContactRegisteredTrigger` だけはレスポンスから完全省略**される
- 受け取った JSON をそのまま PATCH に戻したい場合は、欠落フィールドを `null` で補完してから送る

### `getCreatorScenarios` の `lifecycle` クエリは2値のみ

- `"INACTIVE_ACTIVE"`（稼働 or 停止）か `"ARCHIVED"` のみ
- `"ACTIVE"` 単体や `"INACTIVE"` 単体は指定できない（クエリパラメータの enum）

## 埋め込み変数を使う際の注意

- 対応する変数は `{{line_name}}` / `{{guest_name}}` / `{{service_name}}` / `{{reservation_time_range}}` / `{{zoom_url}}` の5つのみ（詳細・使える条件は [content-schema.md](content-schema.md) 参照）。それ以外の名前は差し込まれない
- **外部ドキュメント（Excel等）の本文をそのまま転記しない**。`%event_date%` や `%line_name%` のような他ツール由来のプレースホルダはMOSHでは解釈されず、文字列としてそのまま配信されてしまう。転記時は必ず対応表と照合し、`{{...}}` 形式に置き換える
- `service_name` / `reservation_time_range` / `zoom_url` は `triggerType` が `SERVICE_APPLIED` か `SERVICE_SCHEDULE_REMINDER` の場合しか値が入らない。`LINE_CHANNEL_CONTACT_REGISTERED` や `CONTACT_TAG_ADDED` 起点のワークフローでこれらを使っても空文字になるだけで、保存・公開時にエラーにはならない（気づきにくい）
- `guest_name` は `SEND_EMAIL` 専用、`line_name` は `SEND_LINE_MESSAGE` 専用。逆の組み合わせでは常に空文字

## 命名規約

- ワークフロー名 (`name`) は管理画面でも表示されるので、用途がわかる名前にする
  - 良い例: `"ウェルカムメッセージワークフロー"`, `"単発レッスン申込後フォロー（3日後）"`
  - 悪い例: `"テスト"`, `"workflow_1"`
