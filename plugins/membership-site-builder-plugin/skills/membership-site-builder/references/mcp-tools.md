# MCP Tool リファレンス

membership-site-builder で使用する MCP ツール 18 件のパラメータ詳細。

> ツール名は OpenAPI の `operationId` で記載する。MCP クライアントが実際に提示するツール名にはサーバー識別子の接頭辞（例: `mcp__<server>__getCreatorMembershipSites`）が付くが、その形はユーザー環境により異なるため本ドキュメントでは付けない。

すべてのツールで、ログイン中クリエイターが所有していない会員サイトを指定すると「見つからない」応答になる（存在しない場合と区別できない）。

## 会員サイト

### `getCreatorMembershipSites` — 会員サイト一覧を取得

```
queryParams: { limit?, offset? }
```

- `limit`: 1 以上（既定 `20`）
- `offset`: 0 以上（既定 `0`）

レスポンス: `{ membershipSites: [...], totalCount }`。`totalCount` は**総件数**。取得済み件数より多いときは `offset` をずらして続きを取得する。1 件も無い場合は空配列・`totalCount: 0`（エラーではない）。

各要素: `id` / `slug` / `name` / `isPublished` / `headerLogoImageId` / `createdAt` / `updatedAt`。

### `getCreatorMembershipSite` — 会員サイト詳細を取得

```
pathParams: { id }
```

サイトの設定値 7 項目（`name` / `isPublished` / `isFixedViewingOrder` / `headerLogoImageId` / `themeColor` / `homeIconImageId` / `homeTitle`）に `id` / `slug` / `createdAt` / `updatedAt` を加えて返す。フォルダ・コンテンツ・タグは含まれない。

**`patchCreatorMembershipSite` の前には必ずこれを呼ぶ**（全置換のため、変えない項目の現在値が要る）。

### `postCreatorMembershipSites` — 会員サイトを作成

```
bodyParams: { name }
```

- `name`: 1〜100 文字

指定できるのは名前だけ。作成直後は**非公開**で、他の設定は初期値になる。レスポンスは `{ id, slug }`。

作成できるサイト数の上限を超えると、上限超過を示す応答になる。

### `patchCreatorMembershipSite` — 会員サイトを更新

```
pathParams: { id }
bodyParams: { name, isPublished, isFixedViewingOrder, headerLogoImageId, themeColor, homeIconImageId, homeTitle }
```

**7 項目すべて必須の全置換。** 省くと弾かれる（省いても消えはしない）。

- `name`: 1〜100 文字
- `isPublished`: 公開 / 非公開
- `isFixedViewingOrder`: 設定順に閲覧させる制約。`true` → `false` は**全ゲストの閲覧完了状態が消え、元に戻せない**
- `headerLogoImageId` / `homeIconImageId`: 文字列 または `null`（`null` で外す。MCP から新しい画像は付けられない）
- `themeColor`: `#` + 16 進 6 桁（例: `#FA6F78`）
- `homeTitle`: ホーム画面表示名

閲覧順の固定を ON にできないときの理由コード:

| コード | 意味 |
|---|---|
| `BLOCKED_BY_ACTIVE_GUESTS` | 有効な閲覧権限を持つ会員が既にいる（期限切れ・停止中・ブロック済みは数えない） |
| `BLOCKED_BY_FOLDER_SPLIT_LICENSE` | 一部のフォルダだけを見せる閲覧権限がある。今後追加されるフォルダを自動で閲覧可にしない設定の権限があり、現時点で全フォルダを見せている場合も含む |

非公開から公開に変えるときだけ、公開できるサイト数の上限を判定する。

### `deleteCreatorMembershipSite` — 会員サイトを削除

```
pathParams: { id }
```

復元できない。消える範囲は [content-schema.md](content-schema.md) の「削除で消えるもの」を参照。商品のプランで提供中のサイトは削除できない。

## フォルダ

### `getCreatorMembershipSiteFolders` — フォルダ一覧を取得

```
pathParams: { id }
```

**サイトの中身を見る唯一の入口。** コンテンツだけを一覧するツールは存在しない。

各フォルダが `id` / `name` / `displayType` / `isPublished` / `createdAt` / `updatedAt` と、配下コンテンツの概要 `contents` を表示順で持つ。コンテンツ概要に入るのは `id` / `slug` / `title` / `description` / `thumbnailUrl` / `isPublished` / `contentType` / `scheduledPublishAt` / `scheduledUnpublishAt` / `visibleAfterPurchaseDays` / `tagNames` / `tagIds` / `createdAt` / `updatedAt`。**本文とチャプターは含まない。**

ページングは無く、全フォルダ・全コンテンツの概要が 1 回で返る。概要は 1 件あたり最大 10,000 文字のため、コンテンツ数が多いサイトでは応答が大きくなる。

### `postCreatorMembershipSiteFolders` — フォルダを作成

```
pathParams: { id }
bodyParams: { name, displayType, isPublished }
```

- `name`: 1〜100 文字
- `displayType`: `"CAROUSEL"` | `"FOLDER_VIEW"` | `"TILE"`（日本語表記は [content-schema.md](content-schema.md)）
- `isPublished`: `true` にすると、閲覧権限のある会員にすぐ公開される。下書きとして用意するなら `false`

レスポンスは `{ id }`。既存フォルダの末尾に追加される。

### `patchCreatorMembershipSiteFolder` — フォルダを更新

```
pathParams: { id, folderId }
bodyParams: { name?, displayType?, isPublished? }
```

送った項目だけが変わる。1 項目も送らないと弾かれる。`isPublished` を `false` にすると中のコンテンツは会員から見えなくなる（コンテンツ自体の公開状態は変わらない）。

### `deleteCreatorMembershipSiteFolder` — フォルダを削除

```
pathParams: { id, folderId }
```

復元できない。**中のコンテンツもまとめて消える。** 中身の有無は確認されないため、空でないフォルダでもそのまま消える。残ったフォルダの並び順は詰め直されない。

削除したフォルダは、そのフォルダを閲覧対象にしていた会員の閲覧権限からも外れる。そのフォルダだけを閲覧対象にしていた権限では対象が 0 件になり、購入済みの会員はサイトには入れるがコンテンツが 1 件も見えなくなる。フォルダを作り直しても、今後追加されるフォルダを自動で閲覧可にする設定が OFF の権限には戻らず、管理画面で付け直しが要る（MCP に閲覧権限を変えるツールは無い）。

## タグ

### `getCreatorMembershipSiteTags` — タグ一覧を取得

```
pathParams: { id }
```

`{ tags: [{ id, name }] }` を表示順で返す。1 件も無い場合は空配列。

### `postCreatorMembershipSiteTags` — タグを作成

```
pathParams: { id }
bodyParams: { name }
```

- `name`: 1〜50 文字。**サイト内で一意**

同じ名前のタグが既にあると弾かれる。作る前に必ず一覧を引き、あればその `id` を使い回す（コンテンツに付ける目的なら作成は不要）。弾かれたときは一覧を引き直して既存のタグ名をユーザーに示し、そのタグを使うか別の名前にするかを確認する。

タグが会員側の一覧に出るのは、その会員に見えているコンテンツに 1 件以上付いているときだけ。作成しただけでは会員サイトに出ない。

### `patchCreatorMembershipSiteTag` — タグを更新

```
pathParams: { id, tagId }
bodyParams: { name }
```

変えられるのは名前だけで、コンテンツとの結びつきは残る。既にある名前へは変更できない。

### `deleteCreatorMembershipSiteTag` — タグを削除

```
pathParams: { id, tagId }
```

復元できない。そのタグが付いていたコンテンツからは目印が外れるだけで、コンテンツ自体は消えない。実行前に `getCreatorMembershipSiteFolders` で、そのタグが付いているコンテンツの件数を数えて提示する（各コンテンツの概要に `tagIds` と `tagNames` が入っている）。

## コンテンツ

### `getCreatorMembershipSiteContent` — コンテンツ詳細を取得

```
pathParams: { id, contentId }
```

本文・チャプター・公開設定まで含めて 1 件を返す。`contentId` は `getCreatorMembershipSiteFolders` の `contents` から取る（コンテンツ ID だけを指定するツールは無く、サイトの `id` も必ず一緒に渡す）。

`body` は現在のエディタが保存した Tiptap JSON 文字列で返る。旧サイトから移行したコンテンツは HTML 断片、初期データはプレーンテキストのこともあるため、そのまま引用せずテキストを取り出して扱う。

会員の視聴進捗・コメント・ブックマークは含まれない。

### `postCreatorMembershipSiteContents` — コンテンツを作成

```
pathParams: { id }
bodyParams: 16 項目すべて必須（content-schema.md の表を参照）
```

`folderId` は `getCreatorMembershipSiteFolders`、`tagIds` は `getCreatorMembershipSiteTags` で先に取得する。レスポンスは `{ id }`。フォルダ内の末尾に追加される。

### `patchCreatorMembershipSiteContent` — コンテンツを更新

```
pathParams: { id, contentId }
bodyParams: { folderId?, title?, body?, description?, chapters?, thumbnailAssetId?, assetIds?, isPublished?, isNotifyOnPublish?, isCommentEnabled?, isCompletionButtonVisible?, scheduledPublishAt?, scheduledUnpublishAt?, visibleAfterPurchaseDays?, tagIds? }
```

送った項目だけが変わる。1 項目も送らないと弾かれる。各項目の型・上限は作成と同じ（[content-schema.md](content-schema.md)）。

- `chapters` / `assetIds` / `tagIds` は**配列ごと置き換わる**。現在値を読んでから全体を送る
- `contentType` は変更できない。種類を変えるなら作り直す
- `folderId` を送ると別のフォルダへ移せる。同じ会員サイト内のフォルダを指定する
- `scheduledPublishAt` と `visibleAfterPurchaseDays` の排他は、送らなかった項目に既存の値を当てはめてから判定される
- `thumbnailAssetId`: MCP から画像を上げられないため、新しいサムネイルは付けられない（外すなら `null`）

### `deleteCreatorMembershipSiteContent` — コンテンツを削除

```
pathParams: { id, contentId }
```

復元できない。会員の視聴進捗・ブックマーク・コメントも一緒に消え、予約していた公開通知も送られなくなる。タグ・動画・画像との結びつきは外れるが、タグそのものとアップロード済みの動画・画像は残る。

### `postCreatorMembershipSiteContentDuplicate` — コンテンツを複製

```
pathParams: { id, contentId }
```

似た内容を作るときは、一から組み立てるよりこちらが早い。複製後の書き換えは `patchCreatorMembershipSiteContent` で行う。決まった動きは次のとおりで、変えられない。

- タイトルは「〈元のタイトル〉のコピー」になる。合計が 200 文字を超える場合は元のタイトルが切り詰められるため、実行後の値を `getCreatorMembershipSiteContent` で確認する
- 必ず非公開で作られる
- チャプター・タグ・動画や画像との結びつきは引き継がれる
- 予約公開・予約非公開・購入後◯日で公開の設定は引き継がれない
- コメントの受け付けは引き継がれず、必ず無効になる
- 複製先は元と同じフォルダで、別のフォルダは指定できない

レスポンスは複製後の `{ id }`。実行後は、タイトルが変わることと非公開で作られることをユーザーに伝える。
