# 動画・音声のアップロード手順

会員サイトの動画・音声コンテンツに本体を付ける手順。**順番は入れ替えられない。**

## 前提: ファイルを送れる環境か

MCP が発行するのはアップロード先の URL まで。ファイル本体をその URL へ HTTP PUT するのはクライアント側の仕事になる。

- シェルを実行できる環境（Claude Code 等）で、ファイルが手元にある → 下の手順で最後まで進める
- シェルを実行できない環境、またはファイルが手元に無い → タイトル・概要・本文・タグ・チャプターまでを `isPublished: false` の下書きで作り、本体の紐付けと公開は管理画面で行ってもらう。作業後に確認を頼まれたら `getCreatorMembershipSiteFolders` で読み戻す

## 手順

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

## 失敗したとき

| 状態 | 対応 |
|---|---|
| PUT が 200 番台以外・途中で切れた | 手順 4 の発行からやり直す（`WAITING_UPLOAD` のまま残る） |
| PUT は成功したのに `WAITING_UPLOAD` のまま | 手順 4 からやり直す |
| `ERRORED` | 使えない。手順 4 からやり直す。1 時間を超えた場合もこうなるので、大きいファイルなら回線の速さを先に確かめる |

使わなかったアセットを MCP から消す手段は無い。コンテンツに紐づけない限り会員には見えない。

## 守ること

- **READY になる前に公開しない**（理由は SKILL.md 必須ルール B）
- **アセットは発行したサイトのコンテンツにだけ使う。** 別サイトのコンテンツにも紐づけられてしまい、エラーにならない
- **`uploadUrl`・`assetId`・`uploadId` をユーザーに見せない。** 「アップロード中です」「変換が終わりました」のように状態で伝える
- **サムネイル画像は付けられない。** `thumbnailAssetId` は `null` のまま。動画は再生画面で自動生成のサムネイルになり、一覧では画像なしになる。付けたいなら管理画面を案内する
