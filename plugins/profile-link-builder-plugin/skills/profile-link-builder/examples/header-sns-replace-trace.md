# トレース例: ヘッダーSNSの追加と並べ替え（配列の全置換）

「1件追加」と「並べ替え」で使うツールが違うこと、全置換で取りこぼし・重複を起こさない手順を示す。

## A. 1件追加: 「ヘッダーにInstagramを足して」

ユーザーから URL（`https://www.instagram.com/example`）を得たら、**追加は `postProfileLinkHeaderSnsLink`**。現在の件数は先に一覧で確認する（重複追加の防止）。

```
getProfileLinkHeaderSnsLinks(queryParams: { limit: 100, offset: 0 })
→ snsLinks: [ {id:"s1", snsType:"x", url:"https://x.com/example", orderNum:0},
              {id:"s2", snsType:"youtube", url:"https://youtube.com/@example", orderNum:1} ], totalCount: 2
```

> 今のヘッダーには X と YouTube が並んでいます。末尾に Instagram（https://www.instagram.com/example）を追加します。進めてよいですか？

承認後:

```
postProfileLinkHeaderSnsLink(bodyParams: { snsType: "instagram", url: "https://www.instagram.com/example" })
```

## B. 並べ替え: 「YouTubeを一番左にして」

並べ替えは全件の最終形を送る **`putProfileLinkHeaderSnsLinks`**。**取得した全件の `id` を引き継ぐ**（含めなかったものは削除される）。

```
putProfileLinkHeaderSnsLinks(bodyParams: {
  links: [
    { id: "s2", snsType: "youtube",   url: "https://youtube.com/@example" },
    { id: "s1", snsType: "x",         url: "https://x.com/example" },
    { id: "s3", snsType: "instagram", url: "https://www.instagram.com/example" }
  ]
})
```

配列の順番がそのまま表示順。提示では「消えるものが無い（3件すべて残る）」ことを明示する。

## NG（やってはいけない）

- 並べ替えで取得し損ねた件（limit 20 超）を落とす → その SNS が削除される
- 追加のために `put` で新規1行だけ送る → 既存が全部消える
- `postProfileLinkHeaderSnsLink` が失敗・応答欠落したあと、そのまま再送する → 重複することがある（実機確認）。まず一覧を取り直して状態を確認する
- 取得結果にない `id` や、他の対象の `id` を創作・流用する
