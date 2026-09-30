# MCP Tool リファレンス

LPビルダーで使用する taiyaki MCP ツールのパラメータ詳細。ツール名は operationId のみで書く。実際の呼び出し名には `mcp__<server>__` の接頭辞が付く（プラグイン経由なら `mcp__plugin_lp-builder-plugin_taiyaki__`、個人登録なら登録名に依存）が、環境で変わるため本書には付けない。接続中のツール一覧から末尾が一致するものを使う。

## 事前調査で使う参照系（LP 系以外）

ヒアリングシートの仮埋め（Step 1）と画像収穫（#7）で読む。いずれも読み取り専用。

| ツール | 用途 |
|---|---|
| `getCreatorProducts` / `getCreatorProduct` | 商品名・説明・価格・商品画像（ヒアリング #1・#7） |
| `getCreatorProductPlans` | プランの価格・期間（ゴール C/D の判定材料） |
| `getCreatorLandingPages` / `getCreatorLandingPage` | 既存 LP のトーン・実績表現・使用画像 |
| `getCreatorMaterials` / `getCreatorMaterialImage` | 素材ライブラリの画像（公開 URL を取得して `image.src` に使う） |

## 一覧取得

**`getCreatorLandingPages`** — LP一覧を取得

```
queryParams: { status?, sort?, limit?, offset? }
```

- `status`: `"PUBLISHED_DRAFT"` | `"ARCHIVED"` | `"DRAFT"` | `"PUBLISHED"`（デフォルト: `"PUBLISHED_DRAFT"`）
- `sort`: `"updatedAtAsc"` | `"updatedAtDesc"`（デフォルト: `"updatedAtDesc"`）

**`getCreatorLandingPageTemplates`** — テンプレート一覧を取得

```
queryParams: { category?, limit?, offset? }
```

## 詳細取得

**`getCreatorLandingPage`** — LP詳細を取得

```
pathParams: { id }
```

**`getCreatorLandingPageTemplate`** — テンプレート詳細を取得

```
pathParams: { id }
```

## 作成・複製

**`postCreatorLandingPage`** — LPを作成

```
bodyParams: { templateId: number | null }
```

**`postCreatorLandingPageDuplicate`** — LPを複製

```
pathParams: { id }
```

## 更新

**`patchCreatorLandingPageDraft`** — LP下書きを更新

```
pathParams: { id }
bodyParams: { title?, content?, metaTitle?, metaDescription?, isSearchEngineEnabled?, expirationType?, relativeExpiration?, expireAt?, expirationAction?, redirectUrl?, serviceId?, scheduleDisplayCount?, ogpImageId?, faviconImageId?, metaPixelId?, googleAnalyticsId? }
```

- `expirationType`: `"NONE"` | `"RELATIVE"` | `"ABSOLUTE"`（`"ABSOLUTE"`は`expireAt`まで、`"RELATIVE"`は閲覧開始時刻から`relativeExpiration`後まで。`"NONE"`の場合ページは期限切れにならず、`expirationAction`/`redirectUrl`は参照されない）
- `relativeExpiration`: `{ days, hours, minutes, seconds }`（`0-9999`/`0-23`/`0-59`/`0-59`、全て必須。`"RELATIVE"`時に使用）
- `expireAt`: ISO 8601 datetime | `null`（`"ABSOLUTE"`時に使用）
- `expirationAction`: `"HIDE"` | `"REDIRECT"`（期限到達後にページを非表示にするかリダイレクトするか。`"REDIRECT"`でも`redirectUrl`が空/空白のみの場合は`"HIDE"`と同じ挙動になる）

**`patchCreatorLandingPageStatus`** — LPステータスを更新

```
pathParams: { id }
bodyParams: { status: "DRAFT" | "PUBLISHED" | "ARCHIVED" }
```

## 画像・動画のアップロード（MCP対応済み）

**`image` / `video` 要素は、ユーザーから手元のファイルを渡された場合、以下のツールで MCP から直接アップロードできる。** 外部URLをそのまま `attributes.src` に設定してよいかどうかは要素ごとに異なる（`content-schema.md` の「`image` にリンクを設定する」「`video` の仕様」参照）。

**`postCreatorLandingPagesImages`** — LPに画像を作成しアップロード先URLを発行

```
pathParams: { id }
bodyParams: { name: string, mimeType: "image/jpeg" | "image/png" | "image/webp" | "image/gif" }
```

レスポンス `{ id, moshImageId, fileKey, uploadUrl }` の `uploadUrl`（署名付きURL・**有効期限15分**）へ、Bash から `curl -X PUT --data-binary @<file> -H "Content-Type: <mimeType>" "<uploadUrl>"` で画像本体を PUT する（MCP はこの PUT を代行しない）。

**`getCreatorLandingPagesImages`** — LPの画像一覧を取得

```
pathParams: { id }
```

アップロード後、`images[].status` が `"ready"` になるまでポーリングする。`ready` になった要素の `url` を `image` 要素の `attributes.src` に使う。

**`deleteCreatorLandingPagesImages`** — LPの画像を削除

```
pathParams: { id, imageId }
```

**`postCreatorLandingPagesVideos`** — LPに動画を登録しアップロード先URLを発行

```
pathParams: { id }
bodyParams: { name: string, mimeType: "video/mp4" | "video/quicktime", durationSec: number }
```

レスポンス `{ id, moshVideoId, fileKey, uploadUrl }` の `uploadUrl`（署名付きURL・**有効期限15分・100MBまで**）へ動画本体を PUT する。

**`getCreatorLandingPagesVideos`** — LPの動画一覧を取得

```
pathParams: { id }
```

アップロード後、`videos[].status` が `"ready"` になるまでポーリングする（変換処理があるため画像より時間がかかる）。`ready` になったら `moshVideoId` と `url` の両方を `video` 要素の `attributes.moshVideoId` / `attributes.src` にそのまま設定する（**両方セットで初めて有効**。片方だけだと編集画面クラッシュ。`content-schema.md` 「`video` の仕様」参照）。

**`deleteCreatorLandingPagesVideo`** — LPの動画を削除

```
pathParams: { id, videoId }
```
