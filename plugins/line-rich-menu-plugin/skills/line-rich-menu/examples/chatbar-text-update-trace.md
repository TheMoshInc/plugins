# トレース: 「メニューバーの文言だけ変えて」

1項目だけの変更依頼を、全置換の更新で安全に通す最小例。**変更しない5項目を取得値からそのまま載せ直す**のが要点。

---

**ユーザー**: 「リッチメニューの下のバーの文字、『メニュー』から『メニューを開く』に変えといて」

---

**1. 対象を特定する** — `getCreatorLineRichMenus`（引数なし）

返り値（抜粋）:

```json
{
  "lineRichMenus": [
    { "id": 1234, "name": "通常メニュー 2026年9月", "creatorLineChannelId": 55, "isDefault": true, "linkedCount": 0, "backgroundMoshImageId": "01JXXX...", "updatedAt": "2026-08-20T02:10:00.000Z" },
    { "id": 1180, "name": "購入者向けメニュー", "creatorLineChannelId": 55, "isDefault": false, "linkedCount": 312, "backgroundMoshImageId": "01JYYY...", "updatedAt": "2026-07-02T09:44:00.000Z" }
  ],
  "totalCount": 2
}
```

2件あり、どちらか特定できない。**推測せず聞く。**

> 「リッチメニューが2つあります。どちらのメニューバーの文言を変えますか？
> ・通常メニュー 2026年9月（いま初期表示に設定されているもの）
> ・購入者向けメニュー」

**ユーザー**: 「上のやつ」

---

**2. 現在の内容を取得する** — `getCreatorLineRichMenu`（`pathParams.id` = 1234）

> 文言だけの変更でも**この取得を省かない**。取得した値が更新 body の材料になる。

返り値:

```json
{
  "id": 1234,
  "creatorLineChannelId": 55,
  "name": "通常メニュー 2026年9月",
  "layout": "LEFT_RIGHT_SPLIT",
  "backgroundMoshImageId": "01JXXXXXXXXXXXXXXXXXXXXXXX",
  "areas": [
    { "label": "A", "action": { "url": "https://mosh.jp/classes/12345", "postbackActions": null } },
    { "label": "B", "action": { "url": null, "postbackActions": [ { "postbackActionType": "addContactTag", "contactTagId": 87 } ] } }
  ],
  "chatBarText": "メニュー",
  "chatBarOpenByDefault": true,
  "isDefault": true,
  "createdAt": "2026-06-01T04:00:00.000Z",
  "updatedAt": "2026-08-20T02:10:00.000Z"
}
```

**3. 文字数を確認する** — 「メニューを開く」は 7 文字（上限14以内）。絵文字を含む依頼ならここでコードユニット数を数える。

---

**4. 変更内容を提示して承認を取る**（内部の項目名・JSON は出さない）

> 「通常メニュー 2026年9月 のメニューバーの文言を変更します。
>
> ・メニューバーの文言: 「メニュー」→「メニューを開く」
> ・レイアウト、背景画像、左右2つのボタンの動き、開いた状態で表示する設定は**そのまま**です
>
> このメニューは現在すべての友だちの初期表示に設定されているため、更新すると公開中の表示がすぐ切り替わります。進めてよいですか？」

**ユーザー**: 「OK」

---

**5. 更新する** — `patchCreatorLineRichMenu`

`chatBarText` **以外の5項目は、手順2の返り値をそのまま転記する**（`id` / `creatorLineChannelId` / `isDefault` / 日時は body に含めない）。

```json
{
  "pathParams": { "id": 1234 },
  "bodyParams": {
    "name": "通常メニュー 2026年9月",
    "layout": "LEFT_RIGHT_SPLIT",
    "backgroundMoshImageId": "01JXXXXXXXXXXXXXXXXXXXXXXX",
    "areas": [
      { "label": "A", "action": { "url": "https://mosh.jp/classes/12345", "postbackActions": null } },
      { "label": "B", "action": { "url": null, "postbackActions": [ { "postbackActionType": "addContactTag", "contactTagId": 87 } ] } }
    ],
    "chatBarText": "メニューを開く",
    "chatBarOpenByDefault": true
  }
}
```

**よくある壊し方**（いずれもこの例で避けているもの）:

- `areas` を省く / 空配列にする → 必須項目欠落またはレイアウトとの件数不一致で入力エラー（400）
- `layout` を送らない → エラー
- `areas` を「2件だから」と `label` 順を入れ替える → 位置とラベルがズレたまま公開される（エラーにならない）
- 取得を省いて記憶で `backgroundMoshImageId` を書く → 別の画像に置き換わる

---

**6. 報告する**

> 「メニューバーの文言を『メニューを開く』に変更しました。他の設定は変えていません。LINE アプリ側の表示が切り替わるまで少し時間がかかることがあります。」
