---
name: lp-builder
description: MOSH (taiyaki) のランディングページ (LP) を taiyaki MCP 経由で作成・編集・公開するスキル。ヒアリングシート（事前調査で仮埋め・1往復）→ テンプレート選択 → 構成案の承認 → コンテンツJSON構築 → 下書き保存 → レビュー → 公開、というワークフローを対話で進める。「LP作りたい」「ランディングページ作って」「LP公開して」「LPの下書き更新」「lp-builder」など、MOSH の LP に関する作成・編集・公開リクエストで使用する。
---

# MOSH LP ビルダー

## Overview

taiyaki MCP ツール (`*LandingPage*` 系。本書のツール名は素の名前で書く。実際の呼び出し名は接続中のサーバー名に依存して `mcp__<server>__` が前置される) を使って、MOSH クリエイターのランディングページを作成・編集・公開する。テンプレートから起こすか白紙から作るかを選び、`content` フィールドに渡す JSON を組み立て、下書き保存・確認のうえで公開する。

`content` の構造には固有のルール（必須フィールド、PartType、attributes）がある。プリセットの JSON を写す場合は構造が担保されているので、`references/content-schema.md` は**プリセットを使わず組むとき・プリセットに無い部品を作るとき**に参照する。セクション ID の付け方は `references/section-plan.md`。

## When to use

- ユーザーが新しい LP を作りたいと依頼したとき
- 既存 LP の内容を更新したいとき
- LP を公開・アーカイブしたいとき
- `*LandingPage*` 系ツールが必要な文脈

## When NOT to use

- LP 以外のページ（プラン・サービス紹介、プロフィールリンク等）の編集
- シナリオ・配信設計（別スキル領域）

## Workflow

### 1. ヒアリングシート（1往復）

**`creator-hearing` スキルを起動する**（目的＝LP、追加シート＝`references/hearing-sheet.md`）。1往復の仕組み・状態ラベル・必須欄のゲート・原稿の扱い・顧客像の基準は creator-hearing が正。本スキルは creator-hearing に依存する（個人版は `skills/creator-hearing` を `skills/lp-builder` と同じ階層に置く。plugin 版は別プラグイン creator-hearing-plugin への依存とする。creator-context は creator-hearing の任意の入力で、同梱しない）。**creator-hearing が見つからないときは先に進まず、次のように伝えて止まる:「ヒアリング用のスキルが入っていません。次のどちらかで追加してから、もう一度お声がけください。プラグインで導入している場合: `/plugin install creator-hearing-plugin@mosh-plugins`。zip で導入している場合: ブラウザで https://skills.mosh.jp/creator-hearing.zip を開いて取得し、このスキルと同じ手順で追加してください」。ヒアリングを省いて LP を作らない。**

LP の必須欄は #1〜#3 と #4（ゴール種別のみ）。LP 固有の扱い:

- ゴール種別（リード獲得 / 告知 / 即決販売 / 高額・講座）は推奨に印を付けた4択で選ばせ、セクション数を決める
- **デザインの3点（プリセット / 画像スロット / 山場の材料）は、「おまかせ」と言われても必ず通す**（正本は `hearing-sheet.md` 手順4）。実績数字は同ファイルの #5・#8 に従う
- **原稿（セールスレター・既存LP文面）が渡されたら**、原稿から抽出して `要記入` だけを先頭に出す。原稿の文章は書き換えず、構成案で原文の配置先を示す（`hearing-sheet.md` の「原稿が渡された場合」）

### 2. テンプレート選択（任意）

テンプレートを使う場合：

1. `getCreatorLandingPageTemplates` でテンプレート一覧を取得する
2. ユーザーに候補を提示して選んでもらう
3. `getCreatorLandingPageTemplate` で選択したテンプレートの詳細を確認する

### 3. 構成案の承認（JSON 構築前の関門）

`references/section-plan.md` に従い、セクションごとに「ID・役割・見出し案・伝えること・素材/CTA先・レイアウト型・未確認/仮定」のテキスト表を出し、LP全体のストーリーを2〜3行添える。

- セクションIDは意味的に振る（`sec-01-hero` 形式）。この表がそのままIDマップになる
- **承認が出るまで JSON を組まない。LP も作成しない**（作成数上限を無駄に消費しない）。修正は表を直して再提示する
- 確認の言い回しは「この構成で MOSH に LP を作ってよいですか？」。ユーザー向け文言に JSON・ID・content・patch 等の内部用語を出さない（全ステップ共通。「構成」「セクション」「下書き」「公開」で言う）
- セクション数は `best-practices.md` の「ゴール別の構成目安」が正（本書に数値は再掲しない）。ユーザーが構成・規模を指定した場合はその指定を優先

### 4. LP 作成

構成案の承認後、`postCreatorLandingPage` で LP を新規作成する。

- テンプレート使用時: `bodyParams: { templateId: <選択したID> }`
- 白紙から作成: `bodyParams: { templateId: null }`

返却された `id` を以降のステップで使用する。

### 5. コンテンツ構築

1. **プリセットを選ぶ**: `references/presets.md` に従い、ヒアリングの #4 ゴールと #7 トーンから4〜5択＋おすすめ印で提示し、1つ決める（Step 3 の構成案と同時に提示してよい）
2. **プリセットの JSON を写す**: 選んだプリセットの `examples/catalog/<preset>.json` **だけ**を、章（1行＝1章）ごとに `offset`/`limit`（2章ずつ）で Read し、章単位でコピーして文言・画像・href を差し替える（全体を1回で Read すると打ち切られる。text/heading は tiptap なので文言は `text` ノードの `text` だけ差し替える。詳細は `examples/catalog/README.md`）。`styles` は変えない（例外はブランド色指定時の `brand`/`accent` 置換のみ。`presets.md`「選び方」）。章の増減は `best-practices.md` の「ゴール別の構成目安」に従い ID を振り直す
3. 構造ルールは `references/content-schema.md`、設計指針は `references/best-practices.md`。プリセットに無い部品を新しく作るときだけ、この2つを読んで組む。**日程章（`schedule`）・オートウェビナー（`autoWebinar`）などサービス連携のパーツは catalog に無いので、ゴールが告知・イベント等でこれらを使うときは `content-schema.md` の該当節（「`schedule` の仕様」「`autoWebinar` の仕様」）を必ず読む**

- **新規構築時**: 承認済み構成案の ID・見出し・メッセージをそのまま JSON に落とす。構成案に無いセクションを勝手に足さない
- **既存 LP の編集時**: patch の前に「何を・どう・なぜ変更するか」のサマリーをユーザーに提示し、確認を得てから実行する（新規構築時は Step 3 の承認がこれに相当する）
- 送信前に `best-practices.md` の**「セルフレビューチェックリスト」の節だけ**を読んで content を自己点検する

`patchCreatorLandingPageDraft` で下書きを更新する：

- `title`: LP タイトル
- `content`: 構造ルールに従った JSON オブジェクト
- `metaTitle` / `metaDescription`: SEO 用メタ情報

`content` は一度にすべて送信する。部分更新は不可。

### 6. レビュー

1. `getCreatorLandingPage` で現在の状態を取得する
2. 読み戻した content に対して `best-practices.md` の「セルフレビューチェックリスト」を再実行し、問題があれば修正して再保存する
3. ユーザーには **MOSH エディタで実物を見てもらい**、修正指示を構成案の ID に解決して該当セクションだけ書き換える（手順は `section-plan.md` の「保存後の修正ループ」）

### 7. 公開

公開前に `src` が空・ダミー URL・**プレースホルダ素材（`522545e4fca64345a420e594e1b2f6c1`）**の `image` 要素、`（要記入：` の仮置き、および href の差し替えトークン **`REPLACE_WITH_`** が残っていないか最終確認する（公開ページにも同じレンダラーが使われるため、そのまま公開すると空の画像枠や仮置きの文言がそのまま読者に表示される）。見つけた場合、画像は削除するか、実 URL の提供・MCP からのアップロード（`mcp-tools.md`「画像・動画のアップロード」）・編集画面からのアップロードのいずれかで解消する。`（要記入：` の文言は、ユーザーから得た実際の内容に書き換えるか、その要素・章を削除する。いずれも解消してから公開する。

ユーザーの承認を得てから `patchCreatorLandingPageStatus` で公開する。

```
bodyParams: { status: "PUBLISHED" }
```

## 必須ルール

`content` フィールドのトップレベルは必ず以下の形：

```json
{
  "page": { "styles": {}, "mobileStyles": {} },
  "elements": [ ... ]
}
```

すべての要素は次のフィールドを持つ：

- `id` — `"プレフィックス-識別子"` 形式。セクションは意味的 ID `sec-{2桁順}-{役割}`（例: `"sec-01-hero"`）、子要素は `"hd-01-hero"` のように親の順番・役割を継ぐ（`references/section-plan.md` の規約）
- `type` — PartType 名（`section` / `heading` / `text` / `button` など）
- `content` — テキスト内容。中身がない場合は空文字 `""`
- `styles` — スタイルオブジェクト。空 `{}` でもOK
- `mobileStyles` — 任意

セクション以外の要素は必ず `section` の `children` に入れる。`elements` 配列直下にトップレベルの `section` のみを並べる。

`text` / `heading` の `content` は通常プレーン文字列でよいが、**一文の中で部分的に太字・色・フォントサイズを変えたい場合**は tiptap のdoc構造をそのまま渡せる。詳細と実例は `references/content-schema.md` の「インライン装飾」の項を参照。

`image` の `src` に入れてよいのは**①ユーザー提供の実URL ②下書き専用のプレースホルダ素材** `https://mosh.jp/images/522545e4fca64345a420e594e1b2f6c1`（グレー地に IMAGE 表記、alt「仮画像（差し替え前提）」）の2つだけ。`src: ""`・ダミー URL・テンプレのサムネ等の無関係な画像は禁止。公開前に②を実画像へ差し替えるか要素を削除する（Step 7）。実画像の収穫は `references/hearing-sheet.md` #7、サイズ目安は `references/best-practices.md` の「画像の運用」。

**事実情報は、ユーザーから得たものだけを書く。** 日時・所要時間・会場・価格（「無料」を含む）・定員・実績の数字・お客様の声・保証・「勧誘しません」などの断定は、依頼・原稿・取得した商品情報に無ければ創作しない。無い項目は `（要記入：〇〇）` と書く（LP 本文に書く仮置きの書式で、ヒアリングシートの行ラベル `要記入` とは別物。例: `開催日時：（要記入：開催日と時間）`、`参加費：（要記入）`）。お客様の声・実績は、許諾済みの実物が無ければ中身を `（要記入：お客様の声を1件ご提供ください）` にするか章ごと省く（`best-practices.md` の構成表の注記も参照）。写真が仮画像のときも、本文で「この写真は実際の〜です」と断定しない。仮置きにした箇所は完了報告で**一覧にして伝える**（「汎用的な文言にしました」だけで済ませない）。

スタイルは要素別の**許可プロパティ内のみ**を使う。`height` / `margin` / `boxShadow` など UI に無いプロパティは設定しない（指定すると編集画面が崩れる）。要望されても設定できない旨と代替を伝える。詳細は `references/content-schema.md` の「スタイルの許可プロパティ・禁止プロパティ」の項を参照。

PartType / attributes の詳細は `references/content-schema.md`、ID 命名は `references/section-plan.md`。

## よくあるミス

追記前に、既存の節・`content-schema.md` でカバーできないか確認し、できるならそちらを強化する（この表は他で拾えない落とし穴だけ。詳細はリンク先が正）。

| NG | OK |
|---|---|
| `page` プロパティなし | トップレベルに `page` を必ず含める |
| `type: "header"` | `type: "heading"` を使う |
| `id` なし | 全要素に `"xxx-yyy"` 形式の `id` を付ける |
| `styles` なし | 全要素に `styles: {}` を付ける（空でもOK） |
| `content` なし | `section` でも `content: ""` を付ける |
| `content` に JSON 文字列を渡す | `content` フィールドには JSON オブジェクトを渡す |
| `elements` 直下にセクション以外の要素 | すべて `section` の `children` に入れる |
| `attributes.sectionType` に `"hero"` / `"cta"` | 許容値（`main` / `description` / `merit` / `faq` など14種）から選ぶ |
| `styles.background` に `"linear-gradient(...)"` 文字列 | グラデーションは `attributes.background` の構造化形（`type: "gradationColor"`）で設定（`content-schema.md` の「背景グラデーション」参照） |
| セクションの枠線を `border: "1px solid #..."` ショートハンドで指定 | 辺別プロパティ（`borderTopStyle` / `borderTopWidth` / `borderTopColor` 等）で指定する（ショートハンドはパネルに反映されず編集不能になる） |
| `text` / `heading` の `content` に `null` やオブジェクト（`json`キー無し） | 内容が無ければ `content: ""`（**クラッシュ防止**。`references/content-schema.md`の「絶対に避けること」参照） |
| `schedule` の `content` にオブジェクトを渡す | プレーン文字列のみ（tiptap不可。**クラッシュ防止**） |
| 日程が主役の LP で `schedule` 要素を作らず、日程を text・カードで手書き（日付も創作） | `schedule` 要素を必ず置く（`serviceId` 未指定でも可）。詳細は `content-schema.md`「`schedule` の仕様」 |
| `id` に `wrap-` / `row-` / `num-` / `txt-` など表に無い接頭辞 | 接頭辞は type で決まる。補助要素の付け方は `section-plan.md`「セクション ID の規約」 |
| `heading` の `attributes.level` が `"1"`〜`"4"` の範囲外 | 範囲内の値のみ使う（**クラッシュ防止**） |
| `image-carousel` の `attributes.images` が非配列 or `{src,alt}`等のオブジェクト配列 | URL文字列の配列のみ（**クラッシュ防止**。そもそも実験的パーツにつき新規作成不可） |
| `video` の `attributes.src` に外部URL(GCS/Vimeo等の直リンク)をそのまま設定 | `video` は `moshVideoId` が無いと編集画面が**クラッシュ**するためMCPからは新規作成不可。編集画面からのアップロードを案内する。YouTubeなら`youtubeVideo`で代替可（`content-schema.md`の「`video`の仕様」参照） |

## エラーが返ったとき

操作が業務ルールに反する場合（作成・公開できる LP 数の上限に達しているなど）は、理由を示すエラーが返る。再試行せず、返ってきた内容に沿って原因と対処をユーザーへ平易な日本語で伝える（内部的なコードや用語は出さない）。

## References

| ファイル | 内容 |
|---|---|
| `creator-hearing`（別スキル） | ヒアリングの進め方・状態ラベル・共通3ブロック・原稿の扱い（正本） | Step 1 で必ず（スキルを起動する） |
| `references/hearing-sheet.md` | LP 用のヒアリング追加項目（共通3ブロックは `creator-hearing` が正。ここは CV先の補足、ゴール種別（4択）、証拠、トーン・素材（画像スロット表：収穫→依頼）、#8 見せ場の材料（配布物一覧・ロゴ・表現の禁止事項）、デザインの3点、提示フォーマット、LP 固有の禁止） |
| `references/section-plan.md` | 構成案ゲート（JSON 構築前の承認表の書式、意味的セクション ID 規約、保存後の修正ループ） |
| `references/content-schema.md` | PartType 一覧、ID プレフィックス表、`attributes` 詳細（`sectionType` の許容値を含む）、特殊パーツの仕様（`schedule` / `autoWebinar` / `video` / `image-carousel` / `countdown`。使うときにその節を読む）、セクション+children の例 |
| `references/best-practices.md` | 構成計画（ゴール別のセクション数目安・推奨構成と `sectionType` 対応・推奨 ID）、テキストの具体性、余白の原則、表現テクニック、モバイルファースト、CTA 規約、画像の運用、セルフレビューチェックリスト |
| `references/presets.md` | 6プリセット（配色＋骨格＋部品）の定義と選び方。Step 5 の最初に読む |
| `references/mcp-tools.md` | 各 MCP ツールのパラメータ仕様。表示期限（`expirationType`）・一覧取得のフィルタ・title/content 以外の patch 項目を扱うときに読む |
| `references/photo-patterns.md` | 写真・画像の見せ方40型（ヒーロー／本文の中／並べる／信頼／場所・手順・料金／写真がないとき。LP機能で作れるか ○△× 付き）。構成案の「レイアウト型」に写真の型を添えるとき、素材が足りない枠の代替を選ぶときに読む |
| `examples/catalog/<preset>.json` | プリセット6本の LP 全体 JSON。選んだ1本だけ Read し、章単位で写す。索引は `examples/catalog/README.md` |
| `examples/full-landing-page.json` | 構造の最小例（メイン / 特徴 / CTA / フッター）。プリセットを使わず白紙から組むときの参照 |
