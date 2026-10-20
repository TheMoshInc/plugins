# MCPツール仕様（正本）

本スキルが使うツール（詳細取得・削除・検索・CSVインポート・コンタクトタグ操作）と、条件に使う ID を解決する4ツールのパラメータ・制約・エラーの正本。検索リクエスト/レスポンスの構造だけは [search-schema.md](search-schema.md) が正本。CSVインポートの手順とCSV形式は [csv-import.md](csv-import.md) が正本。

> 本ファイルのパラメータ・制約・エラーは、MOSH の API スキーマ定義で確認したもの（コンタクトタグ操作は 2026-10 時点、それ以外は 2026-09 時点）。
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

## コンタクトタグ操作

### 共通ルール

- **タグ id は `getCreatorContactTags` の id だけ。** `getCreatorCustomerTags` が返す顧客タグは別体系で、id を混用してはならない（数値が偶然一致すると、404 にならず同じ数値の別のコンタクトタグが書き換わる）。存在しない id・他クリエイターのタグは 404
- **タグはコンタクト（`contactId`）単位で付く。** LINE とメールの両方を持つコンタクトでは、両方の一覧・配信条件に反映される
- **副作用**: 新しく付いたコンタクトについては、タグ付与をトリガーにしたワークフローが起動する（配信まで進むと取り消せない）。既に付いていた人は変化せず、起動もしない。**解除ではワークフローは起動しない**。解除を取り消して付け直すと新規付与扱いになり、ワークフローが起動する
- 付け外しの返り値は `{ "message": "OK" }` のみで、**件数は返らない**

### タグの作成・名前変更

| ツール | 入力 | 挙動 |
|---|---|---|
| `postCreatorContactTags` | `{ "name": "<1〜64文字>" }` | 新規作成は 201、**同名のタグが既にあれば作らずに既存の `id` を 200 で返す（冪等）**。作った id はそのまま付与に使える |
| `patchCreatorContactTag` | パス `id` + `{ "name": "<1〜64文字>" }` | 変わるのは名前だけ。付与・ワークフロー・配信条件は id で参照しているため、**全利用箇所に新しい名前が反映される**。同じ名前のままなら何もせず 200。**別のタグと同じ名前にすると 422**（応答の `message` をそのままユーザーに伝える）。タグ自体の削除はできない |

`getCreatorContactTags` の使い方: `keyword`（タグ名の部分一致・大文字小文字を区別しない・64文字まで）で名前から絞れる。部分一致なので、返ってきた中から**名前が完全に一致するタグ**を選ぶ（完全一致が無く候補が複数あるなら、ユーザーに選んでもらう）。`limit` は最大100、続きは `offset`。`isContactCountsIncluded` は付与人数が必要なときだけ `true`（集計が重い。名前から id を引くだけなら付けない）。指定された名前のタグが無ければ、既存タグを提示して選び直してもらうか、`postCreatorContactTags` で作る。

### 名指しで付ける・外す

| ツール | 入力 |
|---|---|
| `postCreatorContactTagContacts`（付与） / `deleteCreatorContactTagContacts`（解除） | パス `id`（タグID） + `{ "contactIds": [9012, 9013] }`（1件以上） |

- **`contactIds` は検索結果の `contacts[].contactId`**（`contacts[].id` ではない。詳細取得・削除とは逆）。メール検索・LINE 検索どちらの結果の `contactId` でもよい
- `contactIds` に他クリエイター・存在しないコンタクトが**1件でも含まれると 404 で全件が反映されない**（部分反映なし）
- 付与: 既に付いている人はそのまま（何度実行しても同じ）。解除: 付いていない人はスキップされエラーにならない。コンタクト自体も他のタグも消えない
- 大人数・条件で決まる対象は、次の検索条件一括を使う（ツール定義の指定）

### 検索条件で一括して付ける・外す（ByCondition）

| 対象 | 付与 / 解除 | リクエストボディ |
|---|---|---|
| メール | `postCreatorContactTagContactsEmailByCondition` / `deleteCreatorContactTagContactsEmailByCondition` | `{ "keyword": "", "filters": { ... } }` |
| LINE | `postCreatorContactTagContactsLineByCondition` / `deleteCreatorContactTagContactsLineByCondition` | `{ "keyword": "", "lineChannelId": null, "filters": { ... } }` |

パスの `id` はタグID。

- **全キー必須**。`keyword` は絞らないなら空文字。`filters` の中身は同じ系統の検索と同じ構造・同じ意味（[search-schema.md](search-schema.md)。中のキーも全キー必須）。`limit` / `offset` / `sort` は無い
- **同じ `keyword` / `filters` で検索すると、検索結果と同じ集合が対象になる**（実行時点の集合）
- **`filters: null` は削除済みを除く全コンタクトが対象になる。** ユーザーが「全員」と明言し、その件数で承認した場合以外は使わない
- **LINE の `lineChannelId` はキー自体が必須**。全 LINE公式アカウント横断なら、検索では `lineChannelId` を省略し、このツールでは `null` を渡す。特定アカウントなら両方に同じ id を渡す

**実行前の件数確認（必須）**: 同じ条件で検索（`limit: 1`）し、`filteredCount` とタグ名を提示して承認を得る。**0件なら「該当なし」と伝えて実行しない。** `filteredCount` はメールアドレス単位／LINE友だち単位で、タグはコンタクト単位で付くため、実際に変わるコンタクト数はこれ以下になることがある（「最大 N 件」と伝えるのは可）。

付与のとき（新しく付く件数＝ワークフローの起動対象を数える）:

1. 元の条件で検索し、`filteredCount`（条件に合う件数。付与済みを含む）を得る
2. `filters.tag.id` に `{ "value": <付与するタグid>, "operator": "ne" }` を加えて検索し、その `filteredCount` を**新しく付く件数**として提示する。`filters` が `null` のときは、`tag` 以外を `null` にした `filters` を作って加える
   - 同じタグ id が既に `ne` で入っている → 加えない（手順1の件数がそのまま新規件数）
   - 同じタグ id が `eq` で入っている → 全員付与済み。**新規件数・ワークフロー起動対象は0件**と提示する（手順1の件数を新規件数として出さない）
   - `filters.tag.id` が既に5件 → 加えられない。手順1の件数を上限の目安として提示する
   - 同じタグ id の重複指定（`eq` と `ne` の同時指定を含む）は 400
3. **付与の実行は元の条件で行う**（`ne` を加えた条件ではない）

解除のとき（付いている件数を数える）:

1. **件数確認の検索と解除ツールの両方で**、`filters.tag.id` に `{ "value": <外すタグid>, "operator": "eq" }` を必ず含める。これで `filteredCount` がそのタグの付いている件数になる。`filters` が `null` のときは、`tag` 以外を `null` にした `filters` を作って加える
   - 既に `eq` で入っている → そのまま使う
   - `ne` で入っている → 対象は0件。「該当なし」と伝えて実行しない
   - `filters.tag.id` が既に5件で `eq` を加えられない → 元の条件で検索し（100件ごとに `offset` を進める）、`contacts[].tags` に外すタグを含むコンタクトの `contactId` を集めて `deleteCreatorContactTagContacts` で解除する

### タグ操作のエラー

| コード | 意味 | 対処 |
|---|---|---|
| 400 | 入力不正（名前が1〜64文字の範囲外、タグ条件の同じ id の重複・6件以上、`filters` のキー欠落 など） | 構造を確認して組み直す |
| 404 | タグが無い（顧客タグの id を渡した・存在しない・他クリエイター）。名指しでは `contactIds` に不正な値が1件でもある | 全件未反映。タグを `getCreatorContactTags` で、`contactId` を検索で取り直す |
| 422 | 名前変更で、同じ名前の別タグが既にある | 応答の `message` をそのまま伝え、別の名前を確認する |

## 条件に使う ID を解決するツール

| 解決したいもの | ツール | 備考 |
|---|---|---|
| コンタクトタグID | `getCreatorContactTags` | タグ名から `id` を引く（使い方は上の「タグの作成・名前変更」）。該当が無ければ既存タグを提示して選び直してもらう |
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
| タグ自体の削除 | `deleteCreatorContactTag` | `mcp` タグ無し（復元不可で、使用中のワークフロー・未送信の配信が参照していると拒否される仕様） |
| タグを使っているワークフロー・配信の確認 | `getCreatorContactTagsUsages` | `mcp` タグ無し |
| 1人のコンタクトから1タグを外す（旧形式） | `deleteCreatorContactTagContact` | `mcp` タグ無し。`deleteCreatorContactTagContacts` に `contactId` 1件を渡せば同じことができる |

いずれも（最後の行を除き）MOSH 管理画面での操作を案内する。コンタクトを1件ずつ新規作成するエンドポイントは、このドメインに存在しない。
