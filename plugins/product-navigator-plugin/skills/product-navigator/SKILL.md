---
name: product-navigator
description: MOSH の商品と商品プランを MCP 経由で参照するスキル。要件ヒアリング → 商品一覧・詳細の取得 → プラン一覧・詳細の取得 → わかりやすく整形して提示、という流れで対話的に進める。「商品一覧を見せて」「商品の内容を確認したい」「プランの価格を教えて」「この商品のプランを一覧して」「この商品の今月の販売状況は？」「この商品の今月分の決済金額は？（管理画面の販売状況）」「product-navigator」など、MOSH の商品・プランの参照リクエストで使用する。「商品どれだったっけ」のような曖昧な確認でも `getCreatorProducts` / `getCreatorProduct` / `getCreatorProductPlans` / `getCreatorProductPlan` / `getCreatorProductSalesStatus` を使う場合は本スキルを起動し、提示ルールを必ず適用する。期間を指定した売上・購入者数は sales-reporter、申込者・購読者の名簿は applicant-reporter、商品・プランの作成・変更やクーポン・購入後設定は product-builder の担当。
---

# MOSH 商品ナビゲーター

## Overview

MCP ツール（`getCreatorProducts` / `getCreatorProduct` / `getCreatorProductPlans` / `getCreatorProductPlan` / `getCreatorProductSalesStatus` の 5 つ。以下ツール名は OpenAPI の operationId で記す。実際のツール名にはサーバー識別子の接頭辞（例: `mcp__<server>__getCreatorProducts`）が付くが、その形は環境により異なるため本書では付けない）を使って、MOSH クリエイターの商品と商品プラン（価格・課金サイクル・定員等）、および商品ごとの今月の販売状況（管理画面の商品詳細に出る「決済金額」「申し込み者」「サブスク購読者」）を参照する。

**参照専用**である。商品・プランの作成・変更・削除や、期間を指定した売上・申込者や購読者の一覧も MCP で可能だが、本スキルでは扱わず別スキルに引き継ぐ（引き継ぎ先のスキルは「When NOT to use」が正）。

レスポンスの構造・各項目の意味は [references/content-schema.md](references/content-schema.md) を参照してから提示を組み立てる。

## When to use

- 商品の一覧・内容（説明、注意事項、公開状態、公開 URL 等）を確認したいとき
- 商品に紐づくプランの内容（価格、課金サイクル、定員・残り枠、支払い方法、販売状態、申込期間等）を確認したいとき
- 商品ごとの今月の販売状況（今月の決済金額・今月の申し込み者数・現時点のサブスク購読者数）を確認したいとき
- 上記 5 ツール（`getCreatorProducts` / `getCreatorProduct` / `getCreatorProductPlans` / `getCreatorProductPlan` / `getCreatorProductSalesStatus`）が必要な文脈

## When NOT to use

- 商品・プランの作成・編集・複製・並べ替え・公開設定の変更・削除、クーポン・購入後設定（サンクスページ・自動リダイレクト・感想レポート）・提供コンテンツの確認 → `product-builder` スキル（要インストール）
- 期間を指定した売上（商品別・プラン別・日別・月別の金額、ユニーク購入者数）→ `sales-reporter` スキル（要インストール）
- 申込者・サブスク購読者の一覧・詳細（誰が申し込んだか）→ `applicant-reporter` スキル（要インストール）
- **人数の問い合わせの振り分け**: 1 商品の「今月分の申し込み者数（買い切りプランのみ）」「現時点のサブスク購読者数」＝本スキル（サブスクの新規申込数は含まないため `sales-reporter` で期間を今月に指定）／期間指定・全商品横断の購入者数＝`sales-reporter`／誰が（名簿）＝`applicant-reporter`
- クリエイター全体の新規申込人数の推移・当月値（`getCreatorProductsApplications` / `getCreatorProductsApplicationsSummary`）は本スキルでは扱わない
- 引き継ぎ先のスキルが未インストールなら、そのスキルのインストールを案内する（管理画面での操作を代替として示してもよい。ただし変更できないと確認した操作は管理画面でも案内しない）
- LP（ランディングページ）の参照・編集 → `lp-builder` スキル
- ワークフロー（ステップ配信）→ `workflow-builder` スキル

## Workflow

### 1. 要件ヒアリング

ユーザーに以下を確認する（自明なら聞き直さない）:

- 知りたいのは商品そのものか、プラン（価格・定員等）か
- 対象は全商品か、特定の商品か（商品名・キーワードで良い。ID をユーザーに求めない）
- 絞り込み（公開状態など）や知りたい項目

### 2. 商品の取得

1. `getCreatorProducts` で一覧を取得する。`status` で公開状態の絞り込みができる
2. 件数が多い場合はページングする（`limit` 最大 100・既定 20）。`totalCount` が取得済み件数より多ければ `offset` をずらして続きを取得する
3. 特定の商品が対象なら、一覧から名前で特定し、その `id` で `getCreatorProduct` を呼ぶ。ユーザーが口にした商品名と完全一致しない場合は、候補を提示して選んでもらう（勝手に決めない）

### 3. プランの取得

1. 商品の `id` を `productId` として `getCreatorProductPlans` を**1 回だけ**呼ぶ。**プランについての回答は、この 1 回の応答だけで組み立てる**（価格・課金サイクル・定員・販売状態・申込期間・会員サイトなど、プラン詳細と同じ全項目が全プラン分入っている）
2. このプラン一覧は**ページングされない**（`limit` / `offset` は現状無視され、`totalCount` は返した件数）。続きの取得は不要
3. 自分の商品でない・存在しない `productId` でもエラーにならず**空配列**が返る。0 件のときは「プランが無い」と断定せず、商品の特定が正しいかを先に確認する
4. 一覧を取ったあとに、確認のためにプラン詳細（`getCreatorProductPlan`）を 1 件ずつ呼ばない。一覧を取り直さない。プラン詳細は、ユーザーが 1 つのプランだけを指定して聞いていて、まだ一覧を取っていないときにだけ使う

### 4. 販売状況の取得（聞かれたときだけ）

1. 商品の `id` で `getCreatorProductSalesStatus` を呼ぶ（全プラン合算・今月分。期間は指定できない）
2. 表示する項目はプランの構成で決める（管理画面と同じ）。申し込み者は**買い切りプランがある商品だけ**、サブスク購読者は**サブスクプランがある商品だけ**出す。プランの構成は `getCreatorProductPlans` の `billingCycle` で判断する
3. 値はそのまま提示し、再計算・合算しない。数え方と sales-reporter との違いは [references/content-schema.md](references/content-schema.md) の「販売状況」節を読んでから添える

### 5. 整形して提示

「ユーザーへの提示・コミュニケーション規約」に従い、[references/best-practices.md](references/best-practices.md) の指針で要約して提示する。提示前に同ファイル末尾の「セルフレビューチェックリスト」を確認する。

## ユーザーへの提示・コミュニケーション規約

1. JSON をそのまま見せない。表または箇条書きに整形して提示する（ユーザーが明示的に求めた時だけ JSON を出す。ただし生の内部 ID・Tiptap JSON・enum の英字値は、明示的に求められても出さない）
2. HTTP 用語・ステータスコードを文面に出さない。「取得できませんでした。◯◯が原因です」のように平易な日本語で伝える
3. 生の内部 ID・識別子（一覧は [references/content-schema.md](references/content-schema.md) の「絶対に避けること」）を提示に出さない。商品は商品名、プランはプラン名で示す
4. enum 値は日本語に変換して出す（変換表は [references/content-schema.md](references/content-schema.md)）。`PUBLIC` 等の英字をそのまま見せない
5. `description`（商品説明）に加え `precautions` / `cancellationPolicy` もリッチテキストの内部形式（Tiptap JSON）で返ることがある。内部形式のまま見せず、本文テキストを取り出して要約する

## 必須ルール

- 商品・プランの `id` は必ず一覧・詳細ツールの結果から使う。推測・捏造しない
- 「0 件」「存在しない」と断定する前に、取得経路（`productId` の特定・`status` 絞り込みの有無）を確認する
- 更新系の依頼（「公開にして」「価格を変えて」）には、このスキルでは行えない旨を伝え、「When NOT to use」の引き継ぎ先へ渡す
- プランの名前・価格・課金開始日の変更依頼で、対象の商品が分かる場合は、引き継ぐ前に `getCreatorProductPlans` で対象プランを取得し、`isNameEditable` / `isPriceEditable` / `isBillingStartEditable` を確認する。`false` なら「このプランは今の状態では価格を変えられません（サブスクは非公開かつ購読者がいないときだけ変更できる）」のように添えて引き継ぎ、「引き継ぎ先のスキルを入れれば変更できる」「管理画面から変更できる」とは案内しない
- 販売状況（「4. 販売状況の取得」）で答えられない売上・申込者・購読者の依頼は、推測で答えず「When NOT to use」の引き継ぎ先へ渡す

## よくあるミス

| NG | OK |
|---|---|
| プラン一覧を `limit` / `offset` でページングし続ける | プラン一覧はページングされない。1 回で全件返る |
| プラン一覧を取ったあと、全プランの詳細を 1 件ずつ取って確かめる | 一覧の応答に詳細と同じ全項目がある。一覧 1 回の応答だけで答える |
| プラン一覧が空 → 「商品が存在しません」と断定 | 空配列は「所有していない/存在しない `productId`」でも返る。商品の特定から見直す |
| 販売状況の「決済金額」を「今月の売上」として売上一覧と比べる | 数え方が違う（詳細は content-schema の「販売状況」節） |
| `publicUrl` が `null` → 「URL の取得に失敗」 | 非公開（`PRIVATE`）の商品では公開ページが存在しないため `null` が正常。旧サービス扱い（`isLegacyService: true`）の商品はサービスの公開状態で決まる（詳細は content-schema の `publicUrl` 行） |
| `description` の JSON をそのまま表示 | 本文テキストを取り出して要約する |
| 商品一覧の `totalCount` とプラン一覧の `totalCount` を同じ意味で扱う | 商品一覧は「総件数」、プラン一覧は「返した件数」 |

## エラーが返ったとき

- **見つからない**: 他のユーザーの商品・プランや存在しない ID を指定すると返る（商品詳細・プラン詳細・販売状況）。再試行せず、対象の特定からやり直すか、ユーザーに商品名を確認する
- **権限で取れない（販売状況のみ）**: メンバー（委任）として操作しているアカウントでは、販売状況を取得できないことがある。再試行せず「この商品の販売状況は、今のアカウントの権限では確認できません」と伝え、商品・プランの参照は続ける
- **認証エラー**: MOSH との接続（API トークンの設定）を見直すよう案内する

## References

- [references/mcp-tools.md](references/mcp-tools.md) — 使用する MCP ツールのパラメータ詳細
- [references/content-schema.md](references/content-schema.md) — レスポンス構造・enum の日本語変換表・提示してはいけない項目
- [references/best-practices.md](references/best-practices.md) — 提示の組み立て方とセルフレビューチェックリスト
