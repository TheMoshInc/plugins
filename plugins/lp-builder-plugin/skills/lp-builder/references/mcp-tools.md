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
