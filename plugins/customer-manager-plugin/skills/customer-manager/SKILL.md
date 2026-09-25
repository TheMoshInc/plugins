---
name: customer-manager
description: MOSH のサービス購入者＝ゲスト（顧客）を MCP 経由で検索・件数確認し、顧客1人の詳細（プロフィール・付いているタグ・申込/予約履歴）を確認し、顧客タグの作成・付与・解除を行うスキル。「顧客一覧を見せて」「◯◯さんっていう顧客を探して」「このサービスを買った人は誰？」「ゲストは何人いる？」「サブスク継続中の顧客は何人いる？」「メール受信OKの顧客だけ絞って」「VIPタグの顧客を出して」「この顧客の電話番号と付いてるタグを見せて」「◯◯さんの申込履歴を教えて」「顧客にVIPタグを付けて」「新しい顧客タグを作って付けて」「この人からタグを外して」「顧客タグの一覧を見せて」など、`postCustomersSearch` / `getCreatorCustomer` / `getCreatorCustomerReservations` / `getCreatorCustomerTags` / `postCreatorCustomerTags` / `postCreatorCustomerTagCustomers` / `deleteCreatorCustomerTagCustomer` を使うリクエストで必ず起動し、顧客タグとコンタクトタグの取り違え防止・書き込み前の承認ゲート・個人情報の提示ルールを適用する。顧客カルテ（メモ・画像）の閲覧編集、プロフィール編集、顧客の削除、顧客タグ自体の名前変更・削除は MCP からできない。LINE友だち・メール購読者の「コンタクトリスト」とそのタグは別概念のため対象外（contact-list-manager / workflow-builder の担当）。
---

# MOSH 顧客管理（サービス購入者の検索・詳細確認・顧客タグ）

## Overview

クリエイターのサービスを購入した顧客（管理画面「顧客・連絡先 > 顧客一覧」）を扱う。できることは3系統:

| 系統 | ツール | 書き込み |
|---|---|---|
| 検索・件数 | `postCustomersSearch` | なし |
| 1人の詳細 | `getCreatorCustomer`（プロフィール＋付いているタグ）/ `getCreatorCustomerReservations`（申込・予約履歴） | なし |
| 顧客タグ | `getCreatorCustomerTags`（一覧）/ `postCreatorCustomerTags`（作成＋1人に付与）/ `postCreatorCustomerTagCustomers`（既存タグを最大100人に付与）/ `deleteCreatorCustomerTagCustomer`（1人から解除） | **あり**（必須ルールD の承認ゲート） |

**顧客タグとコンタクトタグは別のID体系。** コンタクトタグの id を顧客タグとして使っても、エラーになるとは限らない。id が偶然一致すると、検索では別タグの顧客が返り、付与・解除では**無関係の顧客タグが書き換わる**（必須ルールA）。

> **ツール名表記について**: 本スキルでは MCP ツールを OpenAPI の `operationId` で参照する。MCP クライアントが提示する実際のツール名にはサーバー識別子のプレフィックスが付くが、その形はユーザー環境により異なるため本スキルでは付けない。

## When to use

- 購入者の一覧・件数を知りたいとき（「顧客は何人いる？」「一覧を見せて」）
- 名前・メールアドレス・MOSH ID で特定の顧客を探したいとき
- 購入条件・顧客タグで絞りたいとき（サブスクの継続／解約、支払い方法、支払いタイプ、決済ステータス、メール受信可否、顧客タグ）
- 特定の顧客の電話番号・性別・付いているタグ・申込/予約履歴を確認したいとき
- 顧客タグの一覧を見たい、顧客にタグを付けたい・外したい、新しい顧客タグを作りたいとき
- 施策の対象者がどれくらいいるかを、実行前に把握したいとき

## When NOT to use

- **コンタクト（LINE友だち・メール購読者）の検索・削除・CSVインポート** → `contact-list-manager` スキル。顧客とは別概念（[references/customer-vs-contact.md](references/customer-vs-contact.md)）
- **メルマガ・LINE の一斉配信を作る／送る** → `contact-broadcast` スキル
- **顧客管理からの一斉配信を送る／編集する／取り消す** → `customer-broadcast` スキル
- **コンタクトへのタグ付与・ステップ配信の構築** → `workflow-builder` スキル（そこで扱うのは**コンタクトタグ**）
- **顧客カルテ（メモ・画像）の閲覧・編集、プロフィール編集、顧客の削除、顧客タグ自体の名前変更・削除** → MCP に手段が無い（必須ルールD）。管理画面「顧客一覧」を案内する
- **売上・注文明細の金額集計** → `sales-reporter` スキル
- **商品・プランの内容やプラン価格の確認** → `product-navigator` スキル
- **特定商品のサブスク購読者を登録日順・ステータス別に見る** → `postCreatorProductSubscribers`（商品単位の別ツール。本スキルの守備範囲外）

## Workflow

最初に依頼を切り分け（1）、依頼の種類ごとに 2〜4 のどれかに進む。

### 1. 依頼の切り分け

- 「顧客（購入者）」の話か「コンタクト（LINE友だち・メール購読者）」の話かを確定する。「登録者」「お客さん」「リスト」のようにどちらとも取れる言い方なら**ユーザーに聞く**。判断表は [references/customer-vs-contact.md](references/customer-vs-contact.md)
- 顧客の話なら、**検索（2）／1人の詳細（3）／タグの付け外し（4）**のどれかを決める。MCP に無い操作（カルテ・プロフィール編集・削除・タグの改名や削除）なら必須ルールD に従って案内して終える

### 2. 検索・件数（`postCustomersSearch`）

1. **条件の組み立て**: 依頼の言葉を検索条件に対応づける。**14フィールドすべて必須**（必須ルールC）。対応表は [references/mcp-tools.md](references/mcp-tools.md)、完成形は [examples/search-body.json](examples/search-body.json)
2. **ID が要る条件の解決**: 顧客タグ名 → `getCreatorCustomerTags` で id を引く（必須ルールB）。サービス名 → `product-navigator` で `moshServiceId` を得る。解決できなければその条件は空配列のままユーザーに確認する
3. **実行**: `{ "bodyParams": { ... } }` の形で呼ぶ。`limit` は通常 10、一覧をまとめて見たい依頼でも最大 100
4. **件数とページング**: `totalCount` が条件に一致した総件数。
   - 件数だけ聞かれている → 1ページ目を1回呼んで `totalCount` を答える。全ページを取りに行かない
   - 一覧が必要 → `page` を増やして必要な分だけ取る。何百件も無言で全ページ取得せず、件数を伝えて「何件まで出すか」を確認する
   - `totalCount: 0` はエラーではない。条件のどれが効きすぎているかを添えて伝える
5. **提示**: 必要な項目だけを出し（必須ルールE）、適用した条件を平易な言葉で1行添える

### 3. 1人の詳細（`getCreatorCustomer` / `getCreatorCustomerReservations`）

1. **顧客ID を確定する**: ユーザーが顧客IDを明示していなければ、名前・メールで `postCustomersSearch` を呼び `customers[].id` を得る。**複数ヒットしたら、どの人かをユーザーに選んでもらう**（名前だけで1人に決めつけない）
2. **必要なほうだけ呼ぶ**: プロフィール・タグ・電話番号・性別なら `getCreatorCustomer`、申込・予約履歴なら `getCreatorCustomerReservations`。両方聞かれたときだけ両方呼ぶ
3. **申込履歴の期間指定**: 「今年」「先月」などは `reservedAtFrom` / `reservedAtTo` に日本時間（`+09:00`）の ISO 8601 で入れる。期間が無ければ省略（全期間）。件数が多そうなら `limit`（最大100）と `offset` で区切る。`queryParams` は条件が無くても `{}` で渡す
4. **提示**: 聞かれた項目だけを答える。表示ラベルは [references/mcp-tools.md](references/mcp-tools.md) の対応表に従う（性別 `none` は「無回答」、空文字のメール・電話は「未設定」）

### 4. 顧客タグの付け外し（書き込み）

1. **対象の顧客を確定する**: 3-1 と同じ。「◯◯を買った人全員に」のような条件指定なら、Workflow 2 で検索して対象者と人数を確定する
2. **タグを解決する**: `getCreatorCustomerTags` で一覧を取り、**名前が完全一致するタグ**を探す（必須ルールB）
   - 完全一致が1つ → その id を使う
   - 無い → 新規作成になる。表記ゆれ（全角半角・大文字小文字・似た名前）の既存タグがあれば、それを使うか新規作成かをユーザーに聞く
3. **承認を取る**（必須ルールD）: 「タグ名／新規作成か既存か／対象の顧客（人数と、少人数なら名前）／付与か解除か」を1つにまとめて提示し、明示的な承認を得る
4. **実行**:
   - 既存タグを付ける → `postCreatorCustomerTagCustomers`（1回100人まで。超える分は100人ずつ分けて呼ぶ）
   - 新しいタグを作って付ける → まず1人目に `postCreatorCustomerTags` で作成＋付与し、返ってきた id で残りの顧客に `postCreatorCustomerTagCustomers`
   - 外す → `deleteCreatorCustomerTagCustomer` を顧客1人ずつ呼ぶ
5. **読み戻し**: 少人数なら `getCreatorCustomer` で付いたこと／外れたことを確認してから完了を報告する。途中で失敗した場合の扱いは「エラーが返ったとき」

## 必須ルール

### A. 顧客タグとコンタクトタグを混同しない【最重要】

`getCreatorContactTags` が返す**コンタクトタグ**の id は、`tagIds` / `excludeTagIds`・付与・解除の**どれにも渡してはならない**。コンタクトタグ側の書き込みツール（`postCreatorContactTags` 等）で顧客へのタグ付けを代用しない。

- **見分け方**: **整数の id が手元にあればコンタクトタグ由来を疑う**（文字列化すると見た目で区別できない理由は [references/customer-vs-contact.md](references/customer-vs-contact.md)）
- 同名のタグが両方に存在し得るため、名前の一致は同一である根拠にならない

顧客タグ id の入手経路、なぜ検索結果のタグを使わないか、振り分けの判断は [references/customer-vs-contact.md](references/customer-vs-contact.md) が正本。

### B. ID を推測・創作しない

- **顧客タグの id**: タグ名からの解決は `getCreatorCustomerTags` の一覧での**完全一致**だけ（理由は [references/customer-vs-contact.md](references/customer-vs-contact.md)）。完全一致が無ければ id を埋めずにユーザーに確認する
- **顧客ID**: ユーザーが明示したか、`postCustomersSearch` の結果で本人と確定した `customers[].id` だけを使う。名前が似ているだけで決めつけない
- **`serviceIds` / `subscriptionIds`**: `product-navigator`（`getCreatorProductPlan` / `getCreatorProductPlans`）の `moshServiceId`（文字列）を数値に変換して使う（[references/mcp-tools.md](references/mcp-tools.md)）。商品プランに紐づいていない古い形式のサービスはこの経路が無く、空配列でユーザーに確認する
- **型に注意**: `postCustomersSearch` の `tagIds` / `excludeTagIds` / `userIds` は**文字列の配列**（`["42"]`）。一方、顧客詳細・予約・タグ付与の顧客ID（`id` / `customerId` / `customerIds`）は**整数**。タグの id はどのツールでも文字列

### C. `postCustomersSearch` は14フィールドすべて必須

使わない条件も既定値（配列は `[]`、`searchQuery` は `""`、それ以外の条件は `null`）で必ず埋める。フィールドごとの既定値は [references/mcp-tools.md](references/mcp-tools.md) の表、そのまま使える完成形は [examples/search-body.json](examples/search-body.json)。以下の2つだけは特に間違えやすい。

**`isEmailReceivingAllowed` の `false` は有効な絞り込み値**（＝受信停止中の顧客）。「絞り込まない」は `null`。

**enum の値は綴りをそのまま使う。** 特に `paymentMethod` の `bank-transfer` は**ハイフン区切り**（`bank_transfer` にしない。実例で確認済みの罠）。使うときは [examples/search-body.json](examples/search-body.json) の `bank-transfer-installment` の綴りをコピーする。

### D. 書き込みは承認してから。MCP に無い操作は代用しない

- **タグの作成・付与・解除は、Workflow 4-3 の確認を出してユーザーが明示的に承認してから実行する。** 「付けておいて」と頼まれていても、対象の顧客・タグ名・新規か既存かが確定するまでは実行しない。承認後に対象やタグが変わったら取り直す
- **一度に大量の顧客を変更するときは、人数を必ず示す**（「対象は 230 名です。100 名ずつ 3 回に分けて付与します」）
- **作成したタグは MCP から名前変更・削除できない。** 新規作成の確認時に「作ったタグを消すのは管理画面になります」と添える
- **MCP に無い操作**（カルテの閲覧・編集、プロフィール編集、顧客の削除、顧客タグ自体の名前変更・削除）は、別のツールで代用したり、できたかのように答えたりしない。管理画面「顧客・連絡先 > 顧客一覧」での操作を案内する
- **管理画面の中の手順は創作しない。** 案内してよいのは「顧客・連絡先 > 顧客一覧」（と、そこから開く顧客詳細）までで、その先のボタン名・メニュー名・「タグ管理を開いて名前を書き換える」といった操作手順は確認できていない。「顧客タグの名前変更は MOSH の管理画面からの操作になります」で止め、具体的な手順は画面で確認してもらう

### E. 個人情報の取り扱い

レスポンスには顧客名・メールアドレス・電話番号・予約時の連絡事項が含まれる。**ユーザーに依頼された用途以外に出力・転記しない。**

- 件数を聞かれたら件数を答える。名簿を勝手に展開しない
- メールアドレス・電話番号・予約時の連絡事項は、依頼が明示的にそれを必要とする場合だけ出す（「タグを見せて」に電話番号を添えない）
- 取得した顧客情報を、依頼されていないファイル・外部サービス・別の会話文脈に書き出さない

### F. 顧客検索に期間の条件は無い

`postCustomersSearch` の `eventDateTime` は開催日時の**分単位の完全一致**でしか絞り込めず（[references/mcp-tools.md](references/mcp-tools.md)）、原則 `null`。「今月申し込んだ顧客」のような**顧客全体を期間で絞る依頼には使えない**。

`getCreatorCustomerReservations` の `reservedAtFrom` / `reservedAtTo` は申込日時の範囲で絞れるが、**顧客1人ごと**のツール。全顧客に対して1人ずつ呼んで「今月の申込者一覧」を組み立てない（顧客数だけ呼び出しが増え、途中で打ち切ると漏れが出る）。顧客全体の期間集計は管理画面、金額なら `sales-reporter` を案内する。

## ユーザーへの提示・コミュニケーション規約

- 平易な日本語で答える。**内部の値・フィールド名をそのまま出さない**（`active` / `bank-transfer` / `one_time` / `approved` / `isEmailReceivingAllowed` / `totalCount` / `male` / `none` など）。画面ラベルとの対応表は [references/mcp-tools.md](references/mcp-tools.md) が正本
- JSON・HTTP用語・ツール名・エンドポイント名をユーザー向け文面に出さない。タグの id もユーザーに見せる必要は無い（タグ名で話す）
- **できない理由を仕組みで説明しない。**「ツールが公開されていない」ではなく「◯◯は MOSH の管理画面からの操作になります」と伝える
- **ID・件数・条件を推測で埋めない。** 分からない項目は埋めずに聞く
- 該当ゼロのときは「いません」で終わらせず、効いている条件を伝えて絞り込みの見直しを提案する
- 書き込みの完了報告は、実際に成功した範囲だけを書く（「230 名中 200 名に付与済み、残り 30 名は失敗」）

## よくあるミス

他セクション・references でカバーされない落とし穴のみ。

| NG | OK |
|---|---|
| `totalCount` が 250 なのに、1ページ目の 10 件だけを見て「顧客は10人です」と答える | `totalCount` が総件数。取得件数と混同しない（Workflow 2-4） |
| 「解約した人を除いて」と言われて反射的に `subscriptionState: "active"` を入れる | `subscriptionState` はサブスク契約の状態を見る条件。単発購入だけの顧客が結果からこぼれないか、対象範囲をユーザーと確認してから使う |
| 依頼に無い条件（例: `paymentStatus: "approved"`）を「たぶんこうだろう」と足す | 依頼された条件だけを入れる。迷ったらユーザーに聞く |
| 既存タグを付けたいだけなのに `postCreatorCustomerTags` を顧客の人数分呼ぶ | 既存タグの付与は `postCreatorCustomerTagCustomers` 1回（100人まで）。`postCreatorCustomerTags` は新規作成のときの1人目だけ |
| 「VIP」を付けたいとき、一覧に「VIP会員」しか無いのにそれを使う／黙って「VIP」を新規作成する | 完全一致が無ければ「既存の『VIP会員』を使うか、新しく『VIP』を作るか」を聞く |
| 顧客詳細の `email` が空文字なのを見て「メールアドレスが取得できませんでした（エラー）」と答える | 空文字は未登録。「メールアドレスは登録されていません」と伝える（検索結果では `null` が同じ意味） |

## エラーが返ったとき

- **該当ゼロ（検索の `totalCount: 0`、予約の `reservations: []`）**: エラーではない。同じ条件で再実行せず、効いている条件を伝える。**タグ条件を使っている場合は、まず必須ルールA の取り違えを疑う**
- **404（顧客詳細・予約・タグ付与・解除）**: 顧客が見つからない（他クリエイターの顧客も同じ扱い）、タグが見つからない、解除の場合はそのタグが付いていない。どれかをユーザーに伝え、ID を推測で差し替えて再実行しない。**逆に、成功したからといって正しいタグだったとは限らない**（必須ルールA）
- **403（顧客詳細・予約）**: その顧客を見る権限が無い。ID を差し替えて再実行せず、ログインしているアカウントの確認を案内する
- **一括付与の途中で失敗した**: `postCreatorCustomerTagCustomers` は先頭から順に付けるため、失敗より前の顧客には付与済み。同じ呼び出しを再実行しても二重には付かない。失敗の原因（見つからない顧客）を取り除いてから再実行し、完了報告は成功した範囲だけにする
- **400（タグ作成）**: MOSH 側でタグ名が受け付けられなかった。名前を見直してユーザーに確認する（空・51文字以上はこれより前のスキーマ検証で弾かれる）
- **401（認証エラー）**: MCP のアクセストークン失効の可能性。同じリクエストを繰り返さず、接続設定の確認を案内する
- **500 / タイムアウト**: 時間を置いて1度だけ再試行する。それでも失敗するならサーバー側の問題として伝える。**条件を勝手に変えて通った結果を、依頼どおりの結果として出さない**。書き込みの場合は、再試行前に `getCreatorCustomer` で反映済みかを確かめる
- **スキーマ検証で弾かれた**: 必須フィールドの欠落・enum 違反・`limit` の範囲外・タグ名の長さ（1〜50文字）・型違い（文字列と整数の取り違え）。[references/mcp-tools.md](references/mcp-tools.md) の型定義に戻って組み直す

## References

| ファイル | 内容 | いつ読むか |
|---|---|---|
| [references/mcp-tools.md](references/mcp-tools.md) | 7ツールの全パラメータ・型・enum と画面ラベルの対応・レスポンス・MCP に無いもの（正本） | ツールを呼ぶ直前、条件を組み立てるとき |
| [references/customer-vs-contact.md](references/customer-vs-contact.md) | 顧客とコンタクトの違い、2つのタグID体系、依頼の振り分け表（正本） | 依頼が顧客かコンタクトか迷うとき、タグを扱うとき |
| [examples/search-body.json](examples/search-body.json) | 条件なし全件・複合条件の完成形リクエストボディ | 検索ボディの形に迷ったとき |
| [examples/tag-search-trace.md](examples/tag-search-trace.md) | 「◯◯タグの顧客を出して」をタグ名から正しく処理する完全トレース | タグ名で絞る依頼を受けたとき |
| [examples/tag-assign-trace.md](examples/tag-assign-trace.md) | 「◯◯を買った人に新しいタグを付けて」を承認ゲート付きで処理する完全トレース | タグの作成・付与・解除の依頼を受けたとき |
