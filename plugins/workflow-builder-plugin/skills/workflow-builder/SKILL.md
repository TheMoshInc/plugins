---
name: workflow-builder
description: MOSH のワークフローを MCP 経由で作成・編集・運用するスキル。要件ヒアリング → 空ワークフロー作成 → ステージ（trigger + action）構築 → レビュー → ユーザー承認のうえ公開、という段階で対話的に進める。「ワークフロー作って」「ワークフロー組んで」「ステップ配信を作って」「ワークフローを有効化」「ワークフローの分岐を追加」「workflow-builder」など、MOSH のワークフロー作成・編集・運用リクエストで使用する。流入経路・トラッキング同意・ワークフロー複製も含む。「流入経路一覧を取得して」「流入経路一覧を見せて」のような単純な参照・一覧取得リクエストでも、`*Scenario*` / `*InflowAction*` 系 MCP ツールを使う場合は本スキルを起動し、提示ルール（表示名変換など）を必ず適用する。
---

# MOSH ワークフロービルダー

## Overview

MCP ツール (`*Scenario*` および `*InflowAction*` / `*TrackingConsent*`) を使って、MOSH クリエイターのワークフローを作成・編集・運用する。空のワークフローを作成し、`stages` 配列で「いつ起動するか（trigger）」と「何を実行するか（action）」を組み立て、ユーザーの承認を得て公開（`ACTIVE` 化）する。

> **ツール名表記について**: 本スキル内では MCP ツールを OpenAPI の `operationId`（例: `postCreatorScenario`）で参照する。MCP クライアントが実際に提示するツール名にはサーバー識別子のプレフィックス（例: `mcp__<server>__postCreatorScenario`）が付くが、その形はユーザー環境により異なるため、本スキルでは付けない。

> **呼称について**: プロダクト上の呼称は「ワークフロー」、API / JSON のキー名は `scenario*`（例: `scenarioAction`, `scenarioTrigger`）。MCP ツール名 (`*Scenario*`) や JSON のキー名はスキーマ通りに使うこと。

`stages[].trigger` と `stages[].action` の構造には**強い制約**があるため、構築前に [references/content-schema.md](references/content-schema.md) を必ず参照する。`actionType` / `triggerType` ごとに必須となるサブオブジェクトが切り替わる仕様で、ナイーブに JSON を組むとほぼ確実に失敗する。

## 前提条件: 参照リソース

ワークフローは「**何かが起きたら → 何かをする**」を組み立てる仕組みなので、**起点となる既存リソースが最低1つ必要**。真っさらなテナント（まだ何も作っていない状態）ではどの trigger も構築できない。

最初に必ずユーザーへ確認する: 以下のうち**最低1つは既に作成済み**ですか？

| triggerType | 必要な既存リソース |
|---|---|
| `LINE_CHANNEL_CONTACT_REGISTERED`（LINE友達追加） | **LINE公式アカウント** が連携済み（`creatorLineChannelId` を`getCreatorLineChannels`で取得・確認可。trigger サブフィールド不要、ボディ最上位で指定） |
| `SERVICE_APPLIED`（プラン・サービス申込） | **プラン・サービス** が1つ以上（`serviceId` をユーザーから確認。MCP に検索ツール無し） |
| `SERVICE_SCHEDULE_REMINDER`（開催リマインダー） | **プラン・サービス** + 開催スケジュール（`serviceId` をユーザーから確認。MCP に検索ツール無し） |
| `CONTACT_TAG_ADDED`（タグ付与） | **コンタクトタグ**（`contactTagId` を`getCreatorContactTags`で取得・確認可。該当タグが無ければ`postCreatorContactTags`で新規作成できる） |
| `INFLOW_ACTION_CONVERTED`（流入経路 CV） | **流入経路** が1つ以上（`inflowActionId` を`getCreatorScenariosInflowActions`で取得可）。流入経路は LINE 公式アカウントに属するため、**ワークフロー最上位の `creatorLineChannelId` を流入経路が属するアカウントと一致させる**（一覧レスポンスの `creatorLineChannelId` で確認） |
| `INSTALLMENT_PAYMENT_FAILED`（分割決済失敗） | 不要（トリガー自体が ID を持たない。既存リソースの確認も不要） |
| `SUBSCRIPTION_PAYMENT_FAILED`（サブスク決済失敗） | 不要（トリガー自体が ID を持たない。既存リソースの確認も不要。初回失敗のみ発火し、同一請求期間内の決済リトライ失敗では再発火しない） |

action 側にも参照リソースが必要なケースがある:

| action | 必要な既存リソース |
|---|---|
| `SEND_LINE_MESSAGE` | **LINE公式アカウント**（`creatorLineChannelId` がワークフロー側に必要。`getCreatorLineChannels`で確認可） |
| `ADD_CONTACT_TAG` / `REMOVE_CONTACT_TAG` | **コンタクトタグ**（`getCreatorContactTags`で確認可。無ければ`postCreatorContactTags`で新規作成できる） |
| `LINK_LINE_RICH_MENU` | **個別リッチメニュー**（`lineRichMenuId` を`getCreatorLineRichMenus`で取得・確認可。ワークフローに紐づく LINE公式アカウントのメニューを選ぶこと。API はクリエイター所有なら別チャンネルのメニューでも保存・公開を通してしまうが、実行時に対象コンタクトが見つからず失敗しうる — 一致を強制するのは編集画面のみ） |
| `CONDITION` (`CONTACT_TAG`) | **コンタクトタグ**（`getCreatorContactTags`で確認可。無ければ`postCreatorContactTags`で新規作成できる） |
| `CONDITION` (`SERVICE_APPLICATION_STATUS`) | **プラン・サービス**（`serviceId` をユーザーから確認。MCP に検索ツール無し） |
| `CONDITION` (`AUTO_WEBINAR_PARTICIPATION` / `AUTO_WEBINAR_WATCH_TIME`) | **オートウェビナー**（`autoWebinarId` をユーザーから確認。MCP に検索ツール無し） |
| `CONDITION` (`BANK_TRANSFER_STATUS`) | **プラン・サービス**（`serviceId` をユーザーから確認。MCP に検索ツール無し。対象ゲストの当該プラン・サービスへの注文の銀行振込状況で分岐） |

参照したいリソースが無い場合は、ユーザーに**先に管理画面で作成**してもらってからワークフロー構築に着手する（`serviceId` / `autoWebinarId` は MCP に検索手段が無いため、実在の場合も含め常にユーザーから直接確認する）。**例外はコンタクトタグ**: `postCreatorContactTags` で MCP から新規作成できる（同名タグが既にあれば既存の `id` を返す冪等動作。返ってきた `id` はそのまま `contactTagId` に使える）ため、管理画面に誘導せずその場で作成して進めてよい。`MARKETING_LEAD_BENEFIT_RECEIVED`（特典取得）は特典提供終了により**新規設定しない**。既存ワークフローの読み取り・維持のみ行う。

## When to use

- ユーザーが新しいワークフローを作りたいと依頼したとき
- 既存ワークフローの stage（trigger / action）を更新したいとき
- 既存ワークフローを複製したいとき
- 流入経路（INFLOW_ACTION）を作成・参照・一覧取得したいとき（「流入経路一覧を取得して」のような単純な参照リクエストも含む）
- `*Scenario*` 系ツールが必要な文脈

## When NOT to use

- LP（ランディングページ）の作成・編集 → `lp-builder` スキル
- 単発のメール・LINE 送信（ワークフロー化しないもの）
- 受講者プロフィール・決済・受講者管理など、ワークフロー以外のドメイン

## Workflow（推奨フロー）

このスキルは **下書き作成 → ユーザー確認 → 公開（`ACTIVE` 化）** を基本フローとする。公開前に必ずユーザーの承認を取る。

### 1. 要件ヒアリング

ユーザーに以下を確認する:

- **起動条件（trigger）**: 何が起きたら走らせるか
- **実行内容（action）**: 何を実行するか、何ステップ並べるか
- **CV ポイント**: ユーザーに最終的にとってほしい行動はどれか（サービス購入 / 個別相談・説明会の予約 / リスト作成のみ＝配信先を増やすだけで有償提案はしない 等）。これが決まると各メッセージの CTA と締めの一通が決まる
- **連絡ツール**: LINE / メール / 両方のどれで送るか（集客経路ではなく「どの手段で送るか」）。メール配信は対象コンタクトのメールアドレス保有が前提（[references/best-practices.md](references/best-practices.md) の「トリガー × アクションの相性」参照）
- **タイミング**: 出来事の**直後**に届けたいのか、後日のある時点で該当していた人に届けば十分なのか。「◯◯したらすぐに」「◯◯した人には即座に」「タグが付いたらいつでも」という言葉が出て、かつ起動条件がその出来事自体（`SERVICE_APPLIED` / `CONTACT_TAG_ADDED`）ではなく `CONDITION` で拾う設計になりそうなら、その要件は**このワークフロー 1 本では実現できない**。専用トリガーの別ワークフローの有無を確認する（[references/best-practices.md](references/best-practices.md) の「単一ワークフローで閉じない要件」）
- **紐付け導線**（申込・リマインダー・決済失敗起点で LINE 配信・タグ操作を使う設計＝コンテキスト適合表の △ のとき）: 対象の申込者・購入者が事前にトラッキング付き URL をタップして MOSH にログインする導線（既存ワークフロー・一斉配信）がテナント内にあるか。無ければ `SEND_EMAIL` に倒すか、紐付け導線を先に設計する（同じく「単一ワークフローで閉じない要件」）
- **参照リソース**: 上の「前提条件」表の該当リソースが既にあるか、その ID
- **メッセージ内容**: メール件名/本文、LINE メッセージ等
- **クリエイター名・トーン**: メッセージ案を組み立てるときに使う

ヒアリングの進め方:

- 初回の要望が曖昧なとき（「ワークフロー作って」「何ができる？」等）は、いきなり質問を始めず**典型例を3〜4個提示して選んでもらう**（ローンチ型 / リテンション型 / セグメント型 / リマインダー型。骨子は [references/best-practices.md](references/best-practices.md) の「代表ケース」参照）
- 質問は1項目ずつ細切れにせず、**関連する2〜3項目をまとめて1回で聞く**
- スケジュール・配信間隔の目安が無いと言われたら、best-practices の「配信間隔の目安」の一般値を提示する

### 2. 空ワークフローの作成（INACTIVE で誕生）

`postCreatorScenario` で名前だけ指定して作成する。

```json
{ "bodyParams": { "name": "ウェルカムメッセージワークフロー" } }
```

返却された `id` を以降のステップで使用する。**作成直後のライフサイクルは `INACTIVE`（下書き）で固定**。

### 3. stages の構築（trigger + action の組み立て）

[references/content-schema.md](references/content-schema.md) を読み込み、構造ルールに沿った JSON を組み立てる。設計指針は [references/best-practices.md](references/best-practices.md) を参照。完成形の実例は [examples/inflow-line-tap-followup.json](examples/inflow-line-tap-followup.json)（LINE 流入経路→urlActions でタグ付与→WAIT_TIME→CONDITION・片方空 branch）を参照する。特に LINE メッセージや CONDITION を含む場合はこの形をベースに組み立てる。

`patchCreatorScenario` で `stages` を更新する:

```json
{
  "pathParams": { "id": <作成したワークフローID> },
  "bodyParams": {
    "creatorLineChannelId": <LINE公式アカウントID または null>,
    "stages": [
      { "trigger": { ... }, "action": { ... } },
      {                     "action": { ... } }
    ]
  }
}
```

`stages` は**配列まるごと置き換え**になる想定で組む。1ステージ目の `trigger` は必ず指定し、2ステージ目以降は `trigger` フィールド自体を省略する（先頭ステージの trigger を契機に連鎖実行される）。`trigger: null` は使用禁止 — スキーマは optional・非 nullable のため `null` は Zod バリデーションで弾かれる（ワークフロー編集画面本体も 2 ステージ目以降は trigger キー無しで送信する）。

PATCH 時は `action` 配下の構造が **`scenarioActionForUpdate`** 系（`condition` だけ `scenarioActionConditionForUpdate`）になる点に注意。詳細は content-schema 参照。

stages を組み上げたら、**送信前に [references/best-practices.md](references/best-practices.md) の「ノード接続ルール」と「トリガー × アクションの相性」で点検する**（WAIT_TIME 終端・宙ぶらりん・両 branch 空・メアド未取得トリガー×SEND_EMAIL の検出）。

> ⚠️ **PATCH 200 OK ≠ 動作保証**: INACTIVE 状態の PATCH では **`benefitId` / `serviceId` の実在が検証されず、存在しない ID でも 200 OK で保存できてしまう**（実在が検証されるのは公開＝`ACTIVE` 化時。`benefitId` は既存の特典取得ワークフローの維持時のみ）。それ以外の参照 ID（`contactTagId` / `inflowActionId` / `creatorLineChannelId` / `autoWebinarId` / `muxAssetId` / `lineRichMenuId`）は PATCH 時にも実在検証され、不正なら 400 で弾かれる（保存できた時点で実在は保証される）。「保存できた」を「正しく動く」と勘違いせず、`serviceId`（および既存ワークフローの `benefitId`）はユーザーに実在のものかを口頭で確認すること。

### 4. レビュー（公開前の最終確認）

MOSHのAPIは `ACTIVE` 化時に埋め込み変数の可否を検証しない（フロントエンドの編集画面だけがこのチェックを行う）。MCP経由で操作する場合はこのガードが効かないため、**埋め込み変数のチェックもこのレビューの一部として自分で行う**。

1. `getCreatorScenario` で現在の状態を取得する
2. **JSON を貼らずに自然言語の箇条書きで要約**してユーザーに提示する（「ユーザーへの提示・コミュニケーション規約」§1 参照）。最低限の項目: ワークフロー名 / 公開状態 / トリガー（と参照リソース ID） / アクション（メール件名・本文の要旨、トラッキング有無等）
3. `serviceId`（および既存ワークフローの `benefitId`）が実在のものか口頭で確認する（この2つだけは INACTIVE の PATCH で実在検証されない。他の参照 ID は保存できた時点で実在が保証されている）
4. **埋め込み変数チェック**: 各 stage の `action` 内の `sendEmail.message` / `sendLineMessage.messages[].text` から `{{...}}` パターンを全て抽出し、そのステージが従う trigger（先頭 stage の `triggerType`）と `actionType` の組み合わせで [content-schema.md](references/content-schema.md) の対応表に照らして使用可能か確認する。使用不可の変数が1つでもあれば、ユーザーに「この文言は現在のトリガーでは差し込まれず空欄配信になります」と日本語で具体的に指摘し、修正（変数を外す／使える変数に変える／トリガーを変える）してもらう
5. **机上デバッグ**: [references/best-practices.md](references/best-practices.md) の「机上デバッグチェックリスト」を実行する（トリガー×アクション整合 / ノード接続 / 日程の矛盾 / メッセージ重複・抜け / 埋め込み変数 / メッセージ内容の参照整合性）。順序は**まず構文解析**（content-schema.md の構造規律・ノード接続ルール・参照整合性・「埋め込み変数」対応表＝対象者が誰でも JSON とテナント設定だけで一律に決まるもの）**→ 次に意味解析**（content-schema.md の「コンテキスト適合表」＝対象者の紐付け・メール保有・友だち状態で結果が変わるもの。定義は content-schema.md の「検査の二層構造」）。API スキーマは trigger 種別で action 種別を制限しないため、意味の検査は自分で行うしかない。問題があれば修正し、懸念点と修正内容を1〜3行で報告する。ユーザーから「見直して」「懸念点は」と言われた場合も同じチェックを実行する
6. 「この内容で公開しますか？」と必ずユーザーに確認する（埋め込み変数・机上デバッグに問題がある間は公開に進まない）
7. 修正が必要なら Step 3 に戻る

### 5. 公開（`ACTIVE` 化）

ユーザーが公開に合意したら、`patchCreatorScenariosLifecycle` で `ACTIVE` 化する:

```json
{ "pathParams": { "id": <ワークフローID> }, "bodyParams": { "lifecycle": "ACTIVE" } }
```

`ACTIVE` 化時には参照リソースの実在や同時稼働数の上限が確認され、満たさない場合は理由を示すエラーが返る（例:「指定された特典が見つかりません。」「LINE公式アカウントを設定してください。」）。内容をユーザーへ平易に伝え、原因を解消してもらってから再度公開する。公開後は `getCreatorScenario` で稼働中（`ACTIVE`）になったことを確認する。

## ユーザーへの提示・コミュニケーション規約

このスキルを使うエンドユーザーは MCP / JSON / HTTP の専門知識を持たない前提で会話する。Claude が「内部で何をしているか」と「ユーザーに見せるもの」を切り分けること。

### 1. JSON 構造を直接ユーザーにレビューさせない

- 下書き作成前のドラフトも、PATCH 後のレビュー要約も、**JSON ブロックを並べてユーザーに「これでよいですか？」と聞かない**。
- 提示するのは、自然言語の箇条書きで「トリガー / アクション / メッセージ内容 / 公開状態」を要約したものだけ。
- 例: 「タグ追加をきっかけに、件名◯◯・本文◯◯のメールを1通送る構成です。トラッキングは OFF、下書き状態です。」
- ユーザーが「中身の JSON が見たい」と明示的に求めた時だけ、参考情報として JSON を出す。

### 2. ステータスコード / HTTP 用語をユーザー向け文面に出さない

- `200 OK`, `500`, `status: ...`, `PATCH`, `GET`, `endpoint` などの語彙は**ユーザー向けの文面に書かない**。
- 内部判定（例: 業務エラーの種別で分岐する）には使ってよいが、ユーザーには「保存できました」「公開しました」「特典が見つからないため公開できませんでした」など平易な日本語で伝える。
- 失敗時も「うまくいきませんでした。原因は◯◯のため、〜してください」のように原因と対処を日本語で示す。コードは出さない。

### 3. `creatorLineChannelId` を生の ID のまま出さない（流入経路一覧・ワークフロー一覧とも）

- `getCreatorScenariosInflowActions` / `getCreatorScenariosInflowAction` の結果をユーザーに提示するときは、`creatorLineChannelId` を数値のまま表示しない。
- **`getCreatorScenarios`（ワークフロー一覧）でも同様**: `creatorLineChannelId` はユーザーにとって意味を持たない内部IDなので、生の数値のまま出さず LINE公式アカウント名に変換して提示する（`null` の場合は「未設定」等と表示）。
- いずれの場合も必ず `getCreatorLineChannels` を合わせて呼び出し、`creatorLineChannelId` を突き合わせて LINE公式アカウントの `displayName` に変換してから提示する（詳細は [references/mcp-tools.md](references/mcp-tools.md)）。

### 4. 参照 ID が未確定でも捏造しない（中間状態で保存してよい）

- `serviceId` 等の参照リソース ID が未確定でも、**推測した ID（空文字・0・ダミー値を含む）を入れてはいけない**。`serviceId`（および既存ワークフローの `benefitId`）は INACTIVE の PATCH で実在検証が行われないため、不正な ID のまま保存してもエラーにならず、ユーザーが気付かず公開するとトリガー・アクションが実リソースに紐づかない壊れたワークフローになる。
- ただし**中間状態での保存は可能**（クラッシュしない）。決まっている部分（ワークフロー名・メール/LINE 本文・待機日数など）は先に保存してよく、**未確定の参照 ID はセットせず空けたまま**にして下書きを進められる。すべてを会話内に抱え込む必要はない。
- 未確定 ID については「`serviceId` が必要です。管理画面でプラン・サービス ID を確認してから教えてください」とユーザーに確認を促し、**実 ID が揃ってから当該 ID をセットして保存し、公開する**。捏造 ID を含んだままの公開（`ACTIVE` 化）は絶対にしない。

## 必須ルール

### A. `action` は `actionType` + 7サブオブジェクト全て指定

`scenarioAction` / `scenarioActionForUpdate` は型定義上、以下 8 フィールドが **全て required**:

- `actionType`（enum）
- `sendEmail` / `sendLineMessage` / `waitTime` / `condition` / `addContactTag` / `removeContactTag` / `linkLineRichMenu`

実態は `actionType` に対応する 1 フィールドだけを実オブジェクトで埋め、**残り 6 つは `null` を明示**する。`undefined` ではなく `null`。`actionType` には `LINK_LINE_RICH_MENU`（`linkLineRichMenu` を設定）と `UNLINK_LINE_RICH_MENU`（詳細フィールドを持たず全て `null`）も追加されている。

```json
{
  "actionType": "SEND_EMAIL",
  "sendEmail": { "subject": "...", "message": "...", "isTrackingEnabled": true },
  "sendLineMessage": null,
  "waitTime": null,
  "condition": null,
  "addContactTag": null,
  "removeContactTag": null,
  "linkLineRichMenu": null
}
```

### B. `trigger` も `triggerType` + 5サブオブジェクト全て指定

`ScenarioTrigger` 型のサブフィールドは `triggerType` 以外に 5 つ（`marketingLeadBenefitReceivedTrigger` / `serviceAppliedTrigger` / `serviceScheduleReminder` / `contactTagAdded` / `inflowActionConverted`）。対応する 1 フィールドだけ実オブジェクト、他は `null`。`lineChannelContactRegisteredTrigger` は型定義に存在しないため指定禁止。`LINE_CHANNEL_CONTACT_REGISTERED` の場合はサブフィールドが全て `null` で、LINE公式アカウントはボディ最上位の `creatorLineChannelId` で指定する。詳細は content-schema 参照。

### C. PATCH 時は `*ForUpdate` 系を使う

- `patchCreatorScenario` の `stages[].action` は `scenarioActionForUpdate` 構造
- 内部の `condition` は `scenarioActionConditionForUpdate`
- `branches[].stages[].action` は再帰参照を避けるため `additionalProperties: true` の汎用オブジェクト扱い（型定義は緩いが、サーバー側で再帰的に再検証される）

### D. ID は数値 / 文字列の使い分けに注意

| 種別 | 型 | 例 |
|---|---|---|
| `serviceId`（既存ワークフローの `benefitId` も同様） | **文字列**（数値文字列） | `"<service_id>"` |
| `contactTagId` / `inflowActionId` / `creatorLineChannelId` / `autoWebinarId` / `muxAssetId` | **整数** | `<contact_tag_id>` |
| `previewMoshImageId` | 文字列 | `"<preview_mosh_image_id>"` |

### E. LINE メッセージは `messages[]` の各要素が自分自身の `messageType` を持つ

`scenarioActionSendLineMessage` はトップレベルに `messages` の1フィールドのみ（`type` や
`isTrackingEnabled` をトップレベルに置かない）。`messages` は最大5件の配列で、**各要素ごとに**
`messageType` を持つオブジェクト（スキーマ上は0件も許容されるが下書き専用で、稼働させるには1件以上必要）。**`messages: ["文字列", ...]` のようなプレーン文字列配列は誤り**
（TEXTタイプでも `{ messageType: "TEXT", text, isTrackingEnabled, urlActions }` を1件ずつ並べる）。

- `TEXT` → `{ messageType: "TEXT", text: string(1-5000), isTrackingEnabled: boolean, urlActions: [...] | null }`
- `CAROUSEL` → `{ messageType: "CAROUSEL", altText, carousels[] }`（1〜4件。各要素の `postbackActions` はキー省略不可 — 使わない場合は `null` を明示）
- `IMAGE_CAROUSEL` → `{ messageType: "IMAGE_CAROUSEL", altText, imageCarousels[] }`（1〜4件）
- `VIDEO` → `{ messageType: "VIDEO", muxAssetId, previewMoshImageId }`
- `RICH_MESSAGE` → `{ messageType: "RICH_MESSAGE", altText, imageUrl, imageWidth, imageHeight, splitPattern, cells[] }`

詳細（`urlActions`/`postbackActions`の形・`RICH_MESSAGE`の`cells`件数対応表等）は content-schema.md の
「SEND_LINE_MESSAGE」節を参照。

### F. 埋め込み変数は次の5つ「だけ」（名前は完全一致・それ以外は無言で失敗）

メール本文・LINE 本文で使える `{{...}}` 埋め込み変数は `{{guest_name}}` / `{{line_name}}` / `{{service_name}}` / `{{reservation_time_range}}`（**`reservation_datetime` ではない**）/ `{{zoom_url}}` の**5つだけ**。この綴りを一字一句そのまま使う。ここに無い変数名（例: `{{reservation_datetime}}`, `{{customer_name}}`, `{{date}}` 等）は**存在せず、エラーにもならずそのまま文字列として配信される（無言の失敗）**ため、絶対に創作しない。

各変数が使える trigger × action の条件・空文字になる組み合わせは、[references/content-schema.md](references/content-schema.md) の「埋め込み変数」対応表が**唯一の真実**。**本文を書く前に必ず同表を参照**し、条件を満たさない・表に無い概念は変数を使わず固定文言にする。

## よくあるミス

上の「前提条件」「Workflow」「提示規約」「必須ルール A〜F」でカバー済みの事項は再掲しない。ここは**他セクションでカバーされない落とし穴のみ**を載せる（新しいミスを追記する前に、既存セクション・references でカバーできないか確認し、できるならそちらを強化する）。

| NG | OK |
|---|---|
| `getCreatorScenarios` の `lifecycle` に `"ACTIVE"` を指定 | 一覧クエリの enum は `"INACTIVE_ACTIVE"` か `"ARCHIVED"` の2値のみ（更新系 `patchCreatorScenariosLifecycle` の3値 enum とは別物。詳細は [references/mcp-tools.md](references/mcp-tools.md)） |
| `CONDITION(BANK_TRANSFER_STATUS)` で `conditionBankTransferStatus` を省略する／他の conditionType 使用時に `conditionBankTransferStatus: null` を書き忘れる | condition のサブフィールドは 5つ（`conditionServiceApplicationStatus` / `conditionContactTag` / `conditionAutoWebinarParticipation` / `conditionAutoWebinarWatchTime` / `conditionBankTransferStatus`）を毎回全部指定する。対応する1つだけ実オブジェクト、残り4つは null |
| `SEND_EMAIL` の `subject` に `{{...}}` 埋め込み変数を使う | 変数が解決されるのは本文のみ。件名に入れるとプレースホルダがそのまま残る。件名は固定文言にする |

## References

| ファイル | 内容 |
|---|---|
| [references/content-schema.md](references/content-schema.md) | `stages[].trigger` / `stages[].action` の JSON 構造、enum 一覧、actionType 別テンプレ、`*ForUpdate` 差分、検査の二層構造（構文→意味）、コンテキスト適合表（トリガーが供給する識別子 × アクションが要求する識別子）、埋め込み変数対応表 |
| [references/best-practices.md](references/best-practices.md) | ID 参照の確認手順、ライフサイクル運用、動作仕様、トリガー×アクションの相性、単一ワークフローで閉じない要件（即時反応の専用トリガー・紐付け導線）のヒアリング確認文、ノード接続ルール、机上デバッグチェックリスト、代表ケースと配信間隔の目安、命名規約 |
| [references/mcp-tools.md](references/mcp-tools.md) | 各 MCP ツールのパラメータ仕様 |
| [examples/inflow-line-tap-followup.json](examples/inflow-line-tap-followup.json) | LINE 流入経路 CV（`INFLOW_ACTION_CONVERTED`）→ 本文 URL のタップでタグ付与（`urlActions`/`postbackActions`）→ 1日待機 → タグ未付与（未クリック）だけに追客、の網羅例。CONDITION の**片方空 branch**（クリック済みは早期終了）を含む。ID はダミー値で、流入経路が属する LINE 公式アカウントと最上位 `creatorLineChannelId` の一致が必須 |
