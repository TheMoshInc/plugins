# ワークフロー設計ベストプラクティス

## ID 参照の確認手順

ワークフローに埋め込む各種 ID のうち、`serviceId` は MCP に検索・一覧取得ツールが無い。それ以外（`inflowActionId` / `creatorLineChannelId` / `contactTagId` / `autoWebinarId` / `lineRichMenuId`）は対応する一覧取得ツールで取得・実在確認できる。`benefitId` は特典提供終了により新規設定しない（既存ワークフローの維持時のみ）。

| ID | MCP で取得できるか | 取得経路 |
|---|---|---|
| `inflowActionId` | ✅ | `getCreatorScenariosInflowActions` |
| `creatorLineChannelId` | ✅ | `getCreatorLineChannels` |
| `contactTagId` | ✅ | `getCreatorContactTags`（無ければ `postCreatorContactTags` で新規作成できる。同名は既存 id を返す冪等動作） |
| `autoWebinarId` | ✅ | `getCreatorAutoWebinars` |
| `lineRichMenuId` | ✅ | `getCreatorLineRichMenus`（`creatorLineChannelId` で絞り込み可） |
| `serviceId` | ❌ | ユーザーから受け取る（管理画面のプラン・サービス管理） |

`serviceId` が必要な trigger/condition（`SERVICE_APPLIED` / `SERVICE_SCHEDULE_REMINDER` / `CONDITION(SERVICE_APPLICATION_STATUS)` / `CONDITION(BANK_TRANSFER_STATUS)`）では、要件ヒアリングの段階で「**該当リソースの ID を教えてください**」を必ず聞き、MCP からは検索・実在確認ができないことを踏まえてユーザー申告のまま扱う（下書き保存時は実在検証されないため、公開前レビューで改めて口頭確認する）。それ以外の ID は対応する一覧取得ツールで実在するものを選んでもらう（一覧を提示する際は名前で選んでもらい、生の ID を会話の主役にしない）。`MARKETING_LEAD_BENEFIT_RECEIVED` は特典提供終了により**新規設定しない**。

## ライフサイクル運用

基本フロー: **`INACTIVE` で作る → ユーザー確認 → 公開（`ACTIVE` 化）**

- **作成直後**: 必ず `INACTIVE`
- **stages 構築中**: `INACTIVE` のまま何度でも更新する。下書き段階では `benefitId` / `serviceId` の実在はチェックされない（他の参照 ID は PATCH 時にも検証され不正なら 400）ので、`serviceId`（および既存ワークフローの `benefitId`）は公開前にユーザーへ口頭で確認する
- **公開（`ACTIVE` 化）**: 公開時に参照リソースの実在・同時稼働数の上限が確認され、満たさない場合は理由を示すエラーが返る
- **停止（`INACTIVE` 化）・廃止（`ARCHIVED`）**: `patchCreatorScenariosLifecycle` で行う

## 動作仕様

### 公開（`ACTIVE` 化）時の検証

- `ACTIVE` 化（公開）時に、参照リソースの実在と同時稼働数の上限が確認される。満たさない場合は理由を示すエラーが返る
  - 同時に稼働できるワークフロー数の上限に達している場合
  - 「指定された特典が見つかりません。特典が削除されていないか確認してください。」（トリガーの特典が削除済み等）
  - 「LINE関連の開始条件またはステップを使用するには、ワークフローにLINE公式アカウントを設定してください。」（LINE公式アカウント未設定）
- 対処: エラーの原因（参照リソース不足・稼働数上限）を解消してから再度 `ACTIVE` 化する。ユーザーには平易な日本語で原因と対処を伝える

### 下書き（`INACTIVE`）中は検証が緩い

- 存在しない `benefitId` 等を入れても保存できてしまう
- 「保存できた」＝「動く」ではないので、参照 ID の実在をユーザーに口頭確認する
- 下書き中は不正な ID でも保存できてしまうので、公開前のレビュー段階で必ず ID 内容をユーザーに見せて確認する（公開時に検証され弾かれるが、手戻りを避けるため事前確認する）

### レスポンス JSON は送信時と形が違う

- `getCreatorScenario` のレスポンスでは、triggerType に該当しない trigger サブフィールドは `null` で返る
- ただし **`lineChannelContactRegisteredTrigger` だけはレスポンスから完全省略**される
- 受け取った JSON をそのまま PATCH に戻したい場合は、欠落フィールドを `null` で補完してから送る

### `getCreatorScenarios` の `lifecycle` クエリは2値のみ

- `"INACTIVE_ACTIVE"`（稼働 or 停止）か `"ARCHIVED"` のみ
- `"ACTIVE"` 単体や `"INACTIVE"` 単体は指定できない（クエリパラメータの enum）

## トリガー × アクションの相性

- **`LINE_CHANNEL_CONTACT_REGISTERED` / `CONTACT_TAG_ADDED` トリガーで `SEND_EMAIL` を選ぶ前に、配信先がメールアドレスを持つか確認する**。`SEND_EMAIL` の宛先は実行時にコンタクトのメールアドレス（ContactEmail）から解決される。メールアドレスが紐付いていないコンタクト（LINE 友達追加だけで登録されたコンタクト等）は配信失敗（実行履歴に「送信先コンタクトなし」）となり、**そのコンタクトの後続ステップも打ち切られる**（保存・公開はエラーにならない）。LINE 友達追加起点は原則 `SEND_LINE_MESSAGE` を選ぶ。タグ付与起点でメールを使う場合は、対象コンタクトがメールアドレスを保有している前提（特典のメール取得経由等）かをユーザーに確認する
- コンタクト起点トリガー（LINE友達追加・タグ付与・流入経路CV。既存の特典取得ワークフローも含む）でも、MOSH アカウントを前提とする CONDITION（`SERVICE_APPLICATION_STATUS` / `AUTO_WEBINAR_PARTICIPATION` / `AUTO_WEBINAR_WATCH_TIME` / `BANK_TRANSFER_STATUS`）は使える（実行時にコンタクトへ紐付いた MOSH アカウントで評価される）。ただし MOSH アカウントが紐付いていないコンタクトは対象者を解決できず、その CONDITION で失敗する
- 予約リマインド（`SERVICE_SCHEDULE_REMINDER`）の最初のメッセージは WAIT_TIME を挟まずトリガー直後に置く。リマインド対象日時を跨ぐ大きな WAIT_TIME を設定しない（開催後に届くリマインドになる）
- `INFLOW_ACTION_CONVERTED` トリガーの流入経路は LINE 公式アカウントに属する。**ワークフロー最上位の `creatorLineChannelId` を、流入経路が属するアカウントと一致させる**（`getCreatorScenariosInflowActions` のレスポンスの `creatorLineChannelId` で確認する。不一致だとトリガーと配信のアカウントがずれる）

## ノード接続ルール

全パスが最終的にメッセージ系ステップ（`SEND_EMAIL` / `SEND_LINE_MESSAGE` / `ADD_CONTACT_TAG` / `REMOVE_CONTACT_TAG` / `LINK_LINE_RICH_MENU` / `UNLINK_LINE_RICH_MENU`）で終端する構造にする。

- **WAIT_TIME で終わる「待って終わり」のパスを作らない**（トップレベル末尾・CONDITION の branch 末尾とも）。中間の WAIT_TIME の後にメッセージが来ない宙ぶらりん構造も同様
- CONDITION の**両 branch を同時に空にしない**。両方空なら存在意義がないので、「CONDITION を削除しますか？それとも条件マッチ時の処理を追加しますか？」とユーザーに確認する
- **片方の branch が空なのは valid**。マッチ側を空にすれば「該当者は早期終了」、不マッチ側を空にすれば「条件外の人だけ終了」という正当な設計パターンで、頻繁に使う。「両 branch を必ず埋めるべき」と過剰反応せず、ユーザーの意図に従う
- 連続するメッセージステップの間に WAIT_TIME を挟まない場合は、それが意図的か確認する（同時に複数通届く）
- stages を組んだら **`patchCreatorScenario` で送る前に配列を上記の観点で点検**し、保存後は `getCreatorScenario` で読み戻して再点検する

## 分岐への追加位置の確認（曖昧なら1段聞いてから）

「分岐のあとにメッセージを足して」という指示は3通りに解釈できる:

| 解釈 | stages 上の場所 | 意味 |
|---|---|---|
| (a) 枝の中 | `condition.branches[].stages` に追加 | matchValue ごとの別ルート（例: タグ持ちの人だけに送る） |
| (b) 分岐の後 | CONDITION ステージの次の stage | 合流後に全員へ共通で実行（片方 branch 空なら早期終了者を除く） |
| (c) 分岐のネスト | branch 内に CONDITION 型 action | 既存の枝をさらに条件分岐 |

**推測で (b) を選ばない**（合流後＝全員共通実行になり、想定外の人にも届くことがある）。どれを指しているか着手前にユーザーへ確認する。

## 机上デバッグチェックリスト（公開前レビューで必ず実行）

SKILL.md の Step 4 レビュー時（および「見直して」「懸念点は」「机上デバッグして」と言われた時）に、`getCreatorScenario` で取得した stages を以下の観点で点検する。問題があれば修正してから公開に進み、懸念点と修正内容を1〜3行でユーザーに報告する。

1. **トリガー×アクション整合**（上記「トリガー × アクションの相性」参照）
2. **ノード接続**（上記「ノード接続ルール」参照）
3. **日程の矛盾**: ローンチ型＝ローンチ日に間に合わないリマインド配信・販売終了後に届く CTA / リマインド系＝イベント開催日時を跨ぐ待機時間 / WAIT_TIME の計算結果が過去時刻になる設定（後続ステップが通知なく停止する。[content-schema.md](content-schema.md) の「WAIT_TIME」参照）
4. **メッセージ重複・抜け**: 同じ内容のメッセージが連続していないか、空文字メッセージが残っていないか
5. **埋め込み変数の不整合**: トリガーで提供されない変数を本文に使っていないか、`subject` に変数を使っていないか（[content-schema.md](content-schema.md) の「埋め込み変数」対応表が正）

## 代表ケース（骨子の参考）

ヒアリング結果に応じて取捨選択する。コピペせず、ユーザーの状況に合わせて再構成する。

- **ケースA ローンチ型**: タグ追加トリガー → ウェルカム配信 → 数日のナーチャリング配信（自己紹介・実績・価値観。1〜2日間隔）→ ウェビナー・個別相談へ誘導 → サービス案内・販売
- **ケースB リテンション型**: プラン・サービス申込トリガー → お礼＋当日案内 → 開講前日リマインド → 開講翌日フォロー → 終了後アンケート
- **ケースC セグメント型**: タグ付与トリガー → CONDITION（タグ・視聴履歴等）で分岐 → 該当セグメントだけに限定オファー配信（片方 branch 空の早期終了を活用）

配信間隔の目安（ユーザーに希望が無いと言われたら提示する一般値）:

- ローンチ型ナーチャリング: 1日後 → 3日後 → 5日後 → 7日後 のように1〜2日間隔
- リマインダー: 前日 / 当日1時間前 / 直前 の組み合わせ
- リテンション: 申込直後 → 開講前日 → 開講翌日 → 終了1週間後

## 命名規約

- ワークフロー名 (`name`) は管理画面でも表示されるので、用途がわかる名前にする
  - 良い例: `"ウェルカムメッセージワークフロー"`, `"単発レッスン申込後フォロー（3日後）"`
  - 悪い例: `"テスト"`, `"workflow_1"`
