# MCPツール仕様（正本）

本スキルが使う7ツールと、条件に使う ID を解決する4ツールのパラメータ・制約・エラーの正本。検索リクエスト/レスポンスの構造だけは [search-schema.md](search-schema.md) が正本。CSVインポートの手順とCSV形式は [csv-import.md](csv-import.md) が正本。

> 本ファイルのパラメータ・制約・エラーは、MOSH の API スキーマ定義で確認したもの（2026-09 時点）。
> 値を書き換えるときは、記憶や推測ではなく MOSH の API スキーマ定義で裏取りする。

## 詳細取得

| ツール | 入力 | 返るもの |
|---|---|---|
| `getCreatorContactEmail` | `id`（メールコンタクトID。1以上） | メールコンタクト1件 |
| `getCreatorContactLine` | `id`（LINEコンタクトID。1以上） | LINEコンタクト1件 |

- `id` は**同じ系統の検索結果の `id`**。`contactId` を渡してはいけない
- **ID 体系は3つとも独立している**（メールコンタクトID / LINEコンタクトID / `contactId` はそれぞれ別々に採番される）。同じ数値が複数の体系に存在しうるため、**取り違えても 404 になるとは限らず、別人のデータが返る**。返ってきた相手が依頼の相手と一致するかを、削除の前に必ず突き合わせる
- **一覧との差は特典だけ**: 一覧が `benefit`（最初に取得した1件のタイトル）なのに対し、詳細は `benefits[]` で取得済みの特典すべて（`id` / `title` / `url` / `claimType`（`line` | `email`） / `claimedAt`）を返す。特典の全件が不要なら検索結果で足りる
- LINE の詳細では `richMenu` に `backgroundMoshImageId`（プレビュー用の画像ID）が加わる
- ログイン中クリエイター以外のコンタクト、存在しない ID は **404**
- レスポンスにはメールアドレス・表示名などの個人情報が含まれる。依頼された用途以外に出力・転記しない

## 削除（`patch` という名前だが実態は削除）

| ツール | 対象 |
|---|---|
| `patchCreatorContactsEmail` | メールコンタクト |
| `patchCreatorContactsLine` | LINEコンタクト |

リクエストボディ（両者共通）:

```json
{ "ids": [123, 456], "active": false }
```

- `ids`: 対象のコンタクトID配列。**1〜100件**。検索条件による一括指定はできない（条件一括の更新ツールは MCP に公開されていない）
- **渡す ID は同じ系統の検索結果から取る**。メール↔LINE↔`contactId` は独立した ID 体系で、取り違えは 404 ではなく**別人の削除**になりうる（上記「詳細取得」参照）
- `active`: **必ず `false`**。管理画面の「削除」と同じ操作
- `active: true`（非表示の解除）は API トークン認証では **403**。管理画面にも復元の導線がないため、MCP からの復元は不可能
- ツール定義自体が「実行前に検索で対象を取得してユーザーに提示し、同意を得てから実行すること」を要求している

削除で起きること:

| | メール | LINE |
|---|---|---|
| データ | メールアドレスは保持される | **表示名とプロフィール画像は削除される**（LINE のデータポリシーに基づくマスク）。解除しても元の表示名は戻らない |
| 検索 | 既定の結果から外れる | 既定の結果から外れる |
| 配信 | 配信対象にならなくなる | 配信対象にならなくなる |

エラー:

| コード | 意味 | 対処 |
|---|---|---|
| 400 | 指定した ID がすべて既に非表示 | 再実行しない。既に削除済みと伝える |
| 403 | `active: true` を指定した | 解除は不可。実行しない |
| 404 | 指定した ID のコンタクトが1件も見つからない | ID の取り違え（メール↔LINE、`id`↔`contactId`）を疑い、検索し直す |

返り値は成功メッセージのみで、**削除された件数は返らない**。報告する件数は渡した ID の件数を使う。

## 検索

| ツール | 対象 |
|---|---|
| `postCreatorContactsEmailSearch` | メールコンタクト |
| `postCreatorContactsLineSearch` | LINEコンタクト（LINE公式アカウントの友だち） |

- リクエスト・レスポンスの構造は [search-schema.md](search-schema.md)
- 不正なリクエストは **400**（`filters` のキー欠落、`limit` が 1〜100 の範囲外 など）
- 該当0件はエラーではない

## CSVインポート

| ツール | 対象 |
|---|---|
| `postCreatorContactsEmailCsvImportJobs` | メールアドレスの一括登録（LINE友だちには相当ツールが無い） |

- 入力は `{ "name": "<CSVファイル名>" }`、返りは `{ id, uploadUrl }`（201）
- `uploadUrl` は S3 の Presigned PUT URL。**有効期限15分**、`Content-Type: text/csv` で本体を PUT する必要があり、この API は PUT を代行しない
- 手順・CSV形式・取り込み時の挙動は [csv-import.md](csv-import.md)
- **進捗・結果を取得するツールは MCP に公開されていない**（`getCreatorContactsEmailCsvImportJobsLatest` は存在するが `mcp` タグが無い）

## 条件に使う ID を解決するツール

| 解決したいもの | ツール | 備考 |
|---|---|---|
| コンタクトタグID | `getCreatorContactTags` | タグ名から `id` を引く。該当が無ければ既存タグを提示して選び直してもらう |
| 流入経路ID | `getCreatorScenariosInflowActions` | `creatorLineChannelId` を渡すとそのアカウント分、未指定で全件 |
| リッチメニューID | `getCreatorLineRichMenus` | 最終更新日時の降順 |
| LINE公式アカウントID | `getCreatorLineChannels` | 作成日時の降順（最新が先頭） |

これらの ID を推測・創作して検索条件に入れない。名前が一致しないときはユーザーに選び直してもらう。

## MCP に公開されていない隣接ツール（代用しない）

| 操作 | 該当エンドポイント | 状況 |
|---|---|---|
| CSVインポートの最新ジョブ状況 | `getCreatorContactsEmailCsvImportJobsLatest` | `mcp` タグ無し |
| 条件一括でのコンタクト更新・削除 | `patchCreatorContactsEmailByCondition` / `patchCreatorContactsLineByCondition` | `mcp` タグ無し |
| LINE友だちリストの同期・一括取り込み | `postCreatorContactsLineFriendImportJobs` / `getCreatorContactsLineFriendImportJobsLatest` | `mcp` タグ無し |

いずれも MOSH 管理画面での操作を案内する。コンタクトを1件ずつ新規作成するエンドポイント、および個別コンタクトにタグを付け外しするエンドポイントは、このドメインに存在しない。
