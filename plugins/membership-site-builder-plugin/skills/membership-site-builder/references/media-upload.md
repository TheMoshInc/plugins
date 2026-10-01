# 動画・音声・サムネイル画像のアップロード手順

会員サイトの動画・音声コンテンツに本体を付ける手順と、コンテンツにサムネイル画像を付ける手順。どちらも**順番は入れ替えられない。**

## 前提: ファイルを送れる環境か

MCP が発行するのはアップロード先の URL まで。ファイル本体をその URL へ HTTP PUT するのはクライアント側の仕事になる。

- シェルを実行できる環境（Claude Code 等）で、ファイルが手元にある → 下の手順で最後まで進める
- シェルを実行できない環境、またはファイルが手元に無い → タイトル・概要・本文・タグ・チャプターまでを `isPublished: false` の下書きで作り、本体の紐付けと公開は管理画面で行ってもらう。作業後に確認を頼まれたら `getCreatorMembershipSiteFolders` で読み戻す

サムネイル画像も同じで、送れない環境なら `thumbnailAssetId` を `null` のまま作り、画像は管理画面で付けてもらう。

## 動画・音声の手順

1. **聞くこと**: ファイルの置き場所、字幕を自動で作るか、AI 要約・目次（チャプター）を自動で作るか。AI 要約・目次は発行時にしか指定できない
2. **ID をそろえる**: `getCreatorMembershipSites` でサイトの `id`、`getCreatorMembershipSiteFolders` で `folderId`
3. **サイズを測る**: `stat -f%z "<ファイル>"`（macOS）/ `stat -c%s "<ファイル>"`（Linux）。上限は動画 20GB・音声 500MB
4. **発行**: `postCreatorMembershipSiteAssets` に `assetType`・`fileSizeBytes`・`isGenerateSubtitles`・`isGenerateAiSummary` を渡し、`uploadUrl`・`assetId` を受け取る
5. **PUT**: 発行から 1 時間以内に送り切る。ファイル全体を 1 回で PUT する（分割は不要）

   ```bash
   curl -sS -X PUT -T "<ファイル>" "<uploadUrl>" -o /dev/null -w '%{http_code}\n'
   ```

   200 番台なら成功。大きいファイルはコマンドの時間制限を超えることがあるので、バックグラウンドで実行して終わるのを待つ。1 時間で送り切れる回線速度が要る（目安: 20GB ならおよそ 45Mbps 以上）。間に合いそうにないときは管理画面でのアップロードを案内する
6. **READY 待ち**: `getCreatorMembershipSiteAsset` を 10 秒程度の間隔で呼び、`status` が `READY` になるまで待つ。省かない
7. **紐付け**:
   - 新しく作る → `postCreatorMembershipSiteContents` の `contentType` を `assetType` と同じにし、`assetIds: [assetId]`
   - 既存コンテンツの本体を差し替える → `patchCreatorMembershipSiteContent` の `assetIds`。配列ごと置き換わるので、先に `getCreatorMembershipSiteContent` で現在値を読む
8. **公開**: SKILL.md の「6. 公開」に従う（`isNotifyOnPublish` の確認とユーザーの承認）

## 動画・音声で失敗したとき

| 状態 | 対応 |
|---|---|
| PUT が 200 番台以外・途中で切れた | 手順 4 の発行からやり直す（`WAITING_UPLOAD` のまま残る） |
| PUT は成功したのに `WAITING_UPLOAD` のまま | 手順 4 からやり直す |
| `ERRORED` | 使えない。手順 4 からやり直す。1 時間を超えた場合もこうなるので、大きいファイルなら回線の速さを先に確かめる |

使わなかったアセットを MCP から消す手段は無い。コンテンツに紐づけない限り会員には見えない。

## 動画・音声で守ること

- **READY になる前に公開しない**（理由は SKILL.md 必須ルール B）
- **アセットは発行したサイトのコンテンツにだけ使う。** 別サイトのコンテンツにも紐づけられてしまい、エラーにならない
- **`uploadUrl`・`assetId`・`uploadId` をユーザーに見せない。** 「アップロード中です」「変換が終わりました」のように状態で伝える
- **サムネイル画像は下の「サムネイル画像」の手順で別に付ける。** 付けないなら `thumbnailAssetId` は `null` のまま（見え方は [content-schema.md](content-schema.md) の `thumbnailAssetId` の行）

## サムネイル画像

コンテンツ一覧と再生画面に出る画像。動画・音声・記事のどのコンテンツにも付けられる。形式は JPEG・PNG・WebP、100MB まで。管理画面の推奨は横長の 16:9。

1. **発行**: `postMeMediaUploadUrls` に `name`（拡張子つきのファイル名）と `type`（JPEG は `imageJpeg`、PNG は `imagePng`、WebP は `imageWebp`）を渡し、`moshMediaId`・`uploadUrl` を受け取る。**このサムネイルのために毎回新しく発行する**（別の用途で発行した画像も受け付けてしまうため、使い回さない）
2. **PUT**: 発行から 15 分以内に送る。`Content-Type` ヘッダーは付けなくてよい

   ```bash
   curl -sS -X PUT -T "<画像ファイル>" "<uploadUrl>" -o /dev/null -w '%{http_code}\n'
   ```

   200 番台なら成功
3. **登録**: `postCreatorMembershipSiteImageAssets` に会員サイトの `id` と `moshMediaId` を渡し、`assetId` を受け取る。画像の変換が終わるまでは「画像の処理が完了していません」と返るので、10 秒程度おいて**同じ `moshMediaId` で**呼び直す（PUT はやり直さない）
4. **紐付け**:
   - 新しく作る → `postCreatorMembershipSiteContents` の `thumbnailAssetId` に `assetId` を入れる
   - 既存コンテンツに付ける・差し替える → `patchCreatorMembershipSiteContent` の `thumbnailAssetId` だけを送る（他の項目は元の値のまま残る）

### サムネイル画像で失敗したとき

| 状態 | 対応 |
|---|---|
| PUT が 200 番台以外・15 分を過ぎた | 手順 1 の発行からやり直す |
| 数分たっても「画像の処理が完了していません」が続く | 変換に失敗している可能性がある。手順 1 からやり直す |
| 登録が弾かれた | 形式が JPEG・PNG・WebP 以外か、100MB を超えている。`type` に画像以外を選んで発行した場合と、`postMeMediaUploadUrls` 以外（ファイル共有の画像アップロード等）で作った画像を渡した場合も弾かれる。形式を確かめて手順 1 からやり直す |
| コンテンツへの指定が弾かれた | `thumbnailAssetId` に動画・音声の `assetId` を入れている。画像は手順 3 で返った `assetId` だけを使う |

同じ `moshMediaId` で登録し直しても、新しいアセットは作られず同じ `assetId` が返る。

### サムネイル画像で守ること

- **登録した画像は、登録したサイトのコンテンツにだけ使う。** 同じクリエイターの別サイトのコンテンツにも指定できてしまい、エラーにならない
- **`uploadUrl`・`moshMediaId`・`assetId` をユーザーに見せない。** 「サムネイル画像を設定しました」のように結果で伝える
- 付けたサムネイルを外すときは `thumbnailAssetId: null` を送る
