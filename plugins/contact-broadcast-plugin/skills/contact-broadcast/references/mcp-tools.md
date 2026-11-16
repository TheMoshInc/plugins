# コンタクトメッセージ MCP ツールのパラメータ・状態条件

呼び出す直前に、該当する操作の節だけを読む。ツールの選び分けと承認の要否は SKILL.md が正本。

> 本ファイルの制約・上限・状態条件は、MOSH の API スキーマ定義で確認したもの（2026-11 時点）。
> 値を書き換えるときは、記憶や推測ではなく MOSH の API スキーマ定義で裏取りする。

## 新規作成（`postCreatorContactMessagesEmail` / `postCreatorContactMessagesLine`）

- `isDraft`: 予約を確定するなら false、下書きとして保存するなら true。
- `isTrackingEnabled` の**置き場所がメールと LINE で違う**。メールは**ボディ直下に1つ**。
  LINE は**`contents[]` の要素ごと**で、持つのは `text` タイプのみ（`carousel` / `image_carousel` /
  `video` / `rich_message` は持たない）。同意の要否は `tracking-consent.md` を参照。
- `filterConditions`: 配信先の絞り込み条件。null を指定すると全員（LINE は当該公式アカウントの友だち全員）が対象。
  オブジェクトを指定する場合、**メールは `email` / `tag` の 2 キー、LINE は `line` / `tag` / `inflowAction` / `richMenu` の
  4 キーすべてが必須**で、使わない条件には null を指定する（キーの内訳と各条件の意味はスキーマの
  `filterConditions` の説明を参照）。日時範囲は `from` が `to` より後にならないこと。タグ条件（`tag.id[]`）は
  **1〜5 件**で、複数指定すると AND（すべてを満たすコンタクトのみ。OR にはできない）。各条件の `operator` は
  `eq`（そのタグが付いている）/ `ne`（付いていない）。
- 配信対象が 0 件でも予約配信は受け付けられる（配信実行時に対象を再評価するため）。配信可能数の上限を超える場合は 403。
  件数は承認前に「配信対象の件数の目安」節の手順で確認する。
- 同じタグを `tag.id[]` に重複して入れると、**下書き保存でも 400**。
- 409（「ただいま混み合っています」）は、保存と絞り込み条件のタグ削除・他の保存が競合したときに返る。
  内容の誤りではないので、時間をおいてからもう一度試すようユーザーに伝える（すぐに自動で再試行しない）。

## 配信対象の件数の目安（`postCreatorContactsLineSearch` / `postCreatorContactsEmailSearch`）

承認前に、配信対象の条件に今いくつ当てはまるかを確認する。ツール定義も「メッセージ配信の前に、送信対象が何件いるかを確認する用途」を明記している。
**宛先が全員（`filterConditions: null`）でも省かない**（全員が何人かを示すこと自体が承認の材料になり、0件ならそもそも送る相手がいない）。

- 配信の `filterConditions` を、検索の `filters` に写して呼ぶ。`limit: 1` / `offset: 0` / `keyword: ""` でよく、見るのは `filteredCount` だけ。
  - **LINE**: `lineChannelId` に送信元アカウントを指定する。`filters` は `contact` / `line` / `tag` / `inflowAction` / `richMenu` の5キーが必須。`contact: null`（有効なコンタクトのみが既定）にし、残り4キーは配信の `filterConditions` と同じ値を入れる（`filterConditions: null` なら4キーとも null）
  - **メール**: `filters` は `contact` / `email` / `tag` の3キーが必須。`contact: null` にし、`email` / `tag` には配信の `filterConditions` と同じ値を入れる
- 返る `contacts[]` には表示名やメールアドレスが含まれる。**件数以外は出さない**。
- 件数は**目安**として伝える。配信実行時に対象は再評価され、ブロック・購読解除されたコンタクトの扱いも配信時の判定に従うので、「◯人に届きます」とは断定しない（例「今の条件に当てはまるのは約◯人です」）。
- `filteredCount: 0` のとき:
  - `totalCount` も 0 なら、送れる相手がそもそもいない。コンタクトの追加が必要なことを伝える
  - そうでなければ、条件で全員が外れている。予約はできるが、条件の見直しを促す（管理画面も同じ警告を出す）
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

## 当月送信可能数の確認（`getCreatorLineChannelsMessageQuotas`）

引数なしの読み取り専用ツール。`messageQuotas[]` が返り、要素は次の3項目だけ（残り通数・警告フラグ・アカウント名は返らない）。

| 項目 | 内容 |
|---|---|
| `channelId` | LINE公式アカウントID。`getCreatorLineChannels` の `id` と同じ値で、これで名前に引き当てる |
| `monthlyLimit` | LINE 側の当月の送信上限数。**`null` は「上限なし」**（取得失敗でも 0 でもない） |
| `totalUsage` | LINE 側の当月の送信済みメッセージ数 |

- 残り通数は `monthlyLimit - totalUsage` で自分で計算する。`monthlyLimit: null` のときは計算せず「上限なし」と伝える。
- **ユーザーに出すのは `channelId` ではなくアカウント名**（作り方は上の「送信元 LINE公式アカウントの確定」と同じ）。複数アカウントがあるときはアカウントごとに分けて示す。
- **値は取得時点の最新値で、LINE 側の集計遅延によりリアルタイムの送信数と一致しないことがある**（ツール定義に明記）。ちょうどの残数として断定しない。
- **上限・送信済み数のいずれかを取得できなかったアカウントは一覧に含まれない。** `getCreatorLineChannels` にあるのに `messageQuotas[]` に無いアカウントは「上限なし」ではなく**取得できなかった**として扱い、その旨を伝える（0 や上限なしで埋めない）。
- `totalUsage` は **LINE公式アカウント単位の合計**で、同じ公式アカウントから送った他の配信（ワークフローの配信・LINE 側の管理画面からの配信など）も含む。いま作っている配信だけの送信数ではない。何が数えられるかの内訳は LINE 側の仕様なので、聞かれたら「LINE 側で数えた当月の送信数」とだけ答える。

### 2つの「上限」を混同しない

| | LINE 側の当月送信上限 | MOSH 側の配信可能数の上限 |
|---|---|---|
| 確認する手段 | このツール（`monthlyLimit` / `totalUsage`） | **残数を確認するツールは無い**（超過して初めて分かる） |
| 超過したときの挙動 | **未確認。** API の定義にも記載が無いので断定しない | 作成・予約確定が 403（→「新規作成」節）。配信実行時に超過していると、その配信は送られず失敗として記録される（`failureReason` が `ERROR_BUDGET_LIMIT`。→「一覧・詳細」節の失敗理由）。失敗した配信が自動で再送されることはない |

このツールの残数に余裕があっても MOSH 側の上限で 403 になることはある（逆もある）。「あと◯通送れます」は「その配信が必ず通る」という意味にならない。

**`monthlyLimit: null`（LINE側の上限なし）でも同じ。** LINE公式アカウントの追加メッセージ従量課金契約等では実際に上限なし（LINE Messaging API の `getMessageQuota()` が `type: "none"` を返す）になりうるが、これは MOSH 側の配信可能数の上限（403 / `ERROR_BUDGET_LIMIT`）とは無関係。「LINE側は上限なしなので気にせず送信して大丈夫です」と言い切らない。

## 編集（`patchCreatorContactMessagesEmail` / `patchCreatorContactMessagesLine`）

- **編集できるのは下書きのみ。** 配信予定・配信済みに実行すると 400（配信待ち状態または配信済みのメッセージは
  編集できません）になる。
- **配信予定として確定する（`isDraft: false`）前にも、「配信対象の件数の目安」節の手順で件数を確認する。**
  宛先を変えない編集・下書きに戻してからの再予約でも省かない（作成時から対象者は増減している）。
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
- 内容や配信日時を修正して配信予定に戻す場合: このツールで下書きに戻し、`get〜` で全項目を取得し
  （取得は下書きに戻す前でも後でもよい。前に取ると元の配信日時を把握できる）、
  **「配信対象の件数の目安」節で件数を確認**し、変更内容と件数を提示して承認を得てから、patch で
  `isDraft: false`・`scheduledAt` に未来の日時を指定する。件数確認は宛先を変えない場合も省かない。

## 複製（専用ツールなし。`get〜` → `post〜`）

管理画面の「複製」と同じことを、取得と新規作成の組み合わせで行う。状態（下書き・配信予定・配信済み・配信失敗）にかかわらず元にできる。

1. 元の配信を `get〜` で取得する（一覧の値には本文・配信対象が無いので使わない）
2. 引き継ぐ項目: メールは `title` / `body` / `filterConditions` / `isTrackingEnabled`、LINE は `lineChannel.id`（→ `lineChannelId`）/ `contents` / `filterConditions`
3. `taskName` は元の名前の末尾に「のコピー」を付ける（管理画面と同じ。255文字を超える場合は元の名前を削って収める）。ユーザーが名前を指定したらそちらを使う
4. **引き継がない項目**: `scheduledAt`（配信日時はユーザーに改めて決めてもらう）、`id` / `isSent` / `sentCount` / `failureReason` などの状態
5. 絞り込み条件に削除済みタグがあれば（→「削除済みタグの検出」）、その条件は外して選び直してもらう。管理画面の複製も削除済みタグは引き継がない
6. LINE の場合は**必ず `getCreatorLineChannels` を呼び**、元の送信元アカウントがまだ使えるかを確かめる（詳細取得で送信元が分かっていても省かない）。連携切れ（`isConnected: false` または一覧に無い）なら、送信元を選び直してもらう
7. URL を含む本文で `isTrackingEnabled: true` を引き継ぐ場合は、`tracking-consent.md` の同意確認を行う（既存の値をそのまま送る場合も同意が必要）
8. 「配信対象の件数の目安」節で件数を確認する（宛先が元と同じでも、全員宛でも省かない。元の配信のときから対象者は増減している）
9. SKILL.md Step 3 で新規作成と同じ項目と件数を提示し、承認を得てから `post〜` を呼ぶ

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
  （末尾が画像 ID になる形式。ドメインは環境ごとに異なる）。`linkUrl` / `buttonUrl` / `url`（タップ先）は外部URLでよい。
  素材の扱いのルールは SKILL.md の「LINE の素材制約」が正本。
- `carousel` の各カード: `title` 40文字 / `text` 60文字 / `buttonText` 20文字 / `buttonUrl` 1000文字 / `imageUrl` 2000文字以内。
  **7項目（`buttonUrl` / `postbackActions` / `openExternalBrowser` を含む）すべて必須**（使わないものは null）。
  ボタンの `buttonUrl` と `postbackActions` は独立に指定でき、「URL単独 / アクション単独 /
  併用 / 両方なし=タップしても何も起きない」の4パターンが取れる。併用するとアクションを実行してから URL を開く。
  **`postbackActions` キーを省くと形が合わず 400 になる**ので、タップ時アクションを使わないときも `null` を入れる。
- `image_carousel` の各カラム（`imageCarousels[]`）: **`imageUrl` / `linkUrl` / `postbackActions` の3キーすべて必須**
  （使わないものは null）。`linkUrl` と `postbackActions` は独立に指定でき、「URL単独 / アクション単独 /
  併用 / 両方なし=タップしても何も起きない」の4パターンが取れる。併用するとアクションを実行してから URL を開く。
  **`postbackActions` キーを省くと形が合わず 400 になる**ので、タップ時アクションを使わないときも `null` を入れる。
- `video`: `muxAssetId` は MCP で用意できる（下の「LINE 動画の `muxAssetId` を用意する」）。
  `previewMoshImageId`（プレビュー画像の ID）は**MCP から新規に採番できない**。既存の動画配信を `get〜` した値の流用か、
  ユーザーが値を持っている場合だけ指定できる。
- `rich_message`: `cells` の件数を `splitPattern` の分割数と一致させる
  （`ONE_BLOCK`=1 / `LEFT_RIGHT_SPLIT`・`TOP_BOTTOM_SPLIT`=2 / `GRID_FOUR`=4）。
  `cells` は 1〜4 件で分割数に対応。**`cells[]` の各マスは `url`（タップ先）と `postbackActions`
  （1〜3 件、使わないなら null）を独立に指定でき、
  「URL単独 / アクション単独 / 併用 / 両方なし=タップ不可」の4パターンが取れる**。
  飛び先の数が分割数に足りないときは、余るマスを `url: null` / `postbackActions: null`（タップ不可）にする。
  画像は `imageWidth` / `imageHeight` ともに 300〜1024px。

### タップ時アクション（`postbackActions`。`carousel` / `image_carousel` / `rich_message` 共通）

1つのタップ領域に 1〜3 件。`postbackActionType` で種類を指定する。

| 種類 | 形 | 意味 |
|---|---|---|
| `addContactTag` | `{ "postbackActionType": "addContactTag", "contactTagId": <id> }` | タップした人にコンタクトタグを付ける |
| `removeContactTag` | `{ "postbackActionType": "removeContactTag", "contactTagId": <id> }` | タップした人からコンタクトタグを外す |
| `sendMessage` | `{ "postbackActionType": "sendMessage", "messages": [{ "messageType": "text", "message": "<本文>" }] }` | タップした人にテキストを1通返す（`messages` は1件のみ・本文 5000 文字以内） |

- `contactTagId` は `getCreatorContactTags` で引く（推測で埋めない）。付けるのは**コンタクトタグ**で、顧客タグではない。
- アクションの内容（付ける・外すタグ名、返すメッセージ本文）は、SKILL.md Step 3 の提示に含めて承認を得る。
- `sendMessage` の本文にも SKILL.md の本文規約（与えられた情報の言い換えに留める）を適用する。

### LINE 動画の `muxAssetId` を用意する

**前提: プレビュー画像（`previewMoshImageId`）の当てがあること**（既存の動画配信から流用する等）。当てが無ければ下の手順を始めない（SKILL.md「LINE の素材制約」）。

1. `postCreatorLineMessageMuxAsset`（引数なし）で `uploadUrl` と `muxAssetId` を受け取る
2. ユーザーの手元の mp4（200MB 以下）を、`uploadUrl` へ HTTP PUT する（curl など。MCP はこの PUT を代行しない）
3. `getCreatorLineMessageMuxAsset` を 10 秒間隔で呼ぶ。`playbackId` が空文字でなくなり、`staticRenditions` に
   `name: highest.mp4` かつ `status: ready` の要素が出るまで待つ。**それより前に配信に使うと配信に失敗する**
4. `status: ERRORED` になったら手順 1 からやり直す

動画ファイルの実体を PUT できない環境（手元にファイルが無い等）では、管理画面での作成を案内する。

## 一覧・詳細

- 一覧（`post〜Search`）: `limit` / `offset` / `sort` / `keyword`（配信タスク名・件名・本文の部分一致）。
  **本文・配信対象・送信元の公式アカウント（`lineChannelId`）は返らない**（編集前に `get〜` が必要な理由）。
- 詳細（`get〜`）: 全項目を返す。編集はここを起点にする。
- 一覧には状態の絞り込みが無い。配信予定を見るときは `sort: scheduledAtAsc` にする。

### 状態の判定（管理画面の「配信ステータス」と同じ順で、上から最初に当てはまったもの）

1. **配信失敗**: `failureReason` が null でない
2. **下書き**: `isDraft: true`
3. **配信済み**: `isSent: true`
4. **配信予定**: 上のどれにも当たらない

### 失敗理由（`failureReason`）の伝え方

| 値 | ユーザーへの伝え方 | 対処 |
|---|---|---|
| `ERROR_BUDGET_LIMIT` | 配信可能数の上限を超えていたため送られなかった | プランの配信可能数を確認してもらう（MCP では残数を確認できない） |
| `ERROR_NOT_RETRYABLE` | 送信中にやり直せないエラーが起きて送られなかった | 原因は MCP では分からない。必要ならサポートへの問い合わせを案内する |
| `CONTACT_TAG_NOT_FOUND` | 絞り込み条件に使っていたタグが削除されていたため、配信を止めた | 条件を直した配信を新しく作る（タグの選び直しが必要） |

どの失敗でも自動で再送はされない。生の値（`ERROR_...`）はユーザーに見せない。

### 削除済みタグの検出（`filterConditionContactTags`）

詳細（`get〜`）の `filterConditionContactTags` は、絞り込み条件のタグのうち**今も残っているもの**だけを返す。
`filterConditions.tag.id[].value` にあるのにここに無いタグは**削除済み**。

- 削除済みタグを含む配信は、配信時に `CONTACT_TAG_NOT_FOUND` で止まる。編集・複製・予約確定の前にこの差を確認し、
  見つかったらタグ名が分からなくなっていることを伝えて、条件を選び直してもらう（削除済みの ID を残したまま承認を求めない）。
- 条件のタグ名をユーザーに示すときも、ID ではなくこの配列のタグ名を使う。
