# 埋め込み変数（差し込み文字）— S-11 で使う判定材料

> **正本は [`workflow-builder` の references/content-schema.md](../../workflow-builder/references/content-schema.md) の「埋め込み変数（差し込み文字）」節**。本ファイルはユーザーの指示により、`workflow-review` が S-11（埋め込み変数の使用可否）を builder に問い合わせずに自己完結で判定できるよう複製したもの。**builder 側の対応表を更新したときは、このファイルも同じ内容に同期すること**（内容の追加・変更・削除は必ず builder 側を正として反映する。このファイル単体を先に書き換えない）。

`SEND_EMAIL` / `SEND_LINE_MESSAGE` の本文には `{{variable_name}}` 形式（二重中括弧）の埋め込み変数を使える。変数が解決されるのは**メッセージ本文のみ**で、`SEND_EMAIL` の `subject`（件名）では解決されず `{{...}}` がそのまま件名に残る（件名には使わない）。**Excel等の外部ソースにある `%foo%` のようなプレースホルダ記法はMOSHでは解釈されない**（そのまま文字列として残るだけ）ので、本文を外部ドキュメントから転記した形跡があれば疑う。

## 対応変数一覧（この5つ以外は存在しない）

| 変数名 | 意味 / データソース | 使える条件 |
|---|---|---|
| `line_name` | コンタクトのLINEプロフィール表示名 | `actionType: SEND_LINE_MESSAGE` の時のみ（`SEND_EMAIL`では常に空文字） |
| `guest_name` | ゲスト（Moshユーザー）の名前 | `actionType: SEND_EMAIL` かつ `triggerType` が `SERVICE_APPLIED` / `SERVICE_SCHEDULE_REMINDER` / `INSTALLMENT_PAYMENT_FAILED` / `SUBSCRIPTION_PAYMENT_FAILED` の時のみ。それ以外は空文字 |
| `service_name` | トリガーに紐づくプラン・サービス名 | `triggerType` が `SERVICE_APPLIED` / `SERVICE_SCHEDULE_REMINDER` / `INSTALLMENT_PAYMENT_FAILED` / `SUBSCRIPTION_PAYMENT_FAILED` の時（決済失敗系も実行コンテキストに対象プラン・サービスの参照が積まれるため値が入る）。それ以外のトリガー（`MARKETING_LEAD_BENEFIT_RECEIVED` / `LINE_CHANNEL_CONTACT_REGISTERED` / `CONTACT_TAG_ADDED` / `INFLOW_ACTION_CONVERTED`）では常に空文字 |
| `reservation_time_range` | 予約日時の範囲（`YYYY年M月D日 HH:mm〜HH:mm`, JST） | `SERVICE_SCHEDULE_REMINDER`は常に対応。`SERVICE_APPLIED`はプラン・サービスの`serviceType`が「予約(event)」または「個別(private)」の場合のみ（コンテンツ/サブスク/オンライン単体のプラン・サービスでは空文字） |
| `zoom_url` | ZoomのjoinURL | `reservation_time_range`と同条件に加えて、プラン・サービスの`locationType`がオンライン/ハイブリッドかつクリエイターがZoom連携済みの場合のみ。条件を満たさない場合は静かに空文字になる（保存・送信はブロックされない） |

存在しない変数名（上表の5つ以外のキー）は**エラーにならずプレースホルダ文字列がそのまま残る**（無言の失敗）。上表の5変数は、使える条件を満たさないトリガー・アクションの組み合わせでは**空文字**に置換される（実害例: メール本文の宛名に `{{line_name}}` を書くと、保存・公開はエラーにならないが宛名が空文字のまま配信される）。どちらも意図通りに差し込まれないため、対象の trigger × action の組み合わせが上表の条件を満たすかを必ず確認する。

## レビューでの調べ方

1. 各 stage の `action` 内の `sendEmail.message` / `sendLineMessage.messages[].text` から `{{...}}` パターンを正規表現（例: `/\{\{([a-z_]+)\}\}/g`）で全て抽出する
2. そのステージが従う trigger（先頭 stage の `triggerType`）と `actionType` の組み合わせで、上表に照らして使用可能か判定する
   - 変数名が上表の5つに無い → 使用不可（無言の失敗）
   - 変数名はあるが「使える条件」列の trigger × action を満たさない → 空文字化（無言の失敗）
3. `SEND_EMAIL` の `subject` に `{{...}}` があれば、条件を満たすかに関わらず無条件で不可（件名では解決されない）
4. `reservation_time_range` / `zoom_url` は `serviceType` / `locationType` / Zoom連携有無というプラン・サービス側の設定にも依存するため、trigger × action の組み合わせだけでは白黒つけられないことがある（`SERVICE_APPLIED` で `serviceType` が不明な場合など）。その場合は `要確認` にとどめ、ユーザーに対象プラン・サービスの種別を確認する
