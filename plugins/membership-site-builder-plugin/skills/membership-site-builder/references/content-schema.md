# 会員サイトのデータ構造リファレンス

コンテンツ作成・更新に渡す値の仕様と、レスポンスの読み方。

## 階層と公開の関係

```
会員サイト（isPublished）
└── フォルダ（isPublished）
    └── コンテンツ（isPublished ＋ 予約公開 ＋ 購入後◯日で公開）
```

会員から見えるのは、**サイト・フォルダ・コンテンツの 3 つがすべて公開のとき**だけ。フォルダを非公開にすると中のコンテンツは見えなくなるが、コンテンツ自体の公開状態は変わらない。

会員がサイトの中身を見られるかどうかは、これとは別に会員ごとの閲覧権限で決まる。会員ごとの閲覧権限を扱うツールは無い。プランごとの閲覧できるフォルダと期間は `product-builder` スキルのライセンス設定で扱う。

## コンテンツ作成の全項目（`postCreatorMembershipSiteContents`）

**必須 17 項目 ＋ 省略できる 2 項目（`visibleAfterPurchaseHours` / `visibleAfterPurchaseMinutes`）。**使わない項目にも下の「使わないときの値」を渡す。

| 項目 | 型 | 制約 | 使わないときの値 |
|---|---|---|---|
| `folderId` | 整数 | 1 以上。同じサイトのフォルダ | （必ず指定） |
| `contentType` | 文字列 | `video` / `article` / `audio`。**作成後に変更できない** | （必ず指定） |
| `title` | 文字列 | 1〜200 文字 | （必ず指定） |
| `body` | 文字列 | 表示テキスト 30,000 文字以内（送信は最大 300,000 文字） | `""` |
| `description` | 文字列 | 10,000 文字以内 | `""` |
| `chapters` | 配列 | `{ startTime, title }` の配列。他のキーは受け付けない | `[]` |
| `thumbnailAssetId` | 整数 または `null` | サムネイル画像の ID（[media-upload.md](media-upload.md) の「サムネイル画像」）。動画・音声の `assetId` は入れられない。`null` のとき、変換済み（READY）の動画が紐づいていれば動画から作られた画像が一覧と再生画面に出る。それ以外（変換中の動画・音声・記事）は一覧で画像なしの表示になり、音声は再生画面にも画像が付かない | `null` |
| `assetIds` | 配列 | 動画・音声の ID（[media-upload.md](media-upload.md)）。種類は `contentType` と揃える | `[]` |
| `isPublished` | 真偽値 | 動画・音声は本体が無いと `true` にできない（公開条件は SKILL.md 必須ルール B） | `false` |
| `isNotifyOnPublish` | 真偽値 | `isPublished` と同時に `true` にすると購入者（会員）へメールで通知が届き、**取り消せない**。LINE 通知は送られない | `false` |
| `isCommentEnabled` | 真偽値 | コメントの受け付け | `false` |
| `isCompletionButtonVisible` | 真偽値 | 閲覧完了ボタンの表示。`false` だと会員側に出ない | `true`（管理画面で作ったときと同じ既定値） |
| `scheduledPublishAt` | 日時文字列 または `null` | オフセット必須。`visibleAfterPurchaseDays` と同時に指定できない。`isPublished: true` と組み合わせたときだけ効く（SKILL.md 必須ルール F） | `null` |
| `scheduledUnpublishAt` | 日時文字列 または `null` | オフセット必須。`unpublishAfterPurchaseDays` と同時に指定できない | `null` |
| `visibleAfterPurchaseDays` | 整数 または `null` | 0〜366。`scheduledPublishAt` と同時に指定できない | `null` |
| `visibleAfterPurchaseHours` / `visibleAfterPurchaseMinutes` | 整数（省略可） | 「購入から◯日後」の公開時刻（日本時間）の時 0〜23 / 分 0〜59。`visibleAfterPurchaseDays` が `null` なら 0 で保存される。更新で省略すると元の値のまま | 省略（0 で保存） |
| `unpublishAfterPurchaseDays` | 整数 または `null` | **常に `null`。** 管理画面に設定欄が無い（SKILL.md 必須ルール G） | `null` |
| `tagIds` | 配列 | 最大 20 件。`getCreatorMembershipSiteTags` の id | `[]` |

### チャプター

```json
{ "startTime": 0, "title": "イントロダクション" }
```

- `startTime`: 0 以上の整数（秒）
- `title`: 200 文字以内
- ほかのキーは受け付けない

### 日時の書き方

`2026-09-01T10:00:00+09:00` のようにタイムゾーンオフセットを付ける。省略するとエラーになる。`Z` は UTC として解釈されるため、日本時間で指定するときは `+09:00` を使う。

### 本文（`body`）

- **Markdown として解釈されない。** プレーンテキストなら改行が段落になるだけ。見出し・太字・箇条書き・リンク・Vimeo 埋め込みは Tiptap JSON で書く（[rich-body.md](rich-body.md)）
- 読み取り時は現在のエディタが保存した Tiptap JSON 文字列で返る。旧サイトから移行したコンテンツは HTML 断片、初期データはプレーンテキストのこともある
- 更新するときの手順は [rich-body.md](rich-body.md) の「書くときに守ること」が正（取得した本文を土台にする）

## 英字の値と日本語表記

ユーザーに提示するときは必ず日本語にする。

| 項目 | 値 | 日本語 |
|---|---|---|
| `contentType` | `video` | 動画 |
| `contentType` | `article` | 記事 |
| `contentType` | `audio` | 音声 |
| `displayType` | `carousel` | カルーセル（横スクロールで並べる。最大 15 件表示） |
| `displayType` | `folderView` | リスト（縦のリストで並べる） |
| `displayType` | `tile` | タイル（タイル状に並べる。スマートフォンは 1 列で最大 10 件、PC は 4 列で最大 8 件） |
| `themeColor` | `#FA6F78` ほか 7 色 | ピンク / レッド / オレンジ / ブラウン / グリーン / ブルー / パープル / ブラック（対応は [mcp-tools.md](mcp-tools.md) の表） |

真偽値の項目も、そのまま「true」と書かずに状態で伝える。

| 項目 | 伝え方 |
|---|---|
| `isPublished` | 公開 / 非公開 |
| `isFixedViewingOrder` | 設定順に見てもらう設定が ON / OFF |
| `isCommentEnabled` | コメントを受け付ける / 受け付けない |
| `isCompletionButtonVisible` | 閲覧完了ボタンを出す / 出さない |
| `isNotifyOnPublish` | 公開時に購入者へメールで通知する / しない（LINE では届かない） |
| `visibleAfterPurchaseDays` | 「購入から◯日後に公開」。`null` なら設定なし。時刻（`visibleAfterPurchaseHours` / `Minutes`）が 0:00 以外なら「購入から◯日後の◯時◯分に公開」 |
| `isContentThumbnailVisible` | コンテンツ一覧にサムネイルを出す / 出さない（タイトルだけを縦に並べる） |

## 表示名への変換

提示に生の内部 ID を出さない。名前を引く経路は次のとおり。

| 内部の値 | 何で引くか |
|---|---|
| 会員サイトの `id` | `getCreatorMembershipSites` の `name` |
| フォルダの `folderId` | `getCreatorMembershipSiteFolders` の `name` |
| コンテンツの `contentId` | `getCreatorMembershipSiteFolders` の `contents[].title` |
| タグの `tagId` | `getCreatorMembershipSiteTags` の `name`（コンテンツ概要の `tagNames` は `tagIds` と同順） |
| `assetIds` / `thumbnailAssetId` | 名前を引く経路が無い。「動画 1 本が紐づいています」のように件数と種類で伝える |
| `slug` | 外部公開用の値。ユーザーに出さない |

## 削除で消えるもの

すべて復元できない。

| 削除する対象 | 一緒に消えるもの | 残るもの |
|---|---|---|
| 会員サイト | 配下のフォルダ・コンテンツ・タグ・ヘッダーリンク、会員の閲覧権限・在籍情報・未承認の招待・お知らせ、各コンテンツへのコメント・視聴進捗・ブックマーク | アップロード済みの動画・画像 |
| フォルダ | 中のコンテンツ、そのコンテンツへの視聴進捗・ブックマーク・コメント、予約していた公開通知 | アセット（動画・音声・画像・ファイル）そのもの |
| コンテンツ | 視聴進捗・ブックマーク・コメント、予約していた公開通知 | タグそのもの、アップロード済みの動画・画像 |
| タグ | コンテンツからの目印 | コンテンツ自体 |

会員サイトを削除する前は `getCreatorMembershipSiteDashboardMembers` で会員数を取り、サイト名・フォルダ数・コンテンツ数と一緒に提示する。ここで数えるのは有効な閲覧権限が残る会員だけで、期限切れ・停止中・ブロック済みのゲストのデータも一緒に消えるため、**会員数は消えるゲストの下限**になる。0 人と返っても消えるゲストがいることを添える。

フォルダを削除すると、そのフォルダを閲覧対象にしていた会員の権限からも外れる。そのフォルダだけを対象にしていた権限では対象が 0 件になり、購入済みの会員はサイトには入れるがコンテンツが 1 件も見えなくなる。作り直しても権限は戻らない。

会員サイトは、商品のプランで提供中だと削除できない。見えなくしたいだけなら `isPublished` を `false` にする（データは残り、公開できるサイト数の枠も空く。ただし作成できるサイト数の枠は空かない。商品のプランで提供中のサイトを非公開にすると、購入済みの会員も見られなくなる）。

## 閲覧順の固定を ON にできないとき

`isFixedViewingOrder` を `false` から `true` にできない理由は 2 つ。

| 理由コード | 意味 | ユーザーへの伝え方 |
|---|---|---|
| `BLOCKED_BY_ACTIVE_GUESTS` | 有効な閲覧権限を持つ会員が既にいる（期限切れ・停止中・ブロック済みは数えない） | 会員がいる状態では設定順の閲覧に切り替えられないこと |
| `BLOCKED_BY_FOLDER_SPLIT_LICENSE` | 一部のフォルダだけを見せる閲覧権限がある。今後追加されるフォルダを自動で閲覧可にしない設定の権限があり、現時点で全フォルダを見せている場合も含む | 見せるフォルダを分けている権限があるため切り替えられないこと |

## 絶対に避けること

守らないと、取り消せない通知が飛ぶ・会員のデータが消える・壊れた状態で保存される。

- **`isNotifyOnPublish` を確認せずにコンテンツを公開しない。** 公開前に `getCreatorMembershipSiteContent` で読み、`true` なら購入者へメール通知が届くこと（LINE では届かない）を伝えて、不要なら同じリクエストで `false` を送る。公開中のコンテンツに `isNotifyOnPublish: true` だけを送るのも同じ結果になる（購入後◯日で公開の設定があるコンテンツでは、日数を満たした会員から順に届く）
- **`isFixedViewingOrder` を、ユーザーに頼まれていないのに `false` で送らない。** サイト設定は全置換なので、直前に読んだ値をそのまま送る。`true` → `false` は全会員の閲覧完了状態を消し、元に戻せない
- **サイト設定の全置換で、`themeColor` を確かめずに送り返さない。** 指定できるのは 8 色だけで、読んだ値がその中に無いときはそのまま送ると弾かれ、他の項目の更新も一緒に失敗する。どの色にするかをユーザーに確認してから送る
- **`chapters` / `assetIds` / `tagIds` を、現在値を読まずに送らない。** 配列ごと置き換わる。空配列を送るとすべて外れる
- **`body` を、取得した本文を土台にせず書き直さない。** 保存済みの画像・埋め込み・書式が失われる（手順は [rich-body.md](rich-body.md)）
- **削除する前に、消える範囲を伝えて承認を取らずに実行しない。** 上の「削除で消えるもの」を提示する
- **参照 ID を推測して入れない。** `folderId` / `tagIds` / `assetIds` / `thumbnailAssetId` は取得ツールや登録の結果から使う。0 やダミー値を入れない
- **連続した作成・複製・フォルダ移動を並行して呼ばない。** 並び順の採番が衝突して失敗する
