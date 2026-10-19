# トレース: タブで行き来する2つのメニューを作る

「作成 → 作成 → 更新」（SKILL.md §F）を1回の承認で通す例。作成 body の完成形（7項目・タブありのエリア順・3キーの action）も兼ねる。タブなしの単体作成は、手順4の①だけを行えばよい。

---

**ユーザー**: 「タブで『はじめての方』と『受講生の方』を切り替えられるリッチメニューを作りたい。ボタンはそれぞれ2つで、画像は ~/Desktop/menu-first.png と ~/Desktop/menu-member.png」

---

**1. アカウントと既存メニューを確かめる** — `getCreatorLineChannels` → 連携中は「MOSHヨガ教室」（id 55）の1つだけ。`getCreatorLineRichMenus`（`creatorLineChannelId` = 55）→ 既存は「通常メニュー」（id 1234、デフォルト設定中）。

**2. 必要な情報をそろえる**

- レイアウト: ボタン2つ＋タブ → 左右2分割（タブあり）。エリアは 4件（左タブ / 右タブ / 左 / 右）
- 各ボタンの URL は分からないので聞く。ユーザー回答: はじめての方 = 左「体験レッスン予約」https://mosh.jp/classes/111 ・右「よくある質問」は押したら案内文を返す / 受講生の方 = 左「レッスン予約」https://mosh.jp/classes/222 ・右「受講生コミュニティ」https://mosh.jp/classes/333
- 画像を実測（[../references/image-preparation.md](../references/image-preparation.md)）: 2枚とも 1200×810・PNG・0.6MB。上 180px に左右のタブが描かれ、menu-first は左タブ、menu-member は右タブが選択中の見た目 → 条件を満たす

**3. 内容を提示して承認を取る**（全操作を列挙する）

> 「MOSHヨガ教室 に、タブで切り替わるリッチメニューを2つ作ります。
>
> ① 『はじめての方』メニュー（左右2分割・タブあり）
> ・左のタブ: このメニュー自身（押しても変わりません）
> ・右のタブ: 『受講生の方』メニューに切り替え（③で設定します）
> ・左のボタン: 体験レッスン予約のページを開く
> ・右のボタン: 「よくある質問はこちら…」の案内文を返信
> ② 『受講生の方』メニュー（左右2分割・タブあり）
> ・左のタブ: 『はじめての方』メニューに切り替え／右のタブ: このメニュー自身
> ・左のボタン: レッスン予約のページ／右のボタン: 受講生コミュニティのページ
> ③ ①の右のタブの切り替え先を『受講生の方』に設定
>
> メニューバーの文言は両方「メニュー」、トーク画面を開いたときはメニューを開いた状態にします（こちらで補いました）。背景画像はお送りいただいた2枚をそのまま使います。
> 作っただけでは友だちには表示されません。今のデフォルト『通常メニュー』を『はじめての方』に置き換える場合は、作成後に改めて確認します。
> 右のボタンの案内文の返信は、LINE公式アカウントの Webhook が MOSH に向いていないと動きません。
> この内容で進めてよいですか？」

**ユーザー**: 「OK」

---

**4. 実行**

画像を上げる（2枚とも同じ手順）: `postCreatorMaterialImageUpload`（`fileName` = "menu-first.png", `mimeType` = "image/png", `fileSizeBytes` = 実測値）→ `curl` で `uploadUrl` へ PUT → `moshMediaId` を得る（menu-first = `01JAAAA...`、menu-member = `01JBBBB...`）。

**① 『はじめての方』を作成** — `postCreatorLineRichMenu`。右のタブはまだ相手が無いので `null`。

```json
{
  "bodyParams": {
    "creatorLineChannelId": 55,
    "name": "はじめての方",
    "layout": "LEFT_RIGHT_SPLIT_WITH_TWO_TABS",
    "backgroundMoshImageId": "01JAAAAAAAAAAAAAAAAAAAAAAA",
    "areas": [
      { "label": "TAB1", "action": { "url": null, "postbackActions": null, "richMenuSwitch": null } },
      { "label": "TAB2", "action": { "url": null, "postbackActions": null, "richMenuSwitch": null } },
      { "label": "A", "action": { "url": "https://mosh.jp/classes/111", "postbackActions": null, "richMenuSwitch": null } },
      { "label": "B", "action": { "url": null, "postbackActions": [ { "postbackActionType": "sendMessage", "messages": [ { "messageType": "text", "message": "よくある質問はこちらです。…" } ] } ], "richMenuSwitch": null } }
    ],
    "chatBarText": "メニュー",
    "chatBarOpenByDefault": true
  }
}
```

返り値 `{ "id": 1301 }`

> 「背景画像の取得に失敗しました。」が返ったら、上げた直後の準備待ち。同じ body のまま10秒ほど待って最大3回まで再送してよい（SKILL.md「エラーが返ったとき」の例外）。

**② 『受講生の方』を作成** — 左のタブに ① の id を指定する。

```json
{
  "bodyParams": {
    "creatorLineChannelId": 55,
    "name": "受講生の方",
    "layout": "LEFT_RIGHT_SPLIT_WITH_TWO_TABS",
    "backgroundMoshImageId": "01JBBBBBBBBBBBBBBBBBBBBBBB",
    "areas": [
      { "label": "TAB1", "action": { "url": null, "postbackActions": null, "richMenuSwitch": { "richMenuId": 1301 } } },
      { "label": "TAB2", "action": { "url": null, "postbackActions": null, "richMenuSwitch": null } },
      { "label": "A", "action": { "url": "https://mosh.jp/classes/222", "postbackActions": null, "richMenuSwitch": null } },
      { "label": "B", "action": { "url": "https://mosh.jp/classes/333", "postbackActions": null, "richMenuSwitch": null } }
    ],
    "chatBarText": "メニュー",
    "chatBarOpenByDefault": true
  }
}
```

返り値 `{ "id": 1302 }`

**③ 『はじめての方』の右のタブを設定** — 全置換の更新なので、まず `getCreatorLineRichMenu`（id 1301）で取得し、**TAB2 の `richMenuSwitch` だけ**を `{ "richMenuId": 1302 }` に差し替えて、残りは取得値のまま `patchCreatorLineRichMenu`（id 1301）に送る（SKILL.md §A）。

---

**5. 報告**

> 「『はじめての方』と『受講生の方』の2つのメニューを作り、タブで行き来できるように設定しました。まだ友だちには表示されていません。『はじめての方』を全ての友だちの初期表示（今は『通常メニュー』）に置き換えますか？」

→ 置き換えるなら、デフォルト設定の承認を改めて取る（SKILL.md 承認の判定表）。

**よくある壊し方**（この例で避けているもの）:

- ①で右のタブに 1302 を入れようとする → まだ存在しないので「切り替え先のリッチメニューが見つかりません。」
- タブありなのにエリアを2件で送る → 件数不一致（タブ2件を含めて4件）
- タブを配列の末尾に置く → タブの位置にボタンの動作が、ボタンの位置にタブの動作が割り当たったまま公開される
- 画像の上 180px にタブを描いていない → タブがどこにあるか分からないメニューが公開される（エラーにならない）
- 別の LINE公式アカウントのメニューを切り替え先にする → 「切り替え先のリッチメニューを指定できません。」
