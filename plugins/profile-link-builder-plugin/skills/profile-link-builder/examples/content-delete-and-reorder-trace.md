# トレース例: リンクタブのコンテンツ削除と並べ替え

一覧を全件取り切る → 削除の確認 → 残りの並べ替え、の流れ。

## ユーザー発話

> リンクタブの「キャンペーン告知」の見出しを消して、残りは「無料相談」のリンクを一番上にして。

## 1. 一覧を全件取得

```
getProfileLinkContents(queryParams: { limit: 100, offset: 0 })
→ contents: [
    {id:"c1", type:"heading", title:"キャンペーン告知"},
    {id:"c2", type:"image",   title:"プロフィール写真"},
    {id:"c3", type:"link",    title:"無料相談"},
    {id:"c4", type:"sns",     title:"Instagram"} ], totalCount: 4
```

`totalCount`（4）と取得件数（4）が一致 → 全件取れた。もし件数が足りなければ offset を進めて取り切ってから進む。同名が複数あれば、どれかをユーザーに選んでもらう。

## 2. 提示と承認（削除は対象を名指し）

> 次の2つを行います。
> 1. 見出し「キャンペーン告知」を**削除**します（元に戻せません）
> 2. 残り3件の順番を、「無料相談」→「プロフィール写真」→「Instagram」にします
>
> 進めてよいですか？

## 3. 実行（削除 → 並べ替えの順）

```
deleteProfileLinkContent(pathParams: { id: "c1" })

putProfileLinkContentsLayout(bodyParams: {
  items: [
    { contentId: "c3", displayOrder: 0 },
    { contentId: "c2", displayOrder: 1 },
    { contentId: "c4", displayOrder: 2 }
  ]
})
```

- `items` には並べ替え後の**全コンテンツ**を入れるのが確実（一部だけ送ると、入れなかったものが、並べ替えたものの間に入り込む場合がある）
- 削除済みの `c1` を `items` に含めない（アーカイブ済み ID が混ざると 404 で1件も更新されない）

## 4. 読み戻しと報告

もう一度 `getProfileLinkContents` で並びを確認し、意図どおりなら報告する。削除が失敗した場合は並べ替えに進まず、状況を伝えて止まる。

> 「キャンペーン告知」の見出しを削除し、「無料相談」を一番上にしました。
