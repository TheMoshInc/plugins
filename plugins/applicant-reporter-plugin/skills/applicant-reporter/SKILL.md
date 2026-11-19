---
name: applicant-reporter
description: MOSH の商品の申込者・サブスク購読者を MCP 経由で確認するスキル。要件ヒアリング → 商品・プランの特定 → 申込者一覧・購読者一覧の取得 → 個人情報の扱いに注意して整形提示、という段階で対話的に進める。「この商品の申込者を見せて」「サブスクの継続中メンバー一覧」「解約した人は誰」「振込待ちの人」「分割払いの決済エラーの人」「サロンの入退場管理用の名簿」「◯◯さんの申込の支払い状況・アンケート回答・使ったクーポンを見せて」「次回の分割引き落とし日はいつ」「applicant-reporter」など、申込者・購読者の照会リクエストで使用する。「誰が申し込んでる？」のような曖昧な確認でも `postCreatorProductApplicants` / `postCreatorProductSubscribers` / `getCreatorProductApplicant` / `getCreatorProductSubscriber` を使う場合は本スキルを起動し、提示規約を必ず適用する。
---

# MOSH 申込者レポーター（applicant-reporter）

## Overview

MCP ツール（一覧の `postCreatorProductApplicants` / `postCreatorProductSubscribers`、1 件の詳細の `getCreatorProductApplicant` / `getCreatorProductSubscriber`。以下ツール名は OpenAPI の operationId で記す。実際のツール名には `mcp__<server>__` の接頭辞が付くが、その形は環境により異なるため本書では付けない）を使って、MOSH クリエイターの商品に申し込んだ人・サブスクリプションを購読している人を確認する。返る内容は MOSH 管理画面の「申込者一覧」ページ（申込管理タブ / サブスク管理タブ）と、そこから開く申込・サブスクの詳細と同じ。

プロダクト上の呼称は「商品」「プラン」「申込者」「購読者」。API のキー名では商品が `productId`、プランが `planId`。ユーザーへの提示ではプロダクト上の呼称を使う。

ツールのパラメータは [references/mcp-tools.md](references/mcp-tools.md)、レスポンスの読み方と status の意味は [references/content-schema.md](references/content-schema.md)、用途別の手順は [references/best-practices.md](references/best-practices.md) を参照する。

## 前提条件: 参照リソース

| 参照 ID | MCP で取得できるか | 取得経路 |
|---|---|---|
| `productId`（商品） | ✅ | `getCreatorProducts`（商品一覧）。返る `id` をそのまま使う |
| `planIds`（プラン） | ✅ | `getCreatorProductPlans`（商品のプラン一覧）。空配列なら商品配下の全プラン |
| 申込者・購読者の名前 / メールアドレス | ✅ | 各ツールの `keyword` で検索（名前は前方一致、メールアドレスは部分一致） |
| `applicantId` / `subscriberId`（1 件の詳細） | ✅ | 一覧ツールが返す `applicants[].id` / `subscribers[].id` |

## When to use

- 商品・プランごとの申込者や購読者の名簿を出したい（サロンの入退場管理、参加者の確認）
- 特定の人が申し込んでいるか・購読が続いているかを名前やメールアドレスで確認したい
- 振込待ち・決済エラー・解約済みなど、対応が必要な人を洗い出したい
- 申込日や購読開始日の範囲で絞って一覧を出したい
- 特定の人の申込・購読の中身（支払い明細・分割の進み具合と次回決済日・クーポン・アンケート回答・プラン価格）を確認したい

## When NOT to use

- 売上金額の集計（期間・商品全体の合計や推移） → `sales-reporter` スキル（このスキルが返す金額は 1 件ずつの明細で、集計には使わない）
- 商品・プランの内容そのものの確認 → `product-navigator` スキル
- 引き継ぎ先のスキルはそれぞれインストールが必要。このセッションに無ければインストールを案内し、管理画面での確認も代替として示す
- 申込のキャンセル・返金・購読の解約などの更新（提供していない。参照専用。管理画面へ誘導する）

## Workflow

### 1. 要件ヒアリング

- 対象の商品（名前で聞き、`getCreatorProducts` で `id` を解決する。同名が複数あればユーザーに確認する）
- 見たいのは申込者（単発・分割決済・銀行振込）か、購読者（サブスクリプション）か。商品に両方のプランがあれば両方を取る
- 絞り込み条件（プラン / 名前・メールアドレス / 状態 / 期間）と、名簿として全員必要か、件数の確認でよいか
- 特定の人の申込・購読の中身（支払い・クーポン・アンケート回答）まで見たいか

### 2. 取得

- プランで絞るときは `getCreatorProductPlans` で `planIds` を解決する
- 申込者は `postCreatorProductApplicants`、購読者は `postCreatorProductSubscribers` を呼ぶ。リクエストボディは全項目必須で、使わない条件は「絞り込まない」値（配列は `[]`、`keyword` は `""`、日時範囲は `null`）を渡す。値の意味は [references/mcp-tools.md](references/mcp-tools.md)
- 最初は `limit: 20`、`offset: 0` で件数（`totalCount`）を確認する。多いときは件数を伝え、全員出すか条件で絞るかをユーザーに確認してから `offset` で続きを取る
- 申込者一覧で `paymentPattern` に `cardInstallmentsApproved` / `cardInstallmentsCanceled` を含めると `limit` / `offset` が無視され、条件に合う全件が 1 回で返る。先に `planIds` や `approvedDateTime` で対象を絞ってから指定し、この応答では `offset` で続きを取りに行かない
- 1 件の中身が必要なときは、一覧で対象を特定してから、申込は `getCreatorProductApplicant`（`applicantId` = `applicants[].id`）、購読は `getCreatorProductSubscriber`（`subscriberId` = `subscribers[].id`）を呼ぶ。1 件ずつの呼び出しなので、一覧の全員分を詳細で取りに行かない（多人数の金額は `sales-reporter`）

### 3. 整形して提示

- 「ユーザーへの提示・コミュニケーション規約」に従い、状態を日本語の状態名に変換して表で示す。変換表は [references/content-schema.md](references/content-schema.md)
- 申込者と購読者を 1 つの表に混ぜない。用途別の見せ方は [references/best-practices.md](references/best-practices.md)。一覧のレスポンスの形は [examples/applicants-response.json](examples/applicants-response.json)、1 件の詳細を読むときは [examples/applicant-detail-response.json](examples/applicant-detail-response.json) を参照する

## ユーザーへの提示・コミュニケーション規約

1. **個人情報は依頼された用途にだけ使う。** ゲスト名・メールアドレス・MOSH ID（`guest.id`）が返る。詳細ツールではさらにアンケート回答（`surveyAnswers`。電話番号・性別・住所などを含む場合がある）とゲストのメッセージが返る。名簿作成や本人確認などユーザーが依頼した用途以外に出力・転記・要約しない。一覧・名簿を依頼されたら氏名は出す（用途の確認で止めない）。メールアドレス・MOSH ID は依頼されたとき、または名簿の用途に必要なときだけ出す。アンケート回答は求められた項目だけを出す。ファイルやメッセージに書き出すときは、宛先と用途をユーザーに確認する
2. JSON をそのまま見せない。自然言語の箇条書きか表で要約する
3. HTTP 用語・ステータスコードを文面に出さない
4. 生の内部 ID（`productId` / `planId` / 申込 ID / 購読 ID / 注文 ID / クーポン ID）は表示名に変換するか出さない。変換元は `getCreatorProducts` / `getCreatorProductPlans`。`guest.id` は「MOSH ID」として、ユーザーが求めたときだけ出す
5. `status` は生の値（`installmentActive` など）ではなく日本語の状態名（「分割決済中」）で示す。絞り込みに使った値（`paymentPattern` の `cardInstallmentsCanceled` など）も英字のまま出さず、「分割（一括入金型）のキャンセル」のように言い換える。「決済済み」と「反映待ち」を混同しない
6. 日時は日本時間の年月日（必要なら時分）で示す
7. 金額は円単位で示し、クーポンを使った申込はクーポン適用前の金額（`preCouponAmount`）と支払った金額を区別する。`preCouponAmount` が `null`（分割（分割入金型）など）なら適用前の金額は示さない。取得できないもの（サブスクの過去の請求履歴・返金可否・ゲストのプロフィール項目）は推測しない。「このツールでは取得できません」と伝え、売上の集計は `sales-reporter`、それ以外は管理画面へ誘導する

## 必須ルール

- リクエストボディは全項目必須。省略せず「絞り込まない」値を明示して送る
- 「継続中の購読者」は `statuses: ["active", "pastDue"]`。`pastDue` は決済エラーだが購読は続いている。`trialing` はどのレコードにも付かない
- `paymentMethod` で絞り込みとして効くのは `card` と `bankTransfer` のみ。`ticket` / `cash` を単独で指定しても絞り込まれない。現金・チケットの申込は、絞り込まずに取得した結果の `order.paymentMethod` で判別する（手順は [references/best-practices.md](references/best-practices.md) の「7. 現金・チケットで支払った人を出す」）。`card` で絞ると 0 円の申込も含まれる
- 購読者一覧の `planIds` に商品配下でない ID を渡すとエラーにならず 0 件が返る。0 件のときは「購読者がいない」と断定せず、プランが実在するかを `getCreatorProductPlans` で確かめる
- `totalCount` が多いときに全件を取りに行く前に、ユーザーに確認する
- 件数の内訳（プラン別・状態別）を、取得した 1 ページ分から推定しない。内訳が必要なら `planIds` / `paymentPattern` / `statuses` で絞った呼び出しの `totalCount` を使う。取らないなら合計件数だけを答える
- 参照 ID を推測・捏造しない。商品・プランは一覧ツールで実在確認した ID だけを使う

## よくあるミス

他の節でカバーできていない落とし穴だけを載せる。追記する前に、既存の節を強化して済まないか確認する。

| NG | OK |
|---|---|
| 「継続中」を `statuses: ["active"]` だけで取る | `["active", "pastDue"]`。決済エラー中も購読は続いている |
| サブスクの商品で `postCreatorProductApplicants` だけを呼ぶ | 購読者は `postCreatorProductSubscribers`。両方のプランがある商品は両方呼んで別の表にする |
| `cardInstallmentsApproved` を含めた結果に `offset` で続きを取りに行く | その応答は全件。`totalCount` = 返った件数で、続きは無い |
| `status` の値をそのまま見せる | 日本語の状態名に変換する（[references/content-schema.md](references/content-schema.md)） |
| 「支払い状況・クーポン・アンケートは取れない」と答える | 一覧で対象を特定し、`getCreatorProductApplicant` / `getCreatorProductSubscriber` で取る |

## エラーが返ったとき

商品が見つからない（他のクリエイターの商品や存在しない ID）、申込者一覧で指定したプランがその商品配下でない、詳細ツールで申込・購読が見つからない（別の商品の ID を渡した場合を含む）、といった場合は理由を示すエラーが返る。再試行せず、「その商品は見つかりませんでした」「指定したプランはこの商品のものではありません」のように平易な日本語で伝え、商品・プランの特定からやり直す。

## References

- [references/mcp-tools.md](references/mcp-tools.md) — 使用する MCP ツールのパラメータ詳細と「絞り込まない」値
- [references/content-schema.md](references/content-schema.md) — レスポンスの読み方、status と `paymentPattern` の意味、表示名への変換表
- [references/best-practices.md](references/best-practices.md) — 用途別の手順とセルフレビューチェックリスト
- [examples/applicants-response.json](examples/applicants-response.json) — 申込者一覧の代表的なレスポンス例（ダミーデータ）
- [examples/applicant-detail-response.json](examples/applicant-detail-response.json) — 申込 1 件の詳細のレスポンス例（ダミーデータ）
