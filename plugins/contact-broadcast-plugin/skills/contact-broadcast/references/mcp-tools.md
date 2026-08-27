# コンタクトメッセージ MCP ツールのパラメータ・状態条件

呼び出す直前に、該当する操作の節だけを読む。ツールの選び分けと承認の要否は SKILL.md が正本。

> 本ファイルの制約・上限・状態条件は、MOSH の API スキーマ定義で確認したもの（2026-08 時点）。
> 値を書き換えるときは、記憶や推測ではなく MOSH の API スキーマ定義で裏取りする。

## 新規作成（`postCreatorContactMessagesEmail` / `postCreatorContactMessagesLine`）

- `isDraft`: 予約を確定するなら false、下書きとして保存するなら true。
- `isTrackingEnabled` の**置き場所がメールと LINE で違う**。メールは**ボディ直下に1つ**。
  LINE は**`contents[]` の要素ごと**で、持つのは `text` タイプのみ（`carousel` / `image_carousel` /
  `video` / `rich_message` は持たない）。同意の要否は `tracking-consent.md` を参照。
- `filterConditions`: 配信先の絞り込み条件。null を指定すると全員（LINE は当該公式アカウントの友だち全員）が対象。
  オブジェクトを指定する場合、**メールは `email` / `tag` の 2 キー、LINE は `line` / `tag` / `inflowAction` / `richMenu` の
  4 キーすべてが必須**で、使わない条件には null を指定する（キーの内訳と各条件の意味はスキーマの
  `filterConditions` の説明を参照）。日時範囲は `from` が `to` より後にならないこと。タグ条件は 1〜2 件で、
  2 件指定すると AND（どちらも満たすコンタクトのみ）。
- 配信対象が 0 件でも予約配信は受け付けられる（配信実行時に対象を再評価するため）。配信可能数の上限を超える場合は 403。
- **下書き（`isDraft: true`）の作成では上限確認を含む一部の検証がスキップされる。** 下書きの作成に成功しても、
  あとで予約を確定する際（patch で `isDraft: false`）に 400 / 403 になることがある。
- `scheduledAt` は**必須項目だが null 可**で、要求される値が `isDraft` によって変わる。
  - **`isDraft: true`（下書き）**: `scheduledAt: null` でよい（配信スケジュールの検証がスキップされる）。
    日時が決まっていない下書きに日時を入れる必要はない。
  - **`isDraft: false`（予約確定）**: タイムゾーンオフセット付きの日時が必須（例 `2026-09-05T20:00:00+09:00`）。
    オフセットを省略すると意図しない時刻に配信される。現在時刻から 10 分以上後、かつ 1 ヶ月以内
    （管理画面と同じ条件）。
  - **`isDraft: false` + `scheduledAt: null` は即時配信**で取り消せないため行わない（SKILL.md の判定表）。

## 送信元 LINE公式アカウントの確定（`getCreatorLineChannels`）

LINE 配信の `lineChannelId` は、このツールで取得した `channels[]` から選ぶ（推測で埋めない）。

- 候補は `isConnected: true` のものだけ。`false` は連携が切れているため配信に使わない。
  使えるものが無ければ、管理画面で LINE 連携を確認してもらう。
- **ユーザーに提示する名前は `displayName`。空文字のことがあるので、空なら `officialLineAccountId`
  （`@` から始まる LINE ID）で示す。** ID を裸で出すのはこの場合だけ。
- `officialLineAccountId` は**重複しうる**（同じ公式アカウントを別 `id` で複数保持している状態がある）。
  `displayName` が同名の候補が複数あるときは `officialLineAccountId` を併記し、それでも区別できなければ
  どれを使うかユーザーに確認する。
- 絞り込んだ結果が1件ならそれを採用してよい。2件以上ある場合は SKILL.md の Step 3 で承認を求める前に選んでもらう。

## 編集（`patchCreatorContactMessagesEmail` / `patchCreatorContactMessagesLine`）

- **編集できるのは下書きのみ。** 配信予定・配信済みに実行すると 400（配信待ち状態または配信済みのメッセージは
  編集できません）になる。
- 下書きに対して行える操作は 2 つ: 下書きのまま保存（`isDraft: true`）/ 配信予定として確定
  （`isDraft: false` + `scheduledAt` に未来の日時）。
- **配信予定のメッセージを直したい場合は、先に `post〜Draft` で下書きに戻し、編集してから改めて配信予定にする。**
  下書きに戻せるのは配信予定時刻の 5 分前まで。配信済みは編集も差し戻しもできない。
- **リクエストボディは全項目必須で、部分更新はできない。** 変更しない項目も現在の値を送る。
  一覧（`post〜Search`）のレスポンスには本文・配信対象（メール: `title` / `body` / `filterConditions`、
  LINE: `contents` / `filterConditions` / `lineChannelId`）が**含まれない**ため、
  **必ず `get〜` で現在の内容を取得してから編集する**。一覧の値だけで組み立てると本文と配信対象が失われる。

## 予約解除（`postCreatorContactMessagesEmailDraft` / `postCreatorContactMessagesLineDraft`）

- 配信予定（未送信）のメッセージを、内容・配信先の条件を保持したまま下書きに戻す。配信スケジュールは解除され、
  日時は未設定に戻る。
- 下書き・配信済みは対象外。配信予定時刻の 5 分前を過ぎたメッセージは戻せない。戻せる対象がない場合は 404。
- 内容や配信日時を修正して配信予定に戻す場合は、このツールで下書きに戻したあと patch で編集し、
  `isDraft: false`・`scheduledAt` に未来の日時を指定する。

## 削除（`deleteCreatorContactMessagesEmail` / `deleteCreatorContactMessagesLine`）

- 削除できるのは未配信のみ。配信予定は配信予定時刻の 5 分前まで、下書きは時間制限なく削除できる。
  配信済みは削除できない。
- `ids` は配列。指定 ID のうち削除可能なもののみが削除され、削除可能な対象が 1 件もない場合は 404。

## LINE の `contents` の項目別制約

素材の扱い（外部URLを使わない）は SKILL.md の「LINE の素材制約」が正本。ここは項目ごとの上限・形式のみ。

- `contents` は本文の配列。type ごとに指定項目が異なる（詳細はスキーマの各項目の説明を参照）。
- `text`: 本文は 4000 文字以内。**API の上限は 5000 だが、管理画面に合わせて 4000 で運用する**
  （呼び出す側が守るルール。5000 に緩めない）。埋め込み変数 `{{line_name}}` が効くのは**この type のみ**
  （ルールは SKILL.md の「埋め込み変数」が正本）。
- `image_carousel` の `imageCarousels` / `carousel` の `carousels` はいずれも **1〜4 件**。
  `altText`（`image_carousel` / `carousel` / `rich_message`）は **50 文字以内**。
- `image_carousel` / `carousel` / `rich_message`: `imageUrl` には MOSH の画像リポジトリの URL を指定する
  （末尾が画像 ID になる形式。ドメインは環境ごとに異なる）。`linkUrl` / `buttonUrl`（タップ先）は外部URLでよい。
  素材の扱いのルールは SKILL.md の「LINE の素材制約」が正本。
- `video`: `muxAssetId` は MOSH にアップロード・トランスコード済みの動画に採番される ID、
  `previewMoshImageId` は同じく MOSH にアップロード済みのプレビュー画像の ID。どちらも MCP から新規に作成できない。
- `rich_message`: `cells` の件数を `splitPattern` の分割数と一致させる
  （`ONE_BLOCK`=1 / `LEFT_RIGHT_SPLIT`・`TOP_BOTTOM_SPLIT`=2 / `GRID_FOUR`=4）。
  `cells` は 1〜4 件で分割数に対応。**`cells[]` の各マスは `url`（タップ先）と `postbackActions`
  （タップ時の顧客タグ付与・メッセージ送信。1〜3 件、使わないなら null）を独立に指定でき、
  「URL単独 / アクション単独 / 併用 / 両方なし=タップ不可」の4パターンが取れる**。
  飛び先の数が分割数に足りないときは、余るマスを `url: null` / `postbackActions: null`（タップ不可）にする。
  画像は `imageWidth` / `imageHeight` ともに 300〜1024px。

## 一覧・詳細

- 一覧（`post〜Search`）: `limit` / `offset` / `sort` / `keyword`（配信タスク名・件名・本文の部分一致）。
  **本文・配信対象・送信元の公式アカウント（`lineChannelId`）は返らない**（編集前に `get〜` が必要な理由）。
- 詳細（`get〜`）: 全項目を返す。編集はここを起点にする。
