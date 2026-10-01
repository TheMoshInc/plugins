# 例: 動画 1 本を上げてレッスンにする

前提: Claude Code（シェル実行可）。ID・URL はダミー値。実際は取得した値を使い、ユーザーには見せない。

## 発話

> 「オンライン講座」の会員サイトの「第1章」に、~/Movies/lesson01.mp4 をレッスンとして追加して。字幕も付けて。

## 1. 聞き足す（1 往復）

- AI 要約・目次を自動で作るか → 「作って」
- タイトル → 「第1回 はじめに」
- 公開のタイミング → 「確認してから」

## 2. ID をそろえる

- `getCreatorMembershipSites` → 「オンライン講座」の `id: 12`
- `getCreatorMembershipSiteFolders { id: 12 }` → 「第1章」の `id: 345`

## 3. サイズ → 発行

```bash
stat -f%z ~/Movies/lesson01.mp4   # → 734003200
```

`postCreatorMembershipSiteAssets`

```json
{
  "pathParams": { "id": 12 },
  "bodyParams": { "assetType": "video", "fileSizeBytes": 734003200, "isGenerateSubtitles": true, "isGenerateAiSummary": true }
}
```

→ `uploadUrl`, `assetId: 6789`, `uploadId`

ユーザーへ: 「アップロードを始めます。」

## 4. PUT

```bash
curl -sS -X PUT -T ~/Movies/lesson01.mp4 "<uploadUrl>" -o /dev/null -w '%{http_code}\n'   # → 200
```

## 5. READY 待ち

`getCreatorMembershipSiteAsset { id: 12, assetId: 6789 }` を 10 秒程度の間隔で呼ぶ → `PROCESSING` … → `READY`

ユーザーへ: 「動画の変換が終わりました。」

## 6. 非公開でコンテンツを作る

`postCreatorMembershipSiteContents`

```json
{
  "pathParams": { "id": 12 },
  "bodyParams": {
    "folderId": 345,
    "contentType": "video",
    "title": "第1回 はじめに",
    "body": "",
    "description": "",
    "chapters": [],
    "thumbnailAssetId": null,
    "assetIds": [6789],
    "isPublished": false,
    "isNotifyOnPublish": false,
    "isCommentEnabled": false,
    "isCompletionButtonVisible": true,
    "scheduledPublishAt": null,
    "scheduledUnpublishAt": null,
    "visibleAfterPurchaseDays": null,
    "unpublishAfterPurchaseDays": null,
    "tagIds": []
  }
}
```

## 7. 読み戻して提示 → 承認後に公開

`getCreatorMembershipSite { id: 12 }` と `getCreatorMembershipSiteFolders { id: 12 }` で、サイトと「第1章」が公開か確かめる（どちらかが非公開なら、コンテンツを公開しても会員に見えないことを添える）。

ユーザーへ:

> 「第1章」に動画レッスン「第1回 はじめに」を非公開で追加しました。字幕と AI 要約・目次は自動で作られます。サムネイル画像は付けていないため、一覧と再生画面には動画から作られた画像が出ます。
> 公開しますか？公開時に購入者へメールで通知しますか？

承認後: `patchCreatorMembershipSiteContent { isPublished: true, isNotifyOnPublish: <回答> }`
