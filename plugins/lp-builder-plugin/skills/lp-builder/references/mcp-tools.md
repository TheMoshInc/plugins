# MCP Tool リファレンス

LPビルダーで使用する taiyaki MCP ツールのパラメータ詳細。

## 一覧取得

**`mcp__taiyaki__getCreatorLandingPages`** — LP一覧を取得

```
queryParams: { status?, sort?, limit?, offset? }
```

- `status`: `"PUBLISHED_DRAFT"` | `"ARCHIVED"` | `"DRAFT"` | `"PUBLISHED"`（デフォルト: `"PUBLISHED_DRAFT"`）
- `sort`: `"updatedAtAsc"` | `"updatedAtDesc"`（デフォルト: `"updatedAtDesc"`）

**`mcp__taiyaki__getCreatorLandingPageTemplates`** — テンプレート一覧を取得

```
queryParams: { category?, limit?, offset? }
```

## 詳細取得

**`mcp__taiyaki__getCreatorLandingPage`** — LP詳細を取得

```
pathParams: { id }
```

**`mcp__taiyaki__getCreatorLandingPageTemplate`** — テンプレート詳細を取得

```
pathParams: { id }
```

## 作成・複製

**`mcp__taiyaki__postCreatorLandingPage`** — LPを作成

```
bodyParams: { templateId: number | null }
```

**`mcp__taiyaki__postCreatorLandingPageDuplicate`** — LPを複製

```
pathParams: { id }
```

## 更新

**`mcp__taiyaki__patchCreatorLandingPageDraft`** — LP下書きを更新

```
pathParams: { id }
bodyParams: { title?, content?, metaTitle?, metaDescription?, isSearchEngineEnabled?, expirationType?, relativeExpiration?, expireAt?, expirationAction?, redirectUrl?, serviceId?, scheduleDisplayCount?, ogpImageId?, faviconImageId?, metaPixelId?, googleAnalyticsId? }
```

- `expirationType`: `"NONE"` | `"RELATIVE"` | `"ABSOLUTE"`（`"ABSOLUTE"`は`expireAt`まで、`"RELATIVE"`は閲覧開始時刻から`relativeExpiration`後まで。`"NONE"`の場合ページは期限切れにならず、`expirationAction`/`redirectUrl`は参照されない）
- `relativeExpiration`: `{ days, hours, minutes, seconds }`（`0-9999`/`0-23`/`0-59`/`0-59`、全て必須。`"RELATIVE"`時に使用）
- `expireAt`: ISO 8601 datetime | `null`（`"ABSOLUTE"`時に使用）
- `expirationAction`: `"HIDE"` | `"REDIRECT"`（期限到達後にページを非表示にするかリダイレクトするか。`"REDIRECT"`でも`redirectUrl`が空/空白のみの場合は`"HIDE"`と同じ挙動になる）

**`mcp__taiyaki__patchCreatorLandingPageStatus`** — LPステータスを更新

```
pathParams: { id }
bodyParams: { status: "DRAFT" | "PUBLISHED" | "ARCHIVED" }
```
