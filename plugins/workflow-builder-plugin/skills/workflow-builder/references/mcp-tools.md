# MCP Tool リファレンス

ワークフロービルダーで使用する MCP ツールのパラメータ詳細。

> ツール名は OpenAPI の `operationId`（例: `postCreatorScenario`）で記載する。MCP クライアントが実際に提示するツール名にはサーバー識別子のプレフィックス（例: `mcp__<server>__postCreatorScenario`）が付くが、その形はユーザー環境により異なるため本ドキュメントでは付けない。

## 一覧取得

**`getCreatorScenarios`** — ワークフロー一覧を取得

```
queryParams: { lifecycle?, limit?, offset? }
```

- `lifecycle`: `"INACTIVE_ACTIVE"` | `"ARCHIVED"`（未指定時は全件。**`"ACTIVE"` / `"INACTIVE"` 単体は不可**）
- `limit`: デフォルト `20`
- `offset`: デフォルト `0`

**`getCreatorScenariosInflowActions`** — 流入アクション一覧を取得

```
queryParams: { creatorLineChannelId?, limit? }
```

- `creatorLineChannelId` 未指定時はクリエイター配下の全件を返す

**`getCreatorScenariosTrackingConsents`** — トラッキング利用合意状態を取得

```
（パラメータなし）
```

**`getCreatorLineChannels`** — LINE公式アカウント一覧を取得

```
（パラメータなし）
```

- レスポンス `channels[]`: `{ id, officialLineAccountId, displayName, isConnected, createdAt }`
- `creatorLineChannelId`（`SEND_LINE_MESSAGE` / `LINE_CHANNEL_CONTACT_REGISTERED` / `LINK_LINE_RICH_MENU` で使用）の実在確認・`displayName` 変換に使う

**`getCreatorContactTags`** — 顧客タグ一覧を取得

```
queryParams: { limit?, offset? }
```

- `limit`: デフォルト `100`、`offset`: デフォルト `0`
- レスポンス `contactTags[]`: `{ id, name, creatorId, createdAt, updatedAt }`
- `contactTagId`（`CONTACT_TAG_ADDED` / `ADD_CONTACT_TAG` / `REMOVE_CONTACT_TAG` / `CONDITION(CONTACT_TAG)` で使用）の実在確認に使う

**`getCreatorAutoWebinars`** — オートウェビナー一覧を取得

```
queryParams: { sort?, limit? }
```

- `sort`: `"updatedAtAsc"` | `"updatedAtDesc"`（デフォルト `updatedAtDesc`）、`limit`: デフォルト `20`
- レスポンス `autoWebinars[]`: `{ id, title, thumbnailId, updatedAt }`
- `autoWebinarId`（`CONDITION(AUTO_WEBINAR_PARTICIPATION` / `AUTO_WEBINAR_WATCH_TIME)` で使用）の実在確認に使う

**`getCreatorLineRichMenus`** — 個別リッチメニュー一覧を取得

```
queryParams: { creatorLineChannelId? }
```

- `creatorLineChannelId` で絞り込み可（未指定時は全件）
- レスポンス `lineRichMenus[]`: `{ id, name, creatorLineChannelId, isDefault, linkedCount, backgroundMoshImageId, updatedAt }`
- `lineRichMenuId`（`LINK_LINE_RICH_MENU` で使用）の実在確認に使う。付与先はワークフローの `creatorLineChannelId` と同じチャンネルが持つものに限る

## 詳細取得

**`getCreatorScenario`** — ワークフロー詳細を取得

```
pathParams: { id }
```

- レスポンスは `stages` 含む完全形

**`getCreatorScenariosInflowAction`** — 流入アクション詳細を取得

```
pathParams: { id }
```

## 作成・複製

**`postCreatorScenario`** — ワークフローを作成

```
bodyParams: { name: string }
```

- 必須は `name` のみ。`stages` は作成後に PATCH で追加する
- 作成直後のライフサイクルは `INACTIVE`
- 返り値: `{ id }`

**`postCreatorScenarioDuplicate`** — ワークフローを複製

```
pathParams: { id }
```

- 返り値: `{ id }`（複製後の新ワークフロー ID）
- 複製直後のライフサイクルも `INACTIVE`（元が `ACTIVE` でも複製は稼働しない）

**`postCreatorScenariosTrackingConsent`** — トラッキング利用に合意

```
（パラメータなし）
```

- 同意状態を記録するだけのアクション

**`postCreatorScenariosInflowAction`** — 流入アクションを作成

```
bodyParams: { name: string(1-255), creatorLineChannelId: number, conversionFrequencyType: "once" | "everyTime" }
```

- `conversionFrequencyType`: `once`=同一コンタクトへの初回のみ発火、`everyTime`=毎回発火
- 返り値: `{ id }`（作成された流入アクションの ID）
- `slug` / `inflowUrl` はサーバー側で自動発番（リクエストでは指定しない）

## 更新

**`patchCreatorScenario`** — ワークフローを更新

```
pathParams: { id }
bodyParams: { name?, creatorLineChannelId?, stages? }
```

- `creatorLineChannelId` は `number | null`
- `stages` は配列まるごと差し替え。各要素は `{ trigger, action }` で、`trigger` と `action` の構造は [content-schema.md](./content-schema.md) を参照
- `stages[].action` の型は **`scenarioActionForUpdate`**（POST 用の `scenarioAction` とは別）

**`patchCreatorScenariosLifecycle`** — ライフサイクルを更新

```
pathParams: { id }
bodyParams: { lifecycle: "INACTIVE" | "ACTIVE" | "ARCHIVED" }
```

- 起動: `ACTIVE`、停止: `INACTIVE`、廃止: `ARCHIVED`
- 一覧取得のクエリ enum とは別物（`getCreatorScenarios` の `lifecycle` は `"INACTIVE_ACTIVE"` / `"ARCHIVED"` のみ）

**`patchCreatorScenariosInflowAction`** — 流入アクションを更新

```
pathParams: { id }
bodyParams: { name?, conversionFrequencyType? }
```

- 部分更新。指定したフィールドのみ変更する（未指定のフィールドは維持）
- `creatorLineChannelId` は変更不可（作成時のみ指定）
