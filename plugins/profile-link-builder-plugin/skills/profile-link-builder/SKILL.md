---
name: profile-link-builder
description: MOSH のプロフィールリンク（クリエイターの公開プロフィールページ）の中身を MOSH の MCP 経由で確認・編集するスキル。プロフィール情報（名前・キャッチコピー・自己紹介・アイコン）、ヘッダー（デザイン・背景・SNSアイコン）、テーマ（背景色・グラデーション・背景画像）、メディアタブ（ギャラリー）、リンクタブのコンテンツ（見出し・画像・ウェブリンク・地図・SNSリンク・動画）、サービス／感想レポートの表示設定（表示/非表示・表示順・カードサイズ）を扱う。「プロフィールリンクの自己紹介を変えて」「ヘッダーのSNSアイコンを追加して」「背景色を変えて」「リンクタブに見出しを追加して」「このサービス非表示にして」「レビューの並び順を変えて」「プロフィールリンクの今の状態を見せて」など、`*ProfileLink*` 系ツールを使うリクエストで必ず起動する。公開ページに即時反映され全置換が多いため、取得→変更後の提示→承認→実行の順を守る。サービス・感想レポート自体の作成・削除、サービス購入導線・特典ブロックの追加・編集は対象外。
---

# MOSH プロフィールリンク（公開プロフィールページの編集）

## Overview

MOSH クリエイターの**公開プロフィールページ（プロフィールリンク）**を MCP で読み書きする。管理画面「プロフィール」の対応画面は、上部のプロフィール／ヘッダー／テーマ編集と、タブ「リンク / サービス / レビュー / メディア」。

**書き込みは公開ページに即時反映され、大半が全置換**（含めなかった項目は消える・上書きされる）。そのため**必ず現在値を取得してから**変更後を組み、承認を得て実行する（必須ルールA）。

> **提供状況**: 使えるのはツールカタログに出ているツールだけ。依頼の前に、どのツールがあるか確認する。コンテンツ系ツールの名称と本文のキー（`layout`）は [content-blocks.md](references/content-blocks.md)。「未確認」と書いた挙動は未確認で、断定して案内しない。
>
> **ツール名表記**: 本スキルでは API 定義の `operationId`（例: `patchProfileLinkTheme`）で参照する。実際のツール名にはサーバー識別子のプレフィックスが付く。

## When to use

- プロフィール情報（名前・キャッチコピー・自己紹介・アイコン）、ヘッダー、テーマを見る・変える
- ヘッダーのSNSアイコン、メディアタブ（ギャラリー）の画像・動画を足す・並べ替える・消す
- リンクタブのコンテンツ（見出し・画像・ウェブリンク・地図・SNSリンク・動画）を足す・直す・並べ替える・消す
- サービスカード／感想レポートの表示/非表示・表示順・カードサイズ（サービスのみ）を変える
- プロフィールリンクの現在の状態を確認する

## When NOT to use

- **サービス・感想レポート自体の作成・削除** → MCP に手段が無い（必須ルールF）
- **サービス購入導線・特典ブロックの追加・編集** → MCP ツールが提供されていない。管理画面を案内する（一覧での確認・削除は可）
- **商品・プランの情報（価格・説明・公開設定）** → `product-navigator` / `product-builder` スキル
- **LP（ランディングページ）** → `lp-builder` スキル
- **動画ファイルのアップロード** → このスキルでは扱わない（YouTube の URL か管理画面を案内。必須ルールG）。手元の画像ファイルは登録できる（必須ルールG）

## ツールの選び方

| やりたいこと | ツール | 詳細 |
|---|---|---|
| プロフィール情報の取得／更新 | `getProfileLinkProfile` / `patchProfileLinkProfile` | [profile-header-theme.md](references/profile-header-theme.md) |
| ヘッダー（デザイン・背景・文字色）の取得／更新 | `getProfileLinkHeader` / `patchProfileLinkHeader` | 同上 |
| テーマ（背景・文字色）の取得／更新 | `getProfileLinkTheme` / `patchProfileLinkTheme` | 同上 |
| ヘッダーのSNSアイコン | `getProfileLinkHeaderSnsLink(s)` / `post` / `patch` / `put…SnsLinks` / `delete` | [header-sns-gallery-media.md](references/header-sns-gallery-media.md) |
| メディアタブ（ギャラリー） | `getProfileLinkGalleryItems` / `putProfileLinkGalleryItems` | 同上 |
| 外部URLの画像を使えるようにする | `postProfileLinkMediaFromUrl` | 同上 |
| 手元の画像ファイルを使えるようにする | `postMeMediaUploadUrls` ＋ `curl` で送信 | 同上 |
| リンクタブのコンテンツ一覧・並び順・サイズ・削除 | `getProfileLinkContents` / `putProfileLinkContentsLayout` / `patchProfileLinkContent` / `deleteProfileLinkContent` | [content-blocks.md](references/content-blocks.md) |
| 見出し・画像・ウェブリンク・地図・SNSリンク・動画の追加／詳細取得／更新 | `getCreatorProfileLink{Heading,Image,Link,Map,Sns,Video}Content` / `postCreatorProfileLink{…}Contents` / `patchCreatorProfileLink{…}Content`（`getCreatorProfileLinkServiceContent` は取得のみ） | 同上 |
| サービス／感想レポートの一覧取得 | `getCreatorProfileLinkServices` / `getCreatorProfileLinkReports` | [service-report-tools.md](references/service-report-tools.md) |
| サービス／感想レポートの表示設定・並び替え | `patchCreatorProfileLinkService` / `patchCreatorProfileLinkReport` / `putCreatorProfileLinkServicesDisplayOrder` / `putCreatorProfileLinkReportsDisplayOrder` | 同上 |

画面上の用語（タブ名・ブロックの呼び方）との対応は [ui-model.md](references/ui-model.md) を、ユーザーへの提示文を書く前に読む。

## Workflow

基本フロー: 対象の特定 → **現在値の取得** → 変更後の提示 → **承認** → 実行 → 結果の報告。

### 1. 依頼の切り分けと対象の特定

触る対象（プロフィール／ヘッダー／テーマ／ヘッダーSNS／ギャラリー／コンテンツ／サービス／感想レポート）を最初に確定し、上の表で使うツールを決める。そのツールがカタログに無ければ、**できないことを伝え、管理画面での操作を案内する**。

ID が要る対象（コンテンツ・SNSリンク・ギャラリー項目・サービス・感想レポート）は、一覧取得で ID を特定する。名前が複数に一致したら、推測せずユーザーに選んでもらう（必須ルールD）。

作成・削除できない対象（必須ルールF）の依頼は、この時点で案内に切り替える。

### 2. 現在値の取得

変更前に必ず取得する。更新系は全置換が多く、**変えない項目も現在値で埋める**ため。

| 対象 | 取得に使うもの |
|---|---|
| プロフィール／ヘッダー／テーマ | 各 `getProfileLink{Profile,Header,Theme}` |
| コンテンツ1件 | `getProfileLinkContents`（一覧で `type` と `id` を特定）→ 対応する type 別の取得 |
| ヘッダーSNS／ギャラリー | 一覧取得（全件。ページングは必須ルールB） |
| サービス／感想レポート | `getCreatorProfileLinkServices` / `getCreatorProfileLinkReports` |

**取得ツールがカタログに無い環境では**、まず**対象の ID と現在値をユーザーが分かるか**を聞く。ID は管理画面に表示されないことがあり（画面で確認できる手段は確認できていない）、クリエイターが答えられないことも多い。ID と現在値（項目を具体的に挙げて聞く）が得られたら進める。**得られなければ実行せず**、「ここからは変更できないので、管理画面で操作してください」と案内して終える（必須ルールA）。

### 3. 変更後の内容の提示

変更前 → 変更後を項目ごとに示す。変えない項目も「そのまま」と明記する。**全置換で消えるもの（含めなかった項目）がある場合は、消えるものを明示する。**

### 4. 承認（人ゲート）

**必ず停止して承認を得る。** 公開ページに即時反映される操作であり、全置換・削除は取り消せない。「〇〇にしといて」のような一任表現も承認の代わりにはならない。削除は対象を名指しして確認する。

### 5. 実行

- 更新系: ステップ2で取得した値をそのまま使い、変更する項目だけ差し替えて呼ぶ。**画面表示用の URL ではなく ID を渡す**（必須ルールE）
- 複数ツールの連続実行（画像取り込み → コンテンツ追加など）は、途中で失敗したら止めて状況を報告する。先へ進めない
- 並べ替え・全置換の組み方は [examples/](examples/) を見る

### 6. 結果の報告と確認

何がどう変わったかを平易な日本語で伝える。**完了形で書けるのは、ツールが成功を返した後だけ。** 可能なら実行後に取得ツールで読み戻して、意図どおりか確認してから報告する。公開ページへの反映は基本的に即時。

## 必須ルール

### A. 全置換の更新は、現在値を取得してから。取得できなければ聞く／実行しない

更新系ツールの多くは全項目必須の全置換。変えない項目の値を推測・創作して送ると、公開ページの他の部分を壊す。**取得ツールで読んだ値、または取得ツールが無い環境でユーザーが明示した値だけ**を使う。**直前の会話に出た値でも、今も正しい保証は無い**（画面側で変更が入り得る）ので、更新の直前に取り直す。

### B. 一覧は全件取り切る。並べ替えは取得した全件を見て組む

- 一覧取得（コンテンツ・SNS・ギャラリー）は limit 未指定だと20件まで。**limit に100を指定し、`totalCount` が取得件数より多ければ offset を進めて全件取り切る。** 取り切る前に全置換・並べ替えを組まない
- サービス／感想レポートの一覧は limit 省略で全件返る
- **サービス／感想レポート／ギャラリーの `displayOrder` は0始まりの絶対値**（ユーザーの「1番目」= `0`。ユーザーに現在の順番を伝えるときは逆に `+1` して「◯番目」と言う）。取得した全件の現在値を見て希望順の全体を組む。現在値を知らないまま `0, 1` の連番を振ると他の対象と衝突する（0始まりは画面の並べ替え保存の挙動で確認済み）。コンテンツの並べ替えは、**一覧を全件取れているときは、全件を希望順に並べて渡す**（1件だけ送る方法は、全件を取得できない場合の保険。[content-blocks.md](references/content-blocks.md)）
- **取得ツールが無い環境で「1件だけ動かす」依頼**（サービス／感想レポート／ギャラリー向け。「3番を一番上に」など。コンテンツの1件送信は入れ替えにならず、その1件が動いて残りは相対順を保つ）は、全件の再採番や全件の現在値を求めない。put は含めた対象だけ更新される。**動かす対象と、移動先の位置にいる対象の現在値だけ**を聞き、移動先の値が他と重ならないことを確認してから、その1〜2件だけを送る（重なる可能性を一言伝える）。**この方法は「入れ替え」になる**（移動先にいた対象が、動かす対象の元の位置へ移る）。他の対象の順序を保ったまま「挿入」したいときは、全件の現在値が必要。取得ツールがあれば、取得した全件から組む
- ツールごとの違い（部分更新か全置換か）は references が正本

### C. 「サイズ」は種類が違う。取り違えない

- サービスカードの `cardSize`（`SMALL`/`MEDIUM`/`LARGE`）はサービス表示設定だけ。感想レポートにサイズは無い
- リンクタブのコンテンツの `layout`（`small`/`horizontal`/`vertical`/`large`）は別物。**混ぜて渡さない**（値の形式も違う）

### D. ID を創作しない

コンテンツID・SNSリンクID・ギャラリー項目ID・サービスID・感想レポートIDは、**一覧取得の結果かユーザーが明示した値だけ**を使う。名前一致だけで決め打ちしない（同名があり得る）。**名前が近い候補が複数あるときは、完全一致があっても決め打ちせず、並べてどれかを確認する**（特に削除）。サービス／感想レポートの patch・put とギャラリーの put は、渡された ID の所有を検証せずそのまま保存する（誤った ID でもエラーにならない）。**他の対象や過去の会話の ID を流用しない。**

### E. 更新に渡すのは「ID」。表示用 URL は渡せない

取得結果には表示用の URL（`imageUrl` 等）と、保存用の ID（`imageId` / `thumbnailImageId` / `iconId` / `videoMediaId`）が並ぶ。**更新ツールが受け取るのは ID 側**。画像系の ID を空・`null` で送ると既存の画像が外れる（実機確認: ヘッダーの `imageId: ""`、ウェブリンクの `imageId: null`。`videoMediaId` と `videoPlaybackMode` の挙動は [content-blocks.md](references/content-blocks.md) の表）。**変えないなら取得した ID をそのまま渡し、画像を変える場合だけ新しい ID（必須ルールG）に差し替える。**

### F. 作成・削除できないもの

- **サービス・感想レポート**: MCP から新規作成・削除する手段が無い。サービスは商品/サービス設定側で用意する。感想レポートは顧客の投稿がベースで、「消して」には削除ではなく**非表示（表示設定の `isVisible: false`）を代替案として提案**する
- **サービス購入導線ブロック・特典ブロック**: コンテンツとして一覧には出るが、追加・編集のツールが無い。削除は `deleteProfileLinkContent` でできる
- 作成・削除の依頼は、できないことを伝えて管理画面での操作を案内する（案内先を創作しない）

### G. 画像は2経路で登録できる。動画ファイルのアップロードは扱わない

画像系フィールド（`imageId` 等）に渡す ID は、次のどちらかで得る。手順の詳細は [header-sns-gallery-media.md](references/header-sns-gallery-media.md)。

- **公開URLの画像**: `postProfileLinkMediaFromUrl`。取り込み対象はユーザーが提示した URL だけ。**取り込みは MOSH 側に画像を登録する操作なので、実行前に承認を取る**（画像カードの追加の承認とまとめて1回でよい）
- **手元の画像ファイル（jpeg / png / webp）**: `postMeMediaUploadUrls` で枠を発行 → `curl` で送信 → 返った `moshMediaId` を使う。**枠を発行する前に1回承認を取り、承認後は期限（15分）があるので止めずに通す**。処理完了を調べる手段は無いので、保存後に取得ツールで画像URLが解決されるか読み戻して確認する。

**アップロード動画（`videoSource: upload`）の新規登録は、このスキルでは扱わない**（YouTube 動画は URL で指定できる）。依頼されたら、YouTube の URL をもらうか、管理画面での登録を案内する。

### H. 取り消せない・重複する操作は、実行前に影響を言う

- `deleteProfileLinkContent` / `deleteProfileLinkHeaderSnsLink`: 復元不可
- `putProfileLinkHeaderSnsLinks` / `putProfileLinkGalleryItems`: 含めなかった項目が消える。空配列は全削除
- ヘッダーSNSの追加（`postProfileLinkHeaderSnsLink`）は、**同じ内容を再送すると重複して増える**。失敗や応答の取りこぼしのときは、再送せず先に一覧を取って状態を確認する（`put` の `id: null` 行の挙動は [header-sns-gallery-media.md](references/header-sns-gallery-media.md)）

### I. 並べ替えで「設定がまだ無い」対象の表示状態は、サービスとレビューで逆

`putCreatorProfileLinkServicesDisplayOrder` / `…ReportsDisplayOrder` は、表示設定がまだ無い対象を新規作成する。

- **サービス**: `isVisible: true`（表示）で確定する
- **感想レポート**: 設定が無いレポートは**非表示**扱い。並べ替えだけでは公開ページに出ない。表示したいレポートは `patchCreatorProfileLinkReport` で `isVisible: true` を明示する

非表示にしたいサービスを含む並べ替えも、`patchCreatorProfileLinkService` で明示する。詳細は [service-report-tools.md](references/service-report-tools.md)。

### J. 弾かれた URL は聞き直す

ヘッダーSNSの URL は、http / https 以外だとサーバーが 422 で拒否し、`javascript:` などは届く前に403で遮断される（詳細は [header-sns-gallery-media.md](references/header-sns-gallery-media.md)）。**弾かれたら、そのまま再送せず、`http://` か `https://` で始まる URL をユーザーに聞き直す。**

## ユーザーへの提示・コミュニケーション規約

- 応答は平易な日本語。**内部名を出さない**: フィールド名（`isVisible` / `displayOrder` / `cardSize` / `imageId` 等）、`SMALL` などの列挙値、ヘッダーのテンプレート名（`header-1`〜`header-6`。ユーザー自身が言った呼び方に合わせるのは可。こちらから出さない）、ツール名、JSON、HTTP 用語。画面の言葉（「表示/非表示」「表示順」「カードサイズ（小・中・大）」「ヘッダーデザイン」）に置き換える（[ui-model.md](references/ui-model.md)）
- 提示文には、**何を取得して、どの値を引き継ぐか**を含める（「今の自己紹介は〇〇です。キャッチコピーと名前はそのままにします」）
- 取得ツールが使えず聞いた値で進めるときは、「今の状態を確認する手段が無いため、教えていただいた内容をそのまま使います」と毎回添える
- ID・表示順・座標・Place ID などを推測で埋めない。分からなければ聞く。聞いても得られなければ実行せず、管理画面での操作を案内する
- **管理画面の案内は、確認済みの名称（「プロフィール」画面・タブ名）までにとどめる。** 画面内の位置（「上部」など）や「ID は画面に出ていない」といった未確認の断定をしない。「メディアのレコード」「受け皿」など技術寄りの言葉は避ける
- **未確認の用途は「使えるかは未確認」と添える**（例: gif や svg を取り込めても、背景などに使えるかは未確認。動作確認済みなのは png のリンクタブ画像カード・ヘッダー背景・ギャラリー）
- 承認を待つ提示では「以下の内容で進めてよいですか」と書く。「変更しました」は成功後だけ
- できない理由は仕組みでなく、ユーザーから見える事実で説明する（「このツールは公開されていない」ではなく「その操作はここからはできないため、管理画面での操作をお願いします」）

## よくあるミス

上のセクションでカバー済みの事項は再掲しない。ここは他セクションでカバーされない落とし穴のみ（追記前に既存セクション・references でカバーできないか確認し、できるならそちらを強化する）。

| NG | OK |
|---|---|
| 「自己紹介だけ変えて」で、名前・キャッチコピー・アイコンを空文字や創作値で埋めて送る | プロフィールだけは「変えない項目は `null`」で現状維持できる（他のヘッダー・テーマの全置換とは違う）。仕様は [profile-header-theme.md](references/profile-header-theme.md) |
| ヘッダーのSNSアイコンを1件追加するのに `putProfileLinkHeaderSnsLinks` で新規1行だけ送る | 1件追加は `postProfileLinkHeaderSnsLink`。put は全件の最終形を送るときだけ |
| コンテンツの `title` だけ直すのに、type 別 patch へ `title` だけ送る | type 別 patch も全項目必須。type 別の取得で現在値を取り、他の項目は取得値で埋める |

## エラーが返ったとき

- **401**: 認証エラー（ログインの期限切れなど）。再認証が必要なことを伝える。ID の誤りでは起きない（サービス／レビュー／ギャラリーは誤った ID でもエラーにならず保存される。必須ルールD）
- **404（コンテンツの並べ替え）**: 他のクリエイターのコンテンツや存在しない ID が1件でも含まれると、1件も更新されずに返る（アーカイブ済みの扱いは未確認）。一覧を取り直して ID を確認してから組み直す
- **400（外部URL画像の取り込み）**: 画像でない／サイズ超過（10MB）／取得失敗／http・https 以外。許可形式は png / jpeg / gif / webp / svg / ico。内容を直すか、別の画像URLをもらう（詳細は [header-sns-gallery-media.md](references/header-sns-gallery-media.md)）
- **422 / 入力エラー**: URL が http/https でない、文字数超過、不正・重複する ID など。内容を直してから再実行する。SNS の追加で失敗したときは、再送前に一覧で状態を確認する（必須ルールH）
- **403（HTML のエラー画面）**: 入力に危険な文字列が含まれ、サーバーに届く前に遮断された。内容を直し、同じ内容を再送しない（必須ルールJ）
- **必須項目が欠けたボディ**は、クライアント側のスキーマ検証でサーバーに届く前に弾かれる。ステップ2に戻り、取得した値で埋め直す

## References

| ファイル | 内容 | いつ読むか |
|---|---|---|
| [references/profile-header-theme.md](references/profile-header-theme.md) | プロフィール・ヘッダー・テーマの取得／更新（項目・制約・全置換の違い） | これらを触るとき、ツールを呼ぶ直前 |
| [references/header-sns-gallery-media.md](references/header-sns-gallery-media.md) | ヘッダーSNS・ギャラリー・外部URL画像取り込み | これらを触るとき、ツールを呼ぶ直前 |
| [references/content-blocks.md](references/content-blocks.md) | リンクタブのコンテンツ（一覧・並び順・サイズ・削除・type 別の項目） | コンテンツを触るとき、ツールを呼ぶ直前 |
| [references/service-report-tools.md](references/service-report-tools.md) | サービス／感想レポートの一覧取得・表示設定・並び替え4ツール | サービス／レビューを触るとき、ツールを呼ぶ直前 |
| [references/ui-model.md](references/ui-model.md) | 管理画面のタブ・ブロック名と内部名の対応 | ユーザーへの提示文を書く前、対象を切り分けるとき |
| [examples/theme-patch-trace.md](examples/theme-patch-trace.md) | 「背景色を変えて」を取得→提示→承認→実行→報告まで通すトレース（全置換の基本形） | 全置換の更新を初めて組むとき |
| [examples/header-sns-replace-trace.md](examples/header-sns-replace-trace.md) | ヘッダーSNSの追加・削除・全置換。取りこぼしと重複を避ける手順 | SNS／ギャラリーなど配列の全置換を組むとき |
| [examples/content-delete-and-reorder-trace.md](examples/content-delete-and-reorder-trace.md) | コンテンツの全件取得→削除確認→並べ替え | コンテンツの削除・並べ替え |
| [examples/service-patch-trace.md](examples/service-patch-trace.md) | サービスカードの表示設定（取得できる環境／できない環境の両方） | サービス／レビューの表示設定を触るとき |
| [examples/services-display-order-body.json](examples/services-display-order-body.json) | サービス3件の並べ替えボディ完成形 | 並べ替えの body の形に迷ったとき |
