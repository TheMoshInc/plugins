# コンタクト検索の構造（正本）

`postCreatorContactsEmailSearch` / `postCreatorContactsLineSearch` のリクエスト・レスポンス構造の正本。ツールごとの制約・エラーは [mcp-tools.md](mcp-tools.md) を見る。

> 本ファイルの構造・演算子・上限は、MOSH の API スキーマ定義で確認したもの（2026-09 時点）。
> 値を書き換えるときは、記憶や推測ではなく MOSH の API スキーマ定義で裏取りする。

## 共通の骨格

```jsonc
{
  "keyword": "",        // 必須。部分一致・大文字小文字を区別しない。空文字で絞り込みなし
  "limit": 20,          // 必須。1〜100
  "offset": 0,          // 必須。0以上
  "sort": "createdAtDesc", // 必須。enum は下記
  "filters": { /* 必須。全キーを書く。使わないキーは null */ }
}
```

- **`filters` は「全キー必須」**。使わない条件も `null` を明示する。キーを省くとリクエスト不正になる
- 単項の条件は `{ "value": <値>, "operator": "eq" | "ne" }` の形。日付範囲だけ `{ "from": ..., "to": ... }`（両キー必須、各々 `null` 可）
- `filters.contact.active` を省略（`null`）すると `active: true` が補完される＝**既定では削除済みは返らない**。削除済みを含めたいときだけ明示指定する

### keyword が当たる先

| ツール | keyword の対象 |
|---|---|
| メール | メールアドレス |
| LINE | LINE の表示名 |

「名前で探して」と言われても、メール検索は氏名では引けない（メールアドレスにしか当たらない）。

### sort の enum

| ツール | 指定できる値 |
|---|---|
| メール | `emailAsc` / `emailDesc` / `contactIdAsc` / `contactIdDesc` / `createdAtAsc` / `createdAtDesc` / `moshIdAsc` / `moshIdDesc` |
| LINE | `nameAsc` / `nameDesc` / `contactIdAsc` / `contactIdDesc` / `createdAtAsc` / `createdAtDesc` / `moshIdAsc` / `moshIdDesc` |

メールに `nameAsc`、LINE に `emailAsc` は無い。

## メール検索の filters

```jsonc
{
  "contact": {                       // null 可
    "active":     { "value": true, "operator": "eq" },   // null 可。省略時 active: true 相当
    "subscribed": { "value": true, "operator": "eq" }    // null 可。メール配信の購読状態
  },
  "email": {                         // null 可
    "createdAt": { "from": "2026-01-01T00:00:00+09:00", "to": null },  // null 可。from/to 両キー必須
    "address":   { "value": "example.com", "operator": "eq" }          // null 可
  },
  "tag": null                        // 下記「タグ条件」
}
```

- 必須キーは `contact` / `email` / `tag` の3つ
- **実際に配信が届くのは購読中のコンタクトだけ**。配信対象を数えるときは `subscribed` に `{ "value": true, "operator": "eq" }` を入れる

## LINE検索の filters

```jsonc
{
  "contact": {                       // null 可
    "active":  { "value": true,  "operator": "eq" },   // null 可
    "blocked": { "value": false, "operator": "eq" }    // null 可。ブロック状態
  },
  "line": {                          // null 可
    "createdAt": { "from": null, "to": null },          // null 可。from/to 両キー必須
    "name":      { "value": "佐藤", "operator": "eq" }  // null 可。表示名の部分一致
  },
  "tag": null,
  "inflowAction": null,              // 流入経路。{ "id": { "value": 12, "operator": "eq" } }
  "richMenu": null                   // 表示中リッチメニュー。{ "id": { "value": 3, "operator": "eq" } }
}
```

- 必須キーは `contact` / `line` / `tag` / `inflowAction` / `richMenu` の5つ
- リクエスト直下に任意の `lineChannelId`（整数）を置ける。**未指定なら全 LINE公式アカウントを横断**。特定アカウントに絞るときだけ指定する
- `inflowAction` の `operator`: `eq` = 指定した流入経路からの流入記録があるコンタクト / `ne` = 記録がないコンタクト（**流入記録が1件もないコンタクトを含む**）
- `richMenu` の `operator`: `eq` = そのリッチメニューが表示されているコンタクト / `ne` = 表示されていないコンタクト

## タグ条件（メール・LINE 共通）

```jsonc
"tag": { "id": [ { "value": 1, "operator": "eq" }, { "value": 5, "operator": "ne" } ] }
```

- 条件は 1〜5 件。**複数指定は AND**（すべてを満たすコンタクトだけが対象。いずれかを満たす OR ではない）
- `eq` = そのタグが付いている / `ne` = 付いていない
- `value` に入れるタグIDは `getCreatorContactTags` で名前から解決する（推測しない）
- 「AまたはB」を求められたら、1タグずつ検索して結果を突き合わせる（API 側では表現できない）

## レスポンス

```jsonc
{
  "contacts": [ /* 下記の項目 */ ],
  "filteredCount": 21,   // 条件に一致した件数 ← 「◯件です」はこれ
  "totalCount": 350      // 絞り込み前の有効なコンタクト数（削除済みを含まない）
}
```

- LINE で `lineChannelId` を指定した場合、`totalCount` はそのアカウント内の件数になる
- 該当0件はエラーではない。`contacts: []` / `filteredCount: 0` が返る
- 1ページ最大100件。続きは `offset` を増やす

### 1件あたりの項目

| 項目 | メール | LINE | 備考 |
|---|---|---|---|
| `id` | ○ | ○ | **このツール専用のID**。詳細取得・削除に渡すのはこれ |
| `contactId` | ○ | ○ | メール/LINE をまたぐメイン連絡先ID。詳細取得・削除には**使えない** |
| `moshId` | ○ | ○（無い場合あり） | MOSH アカウントID。未連携は null |
| `email` | ○ | — | |
| `name` | ○ | ○ | メール側は**同一コンタクトの LINE プロフィール名**で、持たない場合は空文字。氏名ではない |
| `avatarUrl` | — | ○ | |
| `active` | ○ | ○ | false = 削除済み |
| `subscribed` | ○ | — | メール配信の購読状態 |
| `blocked` | — | ○ | LINE でブロックされているか |
| `benefit.firstBenefitTitle` | ○ | ○ | **最初に取得した特典1件だけ**。全件は詳細取得（[mcp-tools.md](mcp-tools.md)） |
| `inflowAction` | — | ○ | `{ id, name }` または null |
| `richMenu` | — | ○ | `{ id, name }` または null |
| `tags[]` | ○ | ○ | `{ id, name, ... }` |
| `createdAt` | ○ | ○ | 登録日時 |
