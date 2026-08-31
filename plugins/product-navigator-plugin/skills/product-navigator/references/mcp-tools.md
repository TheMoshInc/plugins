# MCP Tool リファレンス

商品ナビゲーターで使用する MCP ツールのパラメータ詳細。

> ツール名は OpenAPI の `operationId` で記載する。MCP クライアントが実際に提示するツール名にはサーバー識別子の接頭辞（例: `mcp__<server>__getCreatorProducts`）が付くが、その形はユーザー環境により異なるため本ドキュメントでは付けない。

## 商品一覧

**`getCreatorProducts`** — 商品一覧を取得

```
queryParams: { limit?, offset?, status? }
```

- `limit`: 1〜100（既定 `20`）
- `offset`: 0〜（既定 `0`）
- `status`: `"PUBLIC"` | `"LIMITED"` | `"PRIVATE"`（省略時は全件）

レスポンス: `{ products: [...], totalCount }`。`totalCount` は**総件数**。取得済み件数より多いときは `offset` をずらして続きを取得する。

## 商品詳細

**`getCreatorProduct`** — 商品詳細を取得

```
pathParams: { productId }
```

- `productId`: 数値文字列（`^\d+$`）。`getCreatorProducts` の結果の `id` を渡す
- 自分が所有していない・存在しない商品を指定すると「見つからない」エラーが返る

## 商品プラン一覧

**`getCreatorProductPlans`** — 指定商品に紐づくプラン一覧を取得

```
pathParams: { productId }
queryParams: { limit?, offset? }
```

- `limit` / `offset`: **現状は無視され、ページングは行われない**（全件が 1 回で返る）
- 自分の商品でない・存在しない `productId` でもエラーにならず、`productPlans` が空配列・`totalCount` が 0 になる

レスポンス: `{ productPlans: [...], totalCount }`。`totalCount` は**返した件数**（総件数ではない）。続きの取得は不要。

## 商品プラン詳細

**`getCreatorProductPlan`** — 商品プラン詳細を取得

```
pathParams: { productId, productPlanId }
```

- いずれも数値文字列（`^\d+$`）。`productPlanId` は `getCreatorProductPlans` の結果の `id` を渡す
- 自分が所有していない・存在しないプランを指定すると「見つからない」エラーが返る
