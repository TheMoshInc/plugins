---
name: applicant-reporter
description: MOSH の商品の申込者・サブスク購読者を MCP 経由で確認するスキル。要件ヒアリング → 商品・プランの特定 → 申込者一覧・購読者一覧の取得 → 個人情報の扱いに注意して整形提示、という段階で対話的に進める。「この商品の申込者を見せて」「サブスクの継続中メンバー一覧」「解約した人は誰」「振込待ちの人」「分割払いの決済エラーの人」「サロンの入退場管理用の名簿」「applicant-reporter」など、申込者・購読者の照会リクエストで使用する。「誰が申し込んでる？」のような曖昧な確認でも `postCreatorProductApplicants` / `postCreatorProductSubscribers` を使う場合は本スキルを起動し、提示規約を必ず適用する。
---

# MOSH 申込者レポーター（applicant-reporter）

## Overview

MCP ツール（`postCreatorProductApplicants` / `postCreatorProductSubscribers`。以下ツール名は OpenAPI の operationId で記す。実際のツール名には `mcp__<server>__` の接頭辞が付くが、その形は環境により異なるため本書では付けない）を使って、MOSH クリエイターの商品に申し込んだ人・サブスクリプションを購読している人を確認する。返る内容は MOSH 管理画面の「申込者一覧」ページ（申込管理タブ / サブスク管理タブ）と同じ。

プロダクト上の呼称は「商品」「プラン」「申込者」「購読者」。API のキー名では商品が `productId`、プランが `planId`。ユーザーへの提示ではプロダクト上の呼称を使う。

ツールのパラメータは [references/mcp-tools.md](references/mcp-tools.md)、レスポンスの読み方と status の意味は [references/content-schema.md](references/content-schema.md)、用途別の手順は [references/best-practices.md](references/best-practices.md) を参照する。

## 前提条件: 参照リソース

| 参照 ID | MCP で取得できるか | 取得経路 |
|---|---|---|
| `productId`（商品） | ✅ | `getCreatorProducts`（商品一覧）。返る `id` をそのまま使う |
| `planIds`（プラン） | ✅ | `getCreatorProductPlans`（商品のプラン一覧）。空配列なら商品配下の全プラン |
| 申込者・購読者の名前 / メールアドレス | ✅ | 各ツールの `keyword` で部分一致検索 |

## When to use

- 商品・プランごとの申込者や購読者の名簿を出したい（サロンの入退場管理、参加者の確認）
- 特定の人が申し込んでいるか・購読が続いているかを名前やメールアドレスで確認したい
- 振込待ち・決済エラー・解約済みなど、対応が必要な人を洗い出したい
- 申込日や購読開始日の範囲で絞って一覧を出したい

## When NOT to use

- 売上金額・件数の集計 → `sales-reporter` スキル（このスキルは人の一覧で、金額は返らない）
- 商品・プランの内容そのものの確認 → `product-navigator` スキル
- 申込のキャンセル・返金・購読の解約などの更新（提供していない。参照専用）
- 申込 1 件の詳細（アンケート回答・クーポン・支払い明細）（提供していない。管理画面へ誘導する）

## Workflow

### 1. 要件ヒアリング

- 対象の商品（名前で聞き、`getCreatorProducts` で `id` を解決する。同名が複数あればユーザーに確認する）
- 見たいのは申込者（単発・分割決済・銀行振込）か、購読者（サブスクリプション）か。商品に両方のプランがあれば両方を取る
- 絞り込み条件（プラン / 名前・メールアドレス / 状態 / 期間）と、名簿として全員必要か、件数の確認でよいか

### 2. 取得

- プランで絞るときは `getCreatorProductPlans` で `planIds` を解決する
- 申込者は `postCreatorProductApplicants`、購読者は `postCreatorProductSubscribers` を呼ぶ。リクエストボディは全項目必須で、使わない条件は「絞り込まない」値（配列は `[]`、`keyword` は `""`、日時範囲は `null`）を渡す。値の意味は [references/mcp-tools.md](references/mcp-tools.md)
- 最初は `limit: 20`、`offset: 0` で件数（`totalCount`）を確認する。多いときは件数を伝え、全員出すか条件で絞るかをユーザーに確認してから `offset` で続きを取る
- 申込者一覧で `paymentPattern` に `cardInstallmentsApproved` / `cardInstallmentsCanceled` を含めると `limit` / `offset` が無視され、条件に合う全件が 1 回で返る。先に `planIds` や `approvedDateTime` で対象を絞ってから指定し、この応答では `offset` で続きを取りに行かない

### 3. 整形して提示

- 「ユーザーへの提示・コミュニケーション規約」に従い、状態を日本語の状態名に変換して表で示す。変換表は [references/content-schema.md](references/content-schema.md)
- 申込者と購読者を 1 つの表に混ぜない。用途別の見せ方は [references/best-practices.md](references/best-practices.md)、レスポンスの形は [examples/applicants-response.json](examples/applicants-response.json)

## ユーザーへの提示・コミュニケーション規約

1. **個人情報は依頼された用途にだけ使う。** ゲスト名・メールアドレス・MOSH ID（`guest.id`）が返る。名簿作成や本人確認などユーザーが依頼した用途以外に出力・転記・要約しない。依頼されていないのに全員分のメールアドレスを並べない。ファイルやメッセージに書き出すときは、宛先と用途をユーザーに確認する
2. JSON をそのまま見せない。自然言語の箇条書きか表で要約する
3. HTTP 用語・ステータスコードを文面に出さない
4. 生の内部 ID（`productId` / `planId`）は表示名に変換して出す。変換元は `getCreatorProducts` / `getCreatorProductPlans`。`guest.id` は「MOSH ID」として、ユーザーが求めたときだけ出す
5. `status` は生の値（`installmentActive` など）ではなく日本語の状態名（「分割決済中」）で示す。「決済済み」と「反映待ち」を混同しない
6. 日時は日本時間の年月日（必要なら時分）で示す
7. 取得できないもの（金額・支払い明細・アンケート回答・クーポン）は推測しない。「このツールでは取得できません」と伝え、売上は `sales-reporter`、詳細は管理画面へ誘導する

## 必須ルール

- リクエストボディは全項目必須。省略せず「絞り込まない」値を明示して送る
- 「継続中の購読者」は `statuses: ["active", "pastDue"]`。`pastDue` は決済エラーだが購読は続いている。`trialing` はどのレコードにも付かない
- `paymentMethod` で絞り込みとして効くのは `card` と `bank-transfer` のみ。`ticket` / `cash` を単独で指定しても絞り込まれない。`card` は 0 円の申込を含む
- `totalCount` が多いときに全件を取りに行く前に、ユーザーに確認する
- 参照 ID を推測・捏造しない。商品・プランは一覧ツールで実在確認した ID だけを使う

## よくあるミス

| NG | OK |
|---|---|
| 「継続中」を `statuses: ["active"]` だけで取る | `["active", "pastDue"]`。決済エラー中も購読は続いている |
| サブスクの商品で `postCreatorProductApplicants` だけを呼ぶ | 購読者は `postCreatorProductSubscribers`。両方のプランがある商品は両方呼んで別の表にする |
| `paymentMethod: ["cash"]` で現金の申込だけを出そうとする | 効かないので全件が返る。`ticket` / `cash` は絞り込めないと伝える |
| `cardInstallmentsApproved` を含めた結果に `offset` で続きを取りに行く | その応答は全件。`totalCount` = 返った件数で、続きは無い |
| `status` の値をそのまま見せる | 日本語の状態名に変換する（[references/content-schema.md](references/content-schema.md)） |
| 依頼が「人数を知りたい」なのに全員のメールアドレスを列挙する | 件数だけ答える。名簿は求められたときだけ |

## エラーが返ったとき

商品が見つからない（他のクリエイターの商品や存在しない ID）、指定したプランがその商品配下でない、といった場合は理由を示すエラーが返る。再試行せず、「その商品は見つかりませんでした」「指定したプランはこの商品のものではありません」のように平易な日本語で伝え、商品・プランの特定からやり直す。

## References

- [references/mcp-tools.md](references/mcp-tools.md) — 使用する MCP ツールのパラメータ詳細と「絞り込まない」値
- [references/content-schema.md](references/content-schema.md) — レスポンスの読み方、status と `paymentPattern` の意味、表示名への変換表
- [references/best-practices.md](references/best-practices.md) — 用途別の手順とセルフレビューチェックリスト
- [examples/applicants-response.json](examples/applicants-response.json) — 申込者一覧の代表的なレスポンス例（ダミーデータ）
