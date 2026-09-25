# トレース: タグ名で顧客を絞りたいと言われたとき

このスキルで最も間違えやすい依頼。コンタクトタグの ID を代用すると**エラーにならずに間違った結果が返る**（SKILL.md 必須ルールA）。顧客タグの id は `getCreatorCustomerTags` で名前から引く（必須ルールB）。

## 発話

> 「VIPタグが付いてる顧客だけ一覧で出して」

## やってはいけない処理

1. `getCreatorContactTags` を呼ぶ → 「VIP」という名前のコンタクトタグが見つかる（id: `8821`、整数）
2. その id を文字列にして `tagIds: ["8821"]` に入れ、`postCustomersSearch` を呼ぶ
3. `totalCount: 0` が返る（または、偶然 id `"8821"` の別の顧客タグがあれば、その顧客が返る）
4. 「VIPタグの顧客はいませんでした」（または無関係な顧客の一覧）と答える

**何が起きたか**: 顧客タグとコンタクトタグは別のID体系なので、渡した id は VIP の顧客タグを指していない。見た目では区別できない（[customer-vs-contact.md](../references/customer-vs-contact.md)）。エラーが出ないので、間違いに気づけないまま誤った報告をしてしまう。付与・解除でこれをやると、無関係のタグが顧客に付く・外れる。

もう1つの NG: 条件なしで `postCustomersSearch` を呼び、返ってきた顧客の `tags[]` から「VIP」っぽいタグを拾う（理由は [customer-vs-contact.md](../references/customer-vs-contact.md)）。

## 正しい処理

### 1. 顧客タグの一覧から id を引く

「顧客」と言われているので顧客タグとして扱う。`getCreatorCustomerTags` を呼ぶ（引数なし）。

```json
{ "customerTags": [
  { "id": "12", "name": "VIP" },
  { "id": "15", "name": "VIP候補" },
  { "id": "20", "name": "リピーター" }
] }
```

名前が**完全一致**するのは id `"12"` だけ。「VIP候補」は含めない（ユーザーが「VIP系全部」と言ったときだけ含める）。

### 2. その id で検索する

```json
{
  "bodyParams": {
    "serviceIds": [], "subscriptionIds": [], "eventDateTime": null,
    "searchQuery": "", "subscriptionState": null, "paymentMethod": null,
    "paymentType": null, "paymentStatus": null, "isEmailReceivingAllowed": null,
    "tagIds": ["12"], "excludeTagIds": [], "userIds": [],
    "page": 1, "limit": 10
  }
}
```

### 3. ユーザーへの応答

> 顧客タグ「VIP」が付いている顧客は 23 名です。最初の 10 名を表示します。
>
> 1. 山田 花子
> 2. ...
>
> 続きも表示しますか？（なお「VIP候補」という別のタグもありますが、今回は含めていません）

**ポイント**

- `getCreatorContactTags` は呼ばない
- 完全一致が無いとき（例: 「VIP」が無く「VIP会員」だけある）は、推測で使わず「『VIP会員』のことですか？」と確認する。一覧に似た名前も無ければ「顧客タグ『VIP』は見つかりませんでした」と伝え、今あるタグ名を示す
- 内部の値（`tagIds`・id 文字列・JSON）を応答に出さない
- 0 件だった場合は、タグ名の取り違え（LINE友だち側のタグのことではないか）をユーザーと確認する
