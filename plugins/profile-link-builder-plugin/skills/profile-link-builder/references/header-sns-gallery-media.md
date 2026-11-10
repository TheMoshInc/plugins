# ヘッダーSNS・ギャラリー・外部URL画像取り込み（ツール仕様）

## ヘッダーSNSアイコン（ヘッダーに並ぶSNSアイコン）

対応SNS（`snsType`）: `line` / `instagram` / `threads` / `facebook` / `x` / `tiktok` / `youtube` / `note`。

**URL のサーバー検証**: `post` / `patch` / `put` のいずれも http / https 以外を 422 で拒否する（実機確認: `ftp://` を post・patch で拒否。メッセージは「http または https の URL を入力してください」。`put` は前後の空白を除去）。`javascript:` を含む送信は、サーバーに届く前に403（HTML のエラー画面）で遮断される（実機確認）。書き込む前に `http://` / `https://` で始まるかを確認しておくと、無駄な失敗を避けられる。

| ツール | 用途 | 注意 |
|---|---|---|
| `getProfileLinkHeaderSnsLinks` | 一覧（`snsLinks[]` と `totalCount`。各 `id`/`snsType`/`url`/`orderNum`） | limit 既定20・最大100。取り切る（SKILL.md 必須ルールB） |
| `getProfileLinkHeaderSnsLink` | 1件取得（`snsLinkId`） | |
| `postProfileLinkHeaderSnsLink` | **1件追加**（`snsType`・`url`） | ID はサーバーが採番。末尾に追加される |
| `patchProfileLinkHeaderSnsLink` | 1件更新（`snsType`・`url` 両方必須） | 変えない方は取得値で埋める |
| `deleteProfileLinkHeaderSnsLink` | 1件削除 | 復元不可 |
| `putProfileLinkHeaderSnsLinks` | **全件を最終形で置換** | 下記 |

### `putProfileLinkHeaderSnsLinks`（全置換）

ボディ: `links: Array<{ id: string | null; snsType; url }>`

- `links` は**保存後の完全な状態**。**含めなかった既存リンクは削除される。空配列なら全削除**
- 配列の順番がそのまま表示順（`orderNum`）になる
- 既存は `id` を指定、新規は `id: null`。**新規行は呼び出しごとに新しい ID が採番される。** 同じ本文を再送しても、前回作った行は本文に無いので削除されて置き換わるだけで、重複は増えない（実機確認）。重複するのは、前回作った ID も本文に含めたまま新規行を足したとき
- `id: null` かつ `url` が空文字の行は黙って捨てられる（エラーにならない）
- 取得結果にない ID（未知）や、同じ ID の重複は 422 で拒否される。既存行の `url` を空にするのも 422
- `postProfileLinkHeaderSnsLink` も毎回新しい ID を採番する。成功したのに応答を受け取れなかったときに同じ内容を再送すると重複する（実機確認: 同じ `note` を2回 post して2件になった）。先に一覧で確認する

使い分け: 1件の追加・削除・変更は post / delete / patch。**並べ替えや複数件の一括変更のときだけ put**（取得した全件の `id` を引き継ぎ、最終形を組む）。

## ギャラリー（メディアタブの画像・動画）

### `getProfileLinkGalleryItems`
`galleryItems[]` と `totalCount`。各項目: `id` / `mediaType`（`image` | `video`）/ `mediaId` / `mediaUrl` / `thumbnailUrl`（URL 解決に失敗すると空文字）/ `orderNum`。limit 既定20・最大100。取り切る。

### `putProfileLinkGalleryItems`（全置換）

ボディ: `galleryItems: Array<{ id: string | null; mediaType; mediaId; displayOrder }>`

- 配列は**保存後の最終状態**。並べ替え・追加・削除をすべてこの1本で表す。**含めなかった項目は削除される**
- 残す項目は、取得した `id` / `mediaType` / `mediaId` をそのまま渡す（`mediaUrl` / `thumbnailUrl` は渡さない。必須ルールE）。`displayOrder` は0始まり
- 新規追加は `id: null` ＋ アップロード済みの `mediaId`。画像は下の取り込みで得た `mediaId` が使える。**動画の `mediaId` を新規に用意することは、このスキルでは扱わない**（既存の動画項目を残す・並べ替える・削除するのは可）
- **存在しない `mediaId` もエラーにならず保存され、`mediaUrl` / `thumbnailUrl` が空文字になる**（実機確認。無言の失敗）。アップロードで得た ID だけを使い、保存後の `mediaUrl` が空でないか読み戻す
- `mediaId` が空の行は、エラーにならず黙ってスキップされる（実機確認）。空配列を送ると全削除（実機確認）
- 422 は入力エラー（形式違反など）

## 外部URL画像の取り込み: `postProfileLinkMediaFromUrl`

- ボディ: `{ url }`（取得対象の公開画像URL）。返り値: `{ mediaId }`（201）
- サーバーが URL から画像を取得して MOSH のメディアとして登録し、準備完了まで待つ。`mediaId` は次にそのまま使える: 画像コンテンツの `imageId`、ウェブリンク／SNS／動画のサムネイル（`imageId` / `iconId` / `thumbnailImageId`）、ギャラリー項目の `mediaId`、プロフィールの `iconPictureId`、ヘッダー／テーマの背景 `imageId`
- http / https のみ。取り込めない URL がある（社内向けなど）。許可される画像形式は png / jpeg / gif / webp / svg / ico、サイズは10MBまで。取り込みに時間がかかると失敗することがある。条件に合わない・取得に失敗すると 400（確認済み）。**ユーザーが提示した URL だけを使う**（他サイトの画像を勝手に取り込まない。権利は依頼者の責任である旨を、必要なら一言添える）。取り込めても、gif / svg / ico を各用途（ヘッダー背景・テーマ背景・アイコンなど）に使えるかは確認できていない。**使えるかは未確認と添える**（動作確認済みなのは png をリンクタブの画像カードに使った場合だけ）
- 取り込みは外部への取得を伴い、成功すると MOSH 側に画像が登録される。実行前に「この URL の画像を取り込みます」と承認を得る

## 手元の画像ファイルを使う: `postMeMediaUploadUrls` ＋ `curl`

`postMeMediaUploadUrls` で得る ID は、画像の登録に共通のメディア ID。**`isPrivate` は付けない**（省略で公開になる。公開ページに出す画像を非公開にすると表示できない）。確認済みの用途は、リンクタブの画像カード・ヘッダー背景・ギャラリー。プロフィールアイコン・テーマ背景は同じ ID の解決だが未確認。送信直後の読み戻しで URL が解決しない場合があるため、読み戻し確認の手順は維持する。

手順:

1. **承認を取る**（枠を発行すると、使わなくてもメディアのレコードが残り、消す手段が無い。承認後は期限があるので止めずに進める）
2. `postMeMediaUploadUrls`: `{ name: "photo.jpg", type: "imageJpeg" }`（`name` は拡張子付き。`type` は拡張子から選ぶ）。`type` は `imageJpeg` / `imagePng` / `imageWebp` のみ（**gif / svg / ico は不可**）。返り値の `moshMediaId` と `uploadUrl`（**有効期限15分**）を得る
3. Bash で `curl -sS -X PUT --fail-with-body --upload-file "<ファイルパス>" "<uploadUrl>"`（Content-Type は付けなくてよい）。失敗の多くは15分の期限切れ。その場合は枠を発行し直し、古い `moshMediaId` は使わない
4. `moshMediaId` を、`imageId` / `iconPictureId` / `thumbnailImageId` / ギャラリーの `mediaId` などに渡して、通常どおり取得→提示→承認→更新する
5. **読み戻して確認する**: 送信直後は MOSH 側の変換が終わっていないことがある。**処理状況を調べる MCP は無く、プロフィールリンクの更新は未完了の ID でもエラーにならず保存される**（無言の失敗になり得る）。更新後に取得ツール（ギャラリーなら `mediaUrl` / `thumbnailUrl`、コンテンツなら `imageUrl`）で URL が空でないことを確認する。空なら数秒待って読み戻し直す（最大3回）。それでも空なら、ユーザーに「画像は送信済みだが反映待ち」と伝える

注意:
- この ID の所有はプロフィールリンク側で検証されない。**自分が発行した `moshMediaId` だけを使う**（過去の会話や他人の ID を流用しない）
- 動画ファイルは対象外。`postMeMediaUploadUrls` の `type` に動画は無い。プロフィールのアップロード動画（`videoSource: upload`）、ギャラリーの動画項目の新規追加は、このスキルでは扱わない
- file-share の素材登録（`postCreatorMaterialImage`）は使わない。共有リンクが発行されてしまう
