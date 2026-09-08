---
name: membership-site-builder
description: MOSH の会員サイトを MCP 経由で作成・編集・公開するスキル。対象サイトの特定 → 現状の把握 → フォルダとコンテンツの組み立て → 読み戻してレビュー → ユーザー承認のうえ公開、という段階で対話的に進める。「会員サイトを作って」「レッスンを追加して」「フォルダを作って」「会員サイトの中身を見せて」「コンテンツを公開して」「membership-site-builder」など、MOSH の会員サイトの作成・編集・整理リクエストで使用する。「会員サイトに何が入っているか教えて」のような単純な参照でも `*MembershipSite*` 系ツールを使う場合は本スキルを起動し、提示ルールを適用する。
---

# MOSH 会員サイトビルダー

## Overview

MCP ツール（`*MembershipSite*` 系。以下ツール名は OpenAPI の operationId で記す。実際のツール名にはサーバー識別子の接頭辞（例: `mcp__<server>__getCreatorMembershipSites`）が付くが、その形は環境により異なるため本書では付けない）を使って、MOSH クリエイターの会員サイトを作成・編集・公開する。

会員サイトは 3 階層でできている。

```
会員サイト（サイト全体の名前・公開状態・テーマカラー）
└── フォルダ（コンテンツの入れ物。表示形式と公開状態を持つ）
    └── コンテンツ（動画・記事・音声の 1 本ずつ。本文・公開設定を持つ）

タグ … サイトに登録し、コンテンツに付ける目印（階層の外）
```

コンテンツは必ずどれかのフォルダに属するため、**サイト → フォルダ → コンテンツ**の順に作る。

書き込みの前に [references/content-schema.md](references/content-schema.md) を必ず参照する。コンテンツ作成は全項目が必須で、使わない項目にも決まった値を渡す必要がある。

**動画・音声・画像を MCP からアップロードすることはできない。** そのため動画コンテンツ・音声コンテンツは非公開の下書きまでしか作れず、本体の紐付けと公開は管理画面で行ってもらう（記事コンテンツは公開まで作れる）。サムネイル画像・ヘッダーロゴ・ホーム画面アイコンも同じで、新しく付けることはできない（外すことはできる）。

## 前提条件: 参照リソース

| 参照 ID | MCP で取得できるか | 取得経路 |
|---|---|---|
| 会員サイトの `id` | ✅ | `getCreatorMembershipSites` |
| フォルダの `folderId` | ✅ | `getCreatorMembershipSiteFolders` |
| コンテンツの `contentId` | ✅ | `getCreatorMembershipSiteFolders`（各フォルダの `contents` に入っている） |
| タグの `tagId` | ✅ | `getCreatorMembershipSiteTags` |
| 動画・音声・ファイルの `assetIds` | ❌ | MCP にアップロードするツールも一覧するツールも無い。既存コンテンツを `getCreatorMembershipSiteContent` で読んだときの値だけが分かる |
| サムネイルの `thumbnailAssetId` | ❌ | 同上。新規作成では必ず `null` を渡す |
| `headerLogoImageId` / `homeIconImageId` | ❌ | MCP に画像をアップロードするツールが無い。外すときだけ `null` を明示する |

コンテンツだけを一覧するツールは無い。サイトの中身を見るときは常に `getCreatorMembershipSiteFolders` を使う。

## When to use

- 会員サイトを新しく作りたいとき、サイトの設定（名前・公開状態・テーマカラー等）を変えたいとき
- フォルダを作る・名前や表示形式を変える・公開状態を切り替える・削除するとき
- コンテンツ（動画・記事・音声）を作る・書き換える・別のフォルダへ移す・複製する・削除するとき
- コンテンツにタグを付けたい、タグを作る・名前を変える・削除するとき
- サイトの中に何がどれだけ入っているかを確認したいとき
- `*MembershipSite*` 系ツールが必要な文脈

## When NOT to use

- 動画・音声・画像のアップロードと、それを使う公開作業 → MCP にツールが無いため管理画面を案内する
- コンテンツ・フォルダ・タグの並び替え → MCP にツールが無いため管理画面を案内する
- 会員（ゲスト）の招待・閲覧権限の付与・ブロック、会員数や視聴状況の確認 → MCP にツールが無い
- 会員サイトを売る商品・プランの作成や価格の設定 → `product-navigator` スキルで参照し、変更は管理画面
- 会員向けのお知らせ配信・一斉配信 → `contact-broadcast` スキル
- ランディングページ → `lp-builder` スキル、ステップ配信 → `workflow-builder` スキル

## Workflow

このスキルは **下書き → ユーザー確認 → 公開**を基本とする。公開・削除・通知の前には必ず承認を取る。

### 1. 対象サイトの特定と現状の把握

1. `getCreatorMembershipSites` で一覧を取得し、ユーザーが言ったサイト名から対象を特定する。名前が一致しないときは候補を挙げて選んでもらう（勝手に決めない）
2. `totalCount` が取得件数より多ければ `offset` をずらして続きを取る（`limit` 既定 20）
3. サイトの中身を見るときは `getCreatorMembershipSiteFolders`。フォルダと、その中のコンテンツの概要（タイトル・種類・公開状態・タグ）が 1 回で全件返る
4. サイトの設定値（テーマカラー等）を見るときは `getCreatorMembershipSite`、タグを見るときは `getCreatorMembershipSiteTags`

### 2. 要件ヒアリング

作るものによって聞くことが変わる。自明なら聞き直さない。

- **サイトを作る**: サイト名。作成直後は非公開・設定は初期値になる
- **フォルダを作る**: フォルダ名、表示形式（カルーセル / リスト / タイル）、いま公開するか
- **コンテンツを作る**: どのフォルダに入れるか、種類（動画 / 記事 / 音声）、タイトル、本文、概要、タグ、公開のタイミング
- **種類は後から変えられない**ため、作る前に必ず確定させる
- 動画・音声を選んだときは、**MCP からは本体を上げられないので非公開の下書きまでしか作れない**ことを先に伝える

### 3. 作成

サイト → フォルダ → コンテンツの順に作り、返ってきた `id` を次の呼び出しに渡す。

1. `postCreatorMembershipSites`（サイト名だけ。作成直後は非公開）
2. `postCreatorMembershipSiteFolders`（フォルダ名・表示形式・公開状態）
3. タグを使うなら先に `getCreatorMembershipSiteTags` で既存を確認し、無いものだけ `postCreatorMembershipSiteTags` で作る
4. `postCreatorMembershipSiteContents`（[references/content-schema.md](references/content-schema.md) の全項目を埋める。記事の最小例は [examples/article-content-draft.json](examples/article-content-draft.json)。例の `folderId` はダミー値なので、取得した値に置き換えてから送る）

似たコンテンツを増やすときは、一から組み立てるより `postCreatorMembershipSiteContentDuplicate` が早い。複製したものはタイトルが「〈元のタイトル〉のコピー」になり、必ず非公開・コメント無効で作られる。

**同じサイトへ続けて作るときは、前の呼び出しの結果を受け取ってから次を呼ぶ。** 並行して呼ぶと並び順の採番が衝突して失敗する。

### 4. 編集

更新ツールは 2 種類あり、送り方が違う。取り違えると設定が消える。

| ツール | 送り方 |
|---|---|
| `patchCreatorMembershipSite`（サイト設定） | **全置換**。7 項目すべて必須。先に `getCreatorMembershipSite` で現在値を読み、変えない項目はその値をそのまま送り返す |
| `patchCreatorMembershipSiteFolder` / `patchCreatorMembershipSiteContent` / `patchCreatorMembershipSiteTag` | **送った項目だけ**変わる。ただし 1 項目も送らないリクエストは受け付けない |

コンテンツの `chapters` / `assetIds` / `tagIds` は、部分更新のツールでも**配列ごと置き換わる**。1 つ足すだけでも `getCreatorMembershipSiteContent` で現在値を取り、足したものを含む配列全体を送る。空配列を送るとすべて外れる。

本文（`body`）も同じで、取得した本文を土台にせず書き直すと、保存済みの画像・埋め込み・書式が失われる。

### 5. レビュー

1. 書き込んだ対象を読み戻す（サイト設定は `getCreatorMembershipSite`、フォルダとコンテンツの並びは `getCreatorMembershipSiteFolders`、本文まで見るなら `getCreatorMembershipSiteContent`）
2. [references/best-practices.md](references/best-practices.md) 末尾の「セルフレビューチェックリスト」を実行する
3. JSON を貼らず、自然言語の箇条書きで要約して提示する

### 6. 公開

公開はサイト・フォルダ・コンテンツのそれぞれが持つ `isPublished` を `true` にして行う（専用のツールは無い）。会員から見えるのは**サイト・フォルダ・コンテンツの 3 つがすべて公開のとき**だけ。

**公開の前に必ず確認する。**

- コンテンツを公開するときは、先に `getCreatorMembershipSiteContent` で `isNotifyOnPublish` を読む。`true` のまま公開すると会員全員へ通知が届き、**取り消せない**。通知が要らないなら同じリクエストで `isNotifyOnPublish: false` も送る
- 動画・音声は本体が紐づいていないと公開できない。MCP からは紐付けられないため、公開は管理画面で行ってもらう
- サイトを非公開から公開に変えるときだけ、公開できるサイト数の上限を判定する

### 7. 削除

削除ツールはすべて**復元できない**。依頼されたときだけ使い、次の順で進める。

1. 対象を読み戻し、**名前で**提示する（サイトなら `getCreatorMembershipSite` でサイト名 ＋ `getCreatorMembershipSiteFolders` でフォルダ数とコンテンツ数、フォルダなら中のコンテンツ件数と公開中の件数、コンテンツならタイトルと公開状態、タグなら付いているコンテンツの件数）
2. 何が一緒に消えるかを伝える（[references/content-schema.md](references/content-schema.md) の「削除で消えるもの」）
3. 「削除しますか？」と確認を取る
4. 実行し、一覧を読み戻して消えたことを確認する

見えなくしたいだけなら、削除ではなく `isPublished` を `false` にする方法があることを先に案内する。

## ユーザーへの提示・コミュニケーション規約

このスキルを使うのは MCP / JSON / HTTP の知識を持たないクリエイターである前提で会話する。

1. **JSON をそのまま見せない。** 自然言語の箇条書きか表に整形して提示する。ユーザーが明示的に求めたときだけ JSON を出す。ただし本文（Tiptap JSON）と生の内部 ID は、求められても出さない
2. **HTTP 用語・ステータスコードを文面に出さない。** 「保存できました」「動画が紐づいていないため公開できませんでした」のように、起きたことと理由を平易な日本語で伝える
3. **生の内部 ID を出さない。** 会員サイト・フォルダ・コンテンツ・タグはすべて名前（タイトル）で示す。`slug` も内部の値なので出さない
4. **英字の設定値は日本語に変換して出す**（変換表は [references/content-schema.md](references/content-schema.md)）。`VIDEO` や `CAROUSEL` をそのまま見せない
5. **本文はそのまま貼らない。** `body` は現在のエディタが保存した Tiptap JSON 文字列で返り、旧サイトから移行したものは HTML 断片、初期データはプレーンテキストのこともある。テキストを取り出して要約する
6. 参照 ID が未確定でも、推測値・ダミー値・0 を入れない。取得ツールで確かめるか、ユーザーに聞く

## 必須ルール

### A. コンテンツ作成は全項目必須

`postCreatorMembershipSiteContents` は 16 項目すべてが必須で、部分的に送ることはできない。使わない項目に渡す値は [references/content-schema.md](references/content-schema.md) の表で確定している（例: 本文と概要は空文字、配列は空配列、日時は `null`、`thumbnailAssetId` は `null`）。

### B. 動画・音声は非公開でしか作れない

`contentType` が `VIDEO`・`AUDIO` のコンテンツは、本体の動画・音声が紐づいていないと公開できない（`isPublished: true` にすると弾かれる）。MCP からは上げられないため、`isPublished: false` で作り、本体の紐付けと公開は管理画面で行うようユーザーに伝える。記事（`ARTICLE`）はそのまま公開まで作れる。

公開中の動画・音声コンテンツから本体を外すこともできない。外すなら同じリクエストで `isPublished: false` も送る。

### C. 取り消せない操作は先に伝えて承認を取る

| 操作 | 何が起きるか |
|---|---|
| `isNotifyOnPublish: true` での公開 | 会員全員へ通知が届き、取り消せない。公開中のコンテンツに `isNotifyOnPublish: true` だけを送った場合も、次の配信で同じ通知が届く |
| `isFixedViewingOrder` を `true` から `false` へ | 全ゲストの閲覧完了状態が消える（どこまで見たかの再生位置は残る）。元に戻せない |
| サイト・フォルダ・コンテンツ・タグの削除 | 復元できない。消える範囲は [references/content-schema.md](references/content-schema.md) の表 |

`isFixedViewingOrder` は**サイト設定の全置換に巻き込まれやすい**。ユーザーから閲覧順の変更を頼まれていないときは、`getCreatorMembershipSite` で直前に読んだ値をそのまま送る。

### D. 連続した書き込みは 1 件ずつ

フォルダ作成・コンテンツ作成・コンテンツ複製・コンテンツのフォルダ移動は、前の呼び出しの結果を受け取ってから次を呼ぶ。並行して呼ぶと並び順の採番が衝突して失敗する。

### E. 並び順は変えられない

作ったフォルダ・コンテンツ・タグは、それぞれ末尾に追加される。MCP に並び替えのツールは無いため、順番を指定したいときは**作る順番で決める**か、管理画面での並び替えを案内する。

### F. 日時はタイムゾーンオフセット付きで送る

`scheduledPublishAt` / `scheduledUnpublishAt` は `2026-09-01T10:00:00+09:00` のようにオフセットを付ける。省略するとエラーになる。`Z` は UTC として解釈されるため、日本時間のつもりで `Z` を付けると 9 時間ずれる。

`scheduledPublishAt` と `visibleAfterPurchaseDays`（購入後◯日で公開）は同時に設定できない。片方を設定するときは、もう片方が既に入っていれば同じリクエストで `null` を送って外す。

## よくあるミス

上の「前提条件」「Workflow」「提示規約」「必須ルール A〜F」でカバー済みの事項は再掲しない。

| NG | OK |
|---|---|
| コンテンツを一覧するツールを探す | 存在しない。`getCreatorMembershipSiteFolders` がフォルダごとにコンテンツの概要を返す |
| サイト設定を 1 項目だけ送って更新する | 7 項目すべて必須。先に読んで、変えない項目もそのまま送り返す |
| タグを作る前に既存を確認しない | タグ名はサイト内で一意。同じ名前があると弾かれる。先に一覧を引き、あればその id を使い回す |
| コンテンツにタグを 1 つ足すつもりで `tagIds` に 1 件だけ送る | 配列ごと置き換わる。現在の値を読み、足したものを含む全体を送る |
| フォルダを公開にすればコンテンツも見えると考える | サイト・フォルダ・コンテンツの 3 つがすべて公開のときだけ会員から見える |
| 本文に Markdown の記法を書く | Markdown として解釈されない。改行は段落になるが、見出しや太字の記法は文字のまま会員に表示される |
| 空のフォルダだと思って削除する | 中身の有無は確認されずそのまま消える。先に中のコンテンツ件数を数えて提示する |

## エラーが返ったとき

再試行せず、返ってきた理由を平易な日本語でユーザーに伝える。よくあるものは次のとおり。

| 返ってくる状況 | ユーザーへ伝えること |
|---|---|
| サイトの作成上限・公開上限（`QUOTA_ERROR`） | 上限に達したこと。不要なサイトを削除するか非公開に戻すか、契約の見直しを案内する |
| 商品のプランで提供中のサイトを削除しようとした | 商品側の紐付けを外してからでないと削除できないこと |
| 閲覧順の固定を ON にできない | 有効な閲覧権限を持つ会員が既にいるか、一部のフォルダだけを見せる権限があること。詳しい理由は [references/content-schema.md](references/content-schema.md) の対応表 |
| 同じ名前のタグが既にある | 既存のタグ名を示し、そのタグを使うか別の名前にするかを確認する |
| 動画・音声の本体が無いまま公開しようとした | 本体を紐づけないと公開できないこと。管理画面での作業を案内する |
| 見つからない | 対象の特定からやり直す。他のクリエイターのサイトや存在しない対象を指定した場合も同じ応答になる |
| 認証エラー | MOSH との接続（API トークンの設定）を見直すよう案内する |

## References

- [references/mcp-tools.md](references/mcp-tools.md) — 使用する MCP ツール 18 件のパラメータ詳細
- [references/content-schema.md](references/content-schema.md) — コンテンツ作成・更新の全項目仕様、日本語変換表、削除で消えるもの、絶対に避けること
- [references/best-practices.md](references/best-practices.md) — サイト構成の組み立て方とセルフレビューチェックリスト
- [examples/article-content-draft.json](examples/article-content-draft.json) — 記事コンテンツを作るときの最小の送信内容。`folderId` はダミー値なので、`getCreatorMembershipSiteFolders` で取得した値に置き換えてから送る
