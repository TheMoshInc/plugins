# サービス／感想レポートの表示設定（ツール仕様）

サービス／感想レポートの取得（`getCreatorProfileLinkServices` / `getCreatorProfileLinkReports`）と表示設定の更新・並べ替えのツールがある。

**取得ツールがカタログに無い環境では、現在値を読めない。** その場合は SKILL.md のステップ2（ユーザーに聞く／聞けなければ実行しない）に従う。

## 一覧取得

### `getCreatorProfileLinkServices`
クエリ: `limit`（1〜100。**省略すると全件**）/ `offset`。返り値: `services[]`、`totalCount`。各要素の読むところ:
- `service`（サービス本体。`id` と名前など）
- `profileLinkSettings`: `isVisible` / `displayOrder` / `cardSize`（`SMALL`/`MEDIUM`/`LARGE`）← **patch に渡す現在値**
- `hasActivePlans` / `minPlanPaymentInterval` / `address` / `nextSchedule` / `productSlug`: 公開表示に影響する読み取り専用の情報（編集不可）

設定行がまだ無いサービスは、読み取り側では「表示」扱いとして返る。

### `getCreatorProfileLinkReports`
同じクエリ。返り値: `reports[]`、`totalCount`。各要素: `report`（`reportId` / 投稿者名 / スコア / 本文 / `serviceName` など。**中身は編集不可**）、`profileLinkSettings`（`isVisible` / `displayOrder`）。

設定行がまだ無いレポートは、**非表示（`isVisible: false`）として返る**（サービスは表示扱い。逆なので注意）。

## `patchCreatorProfileLinkService`（全置換）

- パス: `id`（サービスID。一覧の `service.id`）
- ボディ（3項目すべて必須）: `isVisible`（boolean）/ `displayOrder`（integer・**0始まり**の絶対値）/ `cardSize`（`SMALL` | `MEDIUM` | `LARGE`）
- 表示/非表示だけ変えたいときも、`displayOrder` と `cardSize` は**現在値**を渡す
- レスポンス: `{ message }`（200）／401

## `patchCreatorProfileLinkReport`（全置換）

- パス: `id`（感想レポートID。一覧の `report.reportId`）
- ボディ（2項目すべて必須）: `isVisible` / `displayOrder`（0始まり）
- **`cardSize` は無い**（感想レポートにサイズ概念は無い。SKILL.md 必須ルールC）
- レスポンス: `{ message }`（200）／401

## `putCreatorProfileLinkServicesDisplayOrder`（対象のみ更新）

- ボディ: `{ items: Array<{ serviceId; displayOrder }> }`（`displayOrder` は0始まりの絶対値）
- `items` に含めたサービスだけ更新する。含めなかったサービスの順序は変わらない（ギャラリーなどの put と違い、消えない）
- 設定行が無い対象を含めると新規作成され、**`isVisible` は `true` で確定する**（SKILL.md 必須ルールI。設定なしは表示扱いで返り、並べ替えで true として保存される。設定が無いサービスの並べ替えは実機未確認）
- サーバー側は `serviceId` の所有・存在を検証せず保存する（実機確認: 存在しない ID でも成功し、一覧には出ない設定行が残る）。一覧の結果の ID だけを使う
- 並べ替えは、**取得ツールがある環境では全件の現在値を見て、希望順の全体を組む**。取得ツールが無い環境で1件だけ動かす場合は、SKILL.md 必須ルールBの1件移動の手順に従う（部分的な数字の指定は、他の対象と順序が衝突し得る）

## `putCreatorProfileLinkReportsDisplayOrder`（対象のみ更新）

- ボディ: `{ items: Array<{ reportId; displayOrder }> }`（0始まり）
- 含めたものだけ更新する点はサービス側と同じ。**ただし設定行が無いレポートを含めると、`isVisible` は `false`（非表示）で作られる**（サービスは `true`）。並べ替えだけではそのレポートは公開ページに出ない。表示したいときは `patchCreatorProfileLinkReport` で `isVisible: true` を明示する
- 設定行が無いレポートは、取得側でも非表示として返る（実機未確認）
- `reportId` の所有・存在の検証は実機未確認（サービス側では検証されないことを実機で確認済み）。一覧の結果の ID だけを使う

## MCP に無いもの

| 欲しい操作 | 有無 | 備考 |
|---|---|---|
| サービス・感想レポートの新規作成 | 無い | サービスは商品/サービス設定側で用意する（SKILL.md 必須ルールF） |
| サービス・感想レポートの削除 | 無い | 感想レポートは `isVisible: false` で代替 |
| 感想レポートの中身（投稿者・スコア・本文）の編集 | 無い | 顧客の投稿であり編集対象外 |

## エラーの読み方

所有・存在の検証が無い点は、サービス／レポートの各 put の節（上）を参照。ID は一覧の結果だけを使う。

公開されているレスポンス定義は 200 と 401 のみ。必須項目が欠けたボディや、`cardSize` が `SMALL`/`MEDIUM`/`LARGE` 以外の値は、クライアント側のスキーマ検証でサーバーに届く前に弾かれる。
