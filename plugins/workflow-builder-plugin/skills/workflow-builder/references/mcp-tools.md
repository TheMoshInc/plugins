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
- ユーザーに一覧を提示する際は `creatorLineChannelId` を生の ID のまま出さず、`getCreatorLineChannels` の結果と突き合わせて `displayName` に変換して表示する（`null` は「未設定」と表示）

**`getCreatorScenariosInflowActions`** — 流入経路一覧を取得

```
queryParams: { creatorLineChannelId?, limit? }
```

- `creatorLineChannelId` 未指定時はクリエイター配下の全件を返す
- ユーザーに一覧を提示する際は `creatorLineChannelId` を生の ID のまま出さず、`getCreatorLineChannels` の結果と突き合わせて `displayName` に変換して表示する

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

**`getCreatorContactTags`** — コンタクトタグ一覧を取得

```
queryParams: { limit?, offset? }
```

- `limit`: デフォルト `100`、`offset`: デフォルト `0`
- レスポンス `contactTags[]`: `{ id, name, creatorId, createdAt, updatedAt }`
- `contactTagId`（`CONTACT_TAG_ADDED` / `ADD_CONTACT_TAG` / `REMOVE_CONTACT_TAG` / `CONDITION(CONTACT_TAG)` で使用）の実在確認に使う
- 指定された名前のタグが一覧に無い場合は、既存タグを提示して選び直してもらうか、`postCreatorContactTags` で新規作成できる

**`getCreatorLineRichMenus`** — 個別リッチメニュー一覧を取得

```
queryParams: { creatorLineChannelId? }
```

- `creatorLineChannelId` で絞り込み可（未指定時は全件）
- レスポンス `lineRichMenus[]`: `{ id, name, creatorLineChannelId, isDefault, linkedCount, backgroundMoshImageId, updatedAt }`
- `lineRichMenuId`（`LINK_LINE_RICH_MENU` で使用）の実在確認に使う。付与先はワークフローの `creatorLineChannelId` と同じチャンネルのものを選ぶ（API はクリエイター所有なら別チャンネルでも保存・公開を通すが、実行時に対象コンタクトが見つからず失敗しうる。一致を強制するのは編集画面のみ）

## 詳細取得

**`getCreatorScenario`** — ワークフロー詳細を取得

```
pathParams: { id }
```

- レスポンスは `stages` 含む完全形

**`getCreatorScenariosInflowAction`** — 流入経路詳細を取得

```
pathParams: { id }
```

## 実行結果取得

**`getCreatorScenariosExecutions`** — ワークフロー実行結果サマリーを取得

```
pathParams: { id }
```

- レスポンスは `trigger`（トリガー種別と設定）と `steps`（各ステージの `totalCount` / `completedCount` / `failedCount`）を含む集計サマリー
- 条件分岐の子ステップは `steps` 配列内に再帰的にネストされる
- ステージ別の個別実行履歴（顧客情報を含むドリルダウン）は取得できない。件数集計のみ必要な場合に使う

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
- **ユーザーの明示的な同意なしに呼び出してはならない**。事前に合意内容（トラッキング利用に関する規約への同意）をユーザーに提示し、同意の意思を確認してから呼び出す

**`postCreatorContactTags`** — コンタクトタグを新規作成

```
bodyParams: { name: string(1-64) }
```

- タグ名はクリエイター内で一意。**同名タグが既に存在する場合は新規作成せず既存タグの `id` を返す（冪等）**
- 返り値: `{ id }`（作成したタグ、または既存の同名タグの ID）。そのまま `contactTagId`（`CONTACT_TAG_ADDED` / `ADD_CONTACT_TAG` / `REMOVE_CONTACT_TAG` / `CONDITION(CONTACT_TAG)`）に指定できる

**`postCreatorScenariosInflowAction`** — 流入経路を作成

```
bodyParams: { name: string(1-255), creatorLineChannelId: number, conversionFrequencyType: "once" | "everyTime" }
```

- `conversionFrequencyType`: `once`=同一コンタクトへの初回のみ発火、`everyTime`=毎回発火
- 返り値: `{ id }`（作成された流入経路の ID）
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

**`patchCreatorScenariosInflowAction`** — 流入経路を更新

```
pathParams: { id }
bodyParams: { name?, conversionFrequencyType? }
```

- 部分更新。指定したフィールドのみ変更する（未指定のフィールドは維持）
- `creatorLineChannelId` は変更不可（作成時のみ指定）
