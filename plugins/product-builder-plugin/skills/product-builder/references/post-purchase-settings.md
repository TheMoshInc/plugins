# 購入後設定（サンクスページ／メール・自動リダイレクト・感想レポート）

**いつ読むか**: 購入直後の画面と完了メール（サンクスページ／メール）、購入完了後の自動リダイレクト、感想レポートの自動依頼を設定・解除するとき。SKILL.md 手順 9 の本体と、項目・拒否メッセージの正本。

## 手順 9. 購入後設定（サンクスページ → 自動リダイレクト → 感想レポート）

対象が旧サービス（商品一覧・詳細で `isLegacyService: true`）なら書き込まず、[content-schema.md](content-schema.md) の「旧サービス（isLegacyService: true）」の案内文で返す。

購入完了後にゲストが見るもの・受け取るものの設定。3 つは独立しているので、依頼されたものだけを扱う。リクエストの形は [../examples/update-thanks-setting.json](../examples/update-thanks-setting.json)（バナー画像の ID はダミー値。アップロードで得た値か読んだ現在値に置き換える）/ [../examples/update-redirect-setting.json](../examples/update-redirect-setting.json) / [../examples/update-review-report-setting.json](../examples/update-review-report-setting.json) を参照する。

**サンクスページ／メール**（購入直後の画面と完了メールに載る）

1. `getCreatorProductPlanSettingsThanks` で現在値（本文・バナー画像・リンク先）を読む
2. `patchCreatorProductPlanSettingsThanks` を呼ぶ。**`content`（本文）は毎回必須**なので、本文を変えないときも現在値をそのまま送る。`bannerMediaId` / `bannerLinkUrl` は**省略すると維持、`null` を送ると削除**
3. `bannerLinkUrl` は `http://` か `https://` で始まる URL だけ。**バナー画像が無い状態ではリンク先を指定できない**（先に画像を設定するか、リンク先も `null` にする）
4. 新しいバナー画像は [media-upload.md](media-upload.md) の手順でアップロードし、返った ID を `bannerMediaId` に使う。**それらしい ID を作らない**
5. 読み戻して、本文の要約・バナーの有無・リンク先を提示する

**自動リダイレクト**（購入完了後に指定ページへ自動で移動させる）

1. `getCreatorProductPlanSettingsRedirect` で現在値（移動先の URL と待機秒数）を読む。**未設定ならどちらも `null`** で返る。設定済みなら「変更前 → 変更後」を提示し、上書きになることを伝える
2. `patchCreatorProductPlanSettingsRedirect` に `redirectUrl` と `pendingSecond`（待機秒数、0〜15）の**両方**を送る。作成も更新も同じツール
3. やめるときは `deleteCreatorProductPlanSettingsRedirect`。「自動リダイレクトを解除します（あとから再設定できます）」と伝えて**承認を取ってから**呼ぶ。未設定でも成功するので、有無は手順 1 の取得で確かめ、削除を確認の手段にしない
4. 銀行振込で支払ったゲストにはリダイレクトが働かないことを添えて伝える
5. `getCreatorProductPlanSettingsRedirect` で読み戻し、移動先の URL と待機秒数（削除後は両方 `null` ＝「未設定」）を提示する

**感想レポート**（購入後に感想レポートの投稿依頼メールを自動で送るか）

1. `getCreatorProductPlanSettingsReviewReport` で現在値を読む。**一度も設定していないプランは有効（`true`）として返る**
2. `patchCreatorProductPlanSettingsReviewReport` に `isAutoRequestEnabled` を送る
3. 読み戻して「自動送信: 有効 / 無効」で提示する

## 使用するツール

パスの `productId` / `productPlanId` はいずれも数値文字列。3 つの設定は互いに独立しており、それぞれ取得ツールがある。送る項目の必須・制約は下の各「項目」節が正本。

| ツール | 用途 | 注意 |
|---|---|---|
| `getCreatorProductPlanSettingsThanks` / `patchCreatorProductPlanSettingsThanks` | サンクスページ／メールの取得・更新 | 取得の応答は `{ content, bannerMediaId, bannerLinkUrl }`。未設定の項目は `content` が空文字、バナーの 2 項目が `null` |
| `getCreatorProductPlanSettingsRedirect` | 自動リダイレクトの現在値 | 応答は `{ redirectUrl, pendingSecond }`。未設定でもエラーにはならない。存在しない・自分のものでないプランは「見つからない」旨のエラー |
| `patchCreatorProductPlanSettingsRedirect` | 自動リダイレクトの作成／更新 | — |
| `deleteCreatorProductPlanSettingsRedirect` | 自動リダイレクトの解除 | パラメータはパスのみ。2 回呼んでも成功する |
| `getCreatorProductPlanSettingsReviewReport` / `patchCreatorProductPlanSettingsReviewReport` | 感想レポートの取得・更新 | 取得の応答は `{ isAutoRequestEnabled }` |

## サンクスページ／メールの項目

購入直後の画面と申し込み完了メールに載る内容。`getCreatorProductPlanSettingsThanks` / `patchCreatorProductPlanSettingsThanks`。

| 項目 | 取得 | 更新 | 意味・扱い |
|---|---|---|---|
| `content` | ○ | **必須** | 本文。10,000 文字まで。**変えないときも現在値をそのまま送る**。未設定なら空文字 |
| `bannerMediaId` | ○ | 任意 | タップで移動するバナー画像の ID。**省略＝維持、`null`＝削除**。未設定なら `null`。**提示しない** |
| `bannerLinkUrl` | ○ | 任意 | バナーをタップしたときの移動先。`http://` / `https://` で始まる URL のみ。**省略＝維持、`null`＝削除**。未設定なら `null` |

「省略」と「`null`」の違いをまとめる。

| 送り方 | `bannerMediaId` / `bannerLinkUrl` の結果 |
|---|---|
| 項目を入れない | 保存済みの値のまま |
| `null` | 削除される |
| 値を入れる | その値に変わる |

- **バナー画像が無い状態でリンク先だけは設定できない。** 保存済みの画像が無く `bannerMediaId` も送らないのに `bannerLinkUrl` を送ると拒否される。画像を消すときはリンク先も `null` にする
- `bannerMediaId` は自分がアップロードした画像の ID だけが通る。存在しない・他人の画像は拒否される。新しい画像は [media-upload.md](media-upload.md) の手順でアップロードして ID を得る。**それらしい ID を作らない**
- 銀行振込の購入では、この画面と完了メールは入金確認後に表示・送信される

## 自動リダイレクトの項目

購入完了後に指定ページへ自動で移動させる設定。`getCreatorProductPlanSettingsRedirect` / `patchCreatorProductPlanSettingsRedirect` / `deleteCreatorProductPlanSettingsRedirect`。

| 項目 | 取得 | 更新 | 意味・扱い |
|---|---|---|---|
| `redirectUrl` | ○ | 必須 | 移動先の URL。**未設定なら `null`** |
| `pendingSecond` | ○ | 必須 | 移動までの待機秒数。**0〜15** の整数。**未設定なら `null`** |

- 取得で 2 項目とも `null` なら未設定。「自動リダイレクトは未設定です」と伝える。値が入っていれば設定済みで、更新は上書きになる
- 作成と更新は同じツールで、2 項目とも毎回送る
- 削除は**未設定でも成功する**。有無は取得で確かめ、削除は「解除します（あとから再設定できます）」と伝えて承認を取ってから呼ぶ
- 更新・削除のあとは取得で読み戻し、移動先と待機秒数（削除後は「未設定」）を提示する
- 銀行振込で支払ったゲストにはリダイレクトが働かない

## 感想レポートの項目

購入後に感想レポートの投稿依頼メールを自動で送るかどうか。`getCreatorProductPlanSettingsReviewReport` / `patchCreatorProductPlanSettingsReviewReport`。

| 項目 | 取得 | 更新 | 意味・扱い |
|---|---|---|---|
| `isAutoRequestEnabled` | ○ | 必須 | `true` で自動送信する。**一度も設定していないプランは `true` で返る**（「未設定」ではなく「有効」と伝える） |

## 拒否されたときのメッセージ（購入後設定）

**再試行せず**、内容を伝えて次の行動を案内する。

| メッセージ | 意味 | 案内すること |
|---|---|---|
| バナー画像が未設定のときはリンク先 URL を指定できません。先にバナー画像を設定するか、リンク先 URL を null にしてください。 | バナー画像が無い状態で `bannerLinkUrl` を指定した | バナー画像を一緒に送る（新しい画像は [media-upload.md](media-upload.md) でアップロード）か、リンク先を `null` にする |
| bannerLinkUrl は http:// または https:// で始まる URL を指定してください。 | リンク先の形式が `http(s)` でない | URL を確認し直す |
| 指定された画像が見つかりません。 | `bannerMediaId` が存在しないか、自分がアップロードした画像でない | 商品の画像と同じ扱い（[content-schema.md](content-schema.md) の「拒否されたときのメッセージ」） |
