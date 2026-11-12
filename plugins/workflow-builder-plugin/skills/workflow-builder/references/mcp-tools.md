# MCP Tool リファレンス

ワークフロービルダーで使用する MCP ツールのパラメータ詳細。

> ツール名は OpenAPI の `operationId`（例: `postCreatorScenario`）で記載する。MCP クライアントが実際に提示するツール名にはサーバー識別子のプレフィックス（例: `mcp__<server>__postCreatorScenario`）が付くが、その形はユーザー環境により異なるため本ドキュメントでは付けない。

## 一覧取得

**`getCreatorScenarios`** — ワークフロー一覧を取得

```
queryParams: { lifecycle?, limit?, offset?, folderId?, sort? }
```

- `lifecycle`: `"INACTIVE_ACTIVE"` | `"ARCHIVED"`（未指定時は全件。**`"ACTIVE"` / `"INACTIVE"` 単体は不可**）
- `limit`: デフォルト `20`
- `offset`: デフォルト `0`
- `folderId`: 管理画面のフォルダで絞り込む。文字列で `"unset"`（フォルダ未設定のみ）または フォルダID（例 `"1"`）。未指定は全件
- `sort`: `"createdAtAsc"` | `"createdAtDesc"` | `"nameAsc"` | `"nameDesc"` | `"updatedAtAsc"` | `"updatedAtDesc"`。未指定時は作成日時の降順（フォルダ指定かつ下書き・稼働中のみの絞り込みでは管理画面の並び順）
- レスポンスの各ワークフローに `folder`（`{ id, name }`）が付く。フォルダ操作は MCP に無い（SKILL.md「MCP でできないこと」参照）。提示は `folder.name`、ID は出さない
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
- 連携が切れたアカウントは `isConnected: false` で `displayName` が空文字のまま一覧に残る。ユーザーに選んでもらうのは `isConnected: true` のものだけにする
- `creatorLineChannelId`（`SEND_LINE_MESSAGE` / `LINE_CHANNEL_CONTACT_REGISTERED` / `LINE_KEYWORD_RESPONSE_MATCHED` / `LINK_LINE_RICH_MENU` で使用）の実在確認・`displayName` 変換に使う

**`getCreatorContactTags`** — コンタクトタグ一覧を取得

```
queryParams: { limit?, offset? }
```

- `limit`: デフォルト `100`、`offset`: デフォルト `0`
- レスポンス `contactTags[]`: `{ id, name, creatorId, createdAt, updatedAt }`
- `contactTagId`（`CONTACT_TAG_ADDED` / `ADD_CONTACT_TAG` / `REMOVE_CONTACT_TAG` / `CONDITION(CONTACT_TAG)` で使用）の実在確認に使う
- 指定された名前のタグが一覧に無い場合は、既存タグを提示して選び直してもらうか、`postCreatorContactTags` で新規作成できる

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
- `lineRichMenuId`（`LINK_LINE_RICH_MENU` で使用）の実在確認に使う。付与先はワークフローの `creatorLineChannelId` と同じチャンネルのものを選ぶ（API はクリエイター所有なら別チャンネルでも保存・公開を通すが、実行時に対象コンタクトが見つからず失敗しうる。一致を強制するのは編集画面のみ）

**`getCreatorLineKeywordResponses`** — LINEキーワード応答一覧を取得

```
queryParams: { creatorLineChannelId?, page?, limit? }
```

- 最終更新日時の降順。`page` デフォルト `1`、`limit` デフォルト `20`。レスポンスは `{ lineKeywordResponses[], totalCount }`
- 各要素: `{ id, name, creatorLineChannelId, keywords[], matchedFrequencyType, isResponseMessageActive, responseMessages[], status, matchedCount, updatedAt }`
- `id` が `LINE_KEYWORD_RESPONSE_MATCHED` の `lineKeywordResponseId`。`creatorLineChannelId` で絞り込んで、ワークフローと同じ LINE公式アカウントのものを選ぶ
- 一覧に出すときは `name`・`keywords`・LINE公式アカウント名（`getCreatorLineChannels` で変換）で選んでもらい、生の ID を出さない
- `matchedFrequencyType`: `once`（同じ友だちに初回のみ）| `everyTime`（毎回）。`isResponseMessageActive` が `true` だとキーワード応答側のメッセージも送られ、ワークフローの配信と**二重返信**になる

**`getCreatorLineKeywordResponse`** — LINEキーワード応答の詳細を取得（`pathParams: { id }`）。申告・選択された ID が意図したキーワード応答かの確認に使う

- キーワード応答の作成・更新（`postCreatorLineKeywordResponses` / `patchCreatorLineKeywordResponse`）の MCP ツールもあるが、このスキルの範囲外。キーワード応答自体の編集を頼まれたら別途案内する

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

- 一度も稼働していないワークフロー（下書きのまま等）は **404**（「リソースが存在しません」）。エラーではなく「まだ実行されていない」と伝える

- `LINE_KEYWORD_RESPONSE_MATCHED` の場合、`trigger.lineKeywordResponseMatchedTrigger` に `{ lineKeywordResponseId, name }`（キーワード応答の設定名）が入る（実機では未確認）。公開後に、申告された ID が意図したキーワード応答だったかを名前で確かめるのに使える
- レスポンスは `trigger`（トリガー種別と設定）と `steps`（各ステージの `totalCount` / `completedCount` / `failedCount`）を含む集計サマリー
- 条件分岐の子ステップは `steps` 配列内に再帰的にネストされる
- 各ステップの `stageSnapshotId` は、次の `getCreatorScenariosStageSnapshotExecutions` に渡す
- 件数集計だけで足りる場合はこのツールで止める（個別の実行履歴は個人情報を含むため、必要なときだけ下のツールで取る）

**`getCreatorScenariosStageSnapshotExecutions`** — ステップ1つの実行対象ごとの成否・失敗理由を取得

```
pathParams: { id, stageSnapshotId }
queryParams: { page?, executionStatus? }
```

- `id` はワークフローID、`stageSnapshotId` は `getCreatorScenariosExecutions` の `steps[].stageSnapshotId`（条件分岐の子ステップは `steps[].steps[]` の中）
- `executionStatus`: `"COMPLETED_FAILED"`（既定・すべて）| `"COMPLETED"` | `"FAILED"`。「届かなかった人」を調べるときは `"FAILED"`
- 実行日時の新しい順に **1ページ50件**。`currentPage < totalPage` なら `page` を1つ増やして続きを取る。`totalPage` が `0` なら該当なし
- 失敗理由は `errorCode` / `errorDetailCode` の組。成功した実行は両方とも空文字:
  - `QUOTA_LIMIT_ERROR` / `LINE_QUOTA_LIMIT`: LINE公式アカウントの配信上限超過 ／ `MOSH_QUOTA_LIMIT`: プランの配信上限超過
  - `TARGET_USER_NOT_FOUND` / `CONTACT_NOT_FOUND`・`GUEST_NOT_FOUND`: 送信先のコンタクト／ゲストが存在しない（紐付け未成立の典型）
  - `TARGET_RESOURCE_NOT_FOUND` / `CONTACT_TAG_NOT_FOUND`・`CREATOR_LINE_CHANNEL_NOT_FOUND`（連携解除）・`LINE_RICH_MENU_NOT_FOUND`
  - `REJECTED` / `LINE_BLOCKED`: 友だちにブロックされている
  - `UNKNOWN_ERROR` / 空文字: 想定外のエラー
- レスポンスに表示名・メールアドレス等の**個人情報が含まれる**。依頼された用途以外に出力・転記しない。ユーザーへは件数と失敗理由の日本語での要約を返し、個人を特定できる一覧は求められたときだけ出す
- `contactLine.id` は `getCreatorContactLine`、`contactEmail.id` は `getCreatorContactEmail` の `id` にそのまま渡せる
- 他人のワークフロー・存在しない ID・そのワークフローに属さない `stageSnapshotId` は 404

## テスト送信

**`postCreatorScenarioTestRun`** — ステップ1件をテスト送信（`SEND_LINE_MESSAGE` / `SEND_EMAIL` のみ）

```
pathParams: { id }
bodyParams: { action, target, embeddedVariables }
```

- `action`: `{ actionType, sendEmail, sendLineMessage }`。`actionType` に対応する1つだけ実オブジェクト、もう一方は `null`。中身は `getCreatorScenario` で取得したステップの `sendEmail` / `sendLineMessage` を元にする。**下書き保存前の編集中の内容も送れる**
- `sendLineMessage.messages[]` は本番と同じ形だが **`postbackActions` / `urlActions` は送れない**ので除く（タップ時のタグ付与などはテスト送信で確認できない）
- `target`: `SEND_LINE_MESSAGE` は必須 `{ contactId, creatorLineChannelId }`、`SEND_EMAIL` は `null`（操作中の MOSH アカウントのメールアドレスへ届く）
  - `contactId` は `postCreatorContactsLineSearch` の結果の `contactId`（`id` ではない）。`creatorLineChannelId` は同じ LINE公式アカウントのID（ワークフローの `creatorLineChannelId`。`null` なら `getCreatorLineChannels` で選ぶ）
- `embeddedVariables`: `{ lineName, guestName, serviceName, reservationTimeRange, zoomUrl }` を**全て必須**で渡す。実データは参照されず渡した値がそのまま展開される。値が無い項目は空文字、ただし **`zoomUrl` だけは URI 必須**（空文字不可）
- テスト送信は効果測定・配信実績・ステップ通過人数に**含まれない**。一方で **LINE公式アカウントの月間通数とMOSHプランの配信枠は消費する**
- `SEND_LINE_MESSAGE` は**実在する友だちに実際に届く**。送信先・内容・通数消費をユーザーに提示し、同意を得てから呼ぶ（勝手に試し送りしない）
- 拒否されたときは 422 で `errorDetails[0]` に `errorCode` / `errorDetailCode`（上の失敗理由と同じ体系）が返る。平易な日本語で原因と対処を伝える

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

## LINE 動画メッセージ用アセット（`VIDEO` メッセージの `muxAssetId`）

**`postCreatorLineMessageMuxAsset`** — 動画アップロード先 URL を発行

```
（パラメータなし）
```

- 返り値: `{ id, uploadUrl }`。`id` がそのまま `VIDEO` メッセージの `muxAssetId` になる
- `uploadUrl` へ実体（mp4、**200MBまで**）を Bash から PUT する（この PUT は MCP が代行しない）
- 手順の全体像は [content-schema.md](./content-schema.md) の「VIDEO の `muxAssetId` の取得方法」参照

**`getCreatorLineMessageMuxAsset`** — 動画アセットの変換ステータスを取得

```
pathParams: { id }
```

- `id` は `postCreatorLineMessageMuxAsset` が返した `muxAssetId`
- `staticRenditions` に `name: "highest.mp4"` かつ `status: "ready"` の要素が現れるまで**10秒間隔**でポーリングする（`playbackId` が空文字の間は未完了）
- `status` が `ERRORED` になったら `postCreatorLineMessageMuxAsset` からやり直す

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
