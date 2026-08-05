# LPコンテンツ構造リファレンス

`patchCreatorLandingPageDraft` の `content` フィールドに渡すJSONオブジェクトの詳細仕様。

## トップレベル構造

```json
{
  "page": { ... },
  "elements": [ ... ]
}
```

## `page` オブジェクト

```json
{
  "page": {
    "styles": {
      "background": "#ffffff",
      "padding": "0 0 0 0"
    },
    "mobileStyles": {
      "padding": "0 0 0 0"
    }
  }
}
```

## 要素の共通フィールド

```json
{
  "id": "sec-main",
  "type": "section",
  "content": "",
  "styles": {},
  "mobileStyles": {}
}
```

- `id` — 必須。`"文字列-文字列"` 形式（例: `"sec-001"`, `"hd-main"`）
- `type` — 必須。下記 PartType 一覧から選ぶ
- `content` — 必須。セクションなど内容がない場合は空文字 `""`
- `styles` — 必須。空オブジェクト `{}` でもOK
- `mobileStyles` — 任意

## ID命名規約

| プレフィックス | type |
|---|---|
| `sec-` | section |
| `hd-` | heading |
| `tx-` | text |
| `btn-` | button |
| `img-` | image |
| `vid-` | video |
| `yt-` | youtubeVideo |
| `sep-` | separator |
| `cd-` | countdown |
| `sch-` | schedule |
| `car-` | image-carousel |
| `aw-` | autoWebinar |

## PartType 一覧

| type | 用途 |
|---|---|
| `section` | セクション（`children` に他要素を持つ） |
| `heading` | 見出し（`header` は不可） |
| `text` | テキスト |
| `button` | ボタン |
| `image` | 画像 |
| `video` | 動画 |
| `youtubeVideo` | YouTube動画 |
| `separator` | 区切り線 |
| `countdown` | カウントダウン |
| `schedule` | スケジュール |
| `image-carousel` | 画像カルーセル（**実験的パーツにつき新規作成不可**。既存要素は維持。詳細は下記「`image-carousel` の仕様」を参照） |
| `autoWebinar` | 自動ウェビナー |

## `attributes` フィールド（任意）

```json
"attributes": {
  "level": "1",
  "href": "https://...",
  "target": "_blank",
  "rel": "noopener",
  "src": "https://...",
  "alt": "説明",
  "background": {
    "type": "color",
    "color": "#000000",
    "image": "https://..."
  },
  "sectionType": "main",
  "showDesktop": true,
  "showMobile": true
}
```

- `level` — `heading` のレベル: `"1"` | `"2"` | `"3"` | `"4"`
- `href` / `target` / `rel` — `button` のリンク設定
- `src` / `alt` — `image` の設定
- `background.type` — `"color"` | `"image"`
- `sectionType` — `section` の分類メタ情報。下記の許容値以外（`"hero"` など）は無効値なので使わない

### type 別の必須 attributes（構造上は任意・機能上は必須）

`attributes` はスキーマ上すべてのキーが任意だが、以下の type では省略すると表示が欠落する（ボタンがリンク切れになる、画像/動画が表示されない等）。実質的に必須として扱う。

| type | 必須 | 補足 |
|---|---|---|
| `image` | `src` | `alt` も推奨（未設定でもクラッシュはしないがアクセシビリティ上推奨） |
| `button` | `href` | `target` も推奨（新規タブで開くか指定） |
| `video` | `src` | |
| `youtubeVideo` | `src` | |
| `image-carousel` | `images` | **URL文字列の配列**必須（詳細・実験的パーツである旨は下記「`image-carousel` の仕様」を参照） |
| `heading` | — | `level` は省略可。省略時は `"2"` 相当として扱われる |

**リンク先・画像・動画などの実値がユーザーから与えられていない場合、ダミー値（`https://example.com/...` のような架空URLや `#` 等）を捏造して保存してはいけない。** ユーザーに確認するか、その旨を明示した上で該当要素の追加を保留する。

### `sectionType` の許容値（14種）

`empty` / `site-header` / `main` / `countdown` / `gallery` / `description` / `profile` / `merit` / `contact` / `schedule` / `faq` / `form` / `footer` / `float-button`

編集画面ではこのうち以下の8種のみがセクション追加パネルに並ぶ（他はテンプレート未提供）。組み立てる際はこの8種から選ぶ。

| 値 | 編集画面の表示名 | 用途 |
|---|---|---|
| `main` | メイン | ファーストビュー（キャッチコピー＋メインビジュアル） |
| `description` | 説明・訴求 | 課題提起・解決策・サービス説明 |
| `profile` | プロフィール | 講師・運営者の紹介 |
| `merit` | メリット | 特徴・受講後の変化 |
| `countdown` | カウントダウン | 締切までの残り時間 |
| `schedule` | 日程表示 | 開催日程 |
| `faq` | よくある質問 | Q&A |
| `footer` | サイトフッター | 特商法表記・注意事項 |

`sectionType` は編集画面でのセクション分類に使われるメタ情報で、レンダリング結果（公開LPの表示）には影響しない。ただし無効値を入れると分類が壊れるため、必ず上記の値を使う。CTA のような専用の値は無いので、CTA セクションは内容に応じて `main` / `description` などを選ぶ。

## `countdown` の仕様

```json
{
  "id": "cd-001",
  "type": "countdown",
  "content": "",
  "styles": { "padding": "24px 0 0 0", "background": "#ffffff" },
  "attributes": { "size": "small" }
}
```

- `size` — 任意。`"small"` | `"large"`。デフォルトは`"large"`。`"large"`は日/時間/分/秒それぞれにラベル付きの大きな箱型、`"small"`はコンパクトな横並び表示。
- `color` — セクションテンプレートの一部に含まれることがあるが、現状のレンダラーは参照しておらず表示に影響しない。指定しても効果はないので新規に組み立てる際は不要。
- **対象日時はこの要素の`attributes`では指定しない。** カウントダウンの対象日時・残り時間は常にLP自体の表示期限設定（`expirationType` / `expireAt` / `relativeExpiration`。`mcp-tools.md`参照）から算出される。`expirationType`が`"NONE"`（無期限）の場合、カウントダウンはダッシュ（`-`）表示になる。
- `content`は空文字`""`のままでよい（表示テキストは持たない）。
- 期限到達時（日/時間/分/秒すべて`0`になった瞬間）は、この`countdown`要素単体が「0」表示になるのではなく、**公開LPのコンテンツ全体**が期限切れ画面（`expirationAction`次第で非表示メッセージ or リダイレクト。`mcp-tools.md`参照）に差し替わる。

## `image-carousel` の仕様

**実験的パーツにつき新規作成は不可。** 通常の編集画面には無く、一部ユーザーのみが利用できるパーツのため、ユーザーが画像カルーセルを希望した場合（実在する画像 URL を提示された場合を含む）でも、**image-carousel 要素は新規に作成せず、実験的パーツで現状お作りできない旨を案内する。** 代替として、`image` 要素を複数並べる、または編集画面から直接追加してもらうよう提案する。

**既存の image-carousel 要素は消さない。** 編集画面から既にカルーセルを追加済みの LP を MCP で編集する場合、その `image-carousel` 要素（と `attributes.images`）は実データなので、他の変更を保存する際にそのまま維持する。新規作成しないことと、既存データを保持することは別の話であり、既存要素を巻き込んで削除・上書きしないこと。

以下は実データ構造の参考情報（新規作成はしないが、既存要素を維持する際の形を把握するため）。

```json
{
  "id": "car-001",
  "type": "image-carousel",
  "content": "",
  "styles": {},
  "attributes": {
    "images": ["https://...", "https://..."]
  }
}
```

- `attributes.images` — **URL文字列の配列**（例: `["https://example.com/a.jpg", "https://example.com/b.jpg"]`）。`{ "src": "...", "alt": "..." }` のようなオブジェクトの配列にしてはいけない。レンダラーは各要素を画像URLの文字列としてそのまま扱うため、配列でも要素がオブジェクトだとクラッシュする（`image` type の `src`/`alt` 属性と混同しないこと）。

## `text` / `heading` のインライン装飾（一文の中で部分的に太字・色・サイズを変える）

`content` には単純な文字列だけでなく、tiptapのdoc構造をそのまま渡せる。1つの`text`/`heading`要素内で、一部の文言だけ太字・色・フォントサイズを変えたい場合はこちらを使う（プレーン文字列を渡した場合、要素全体が`styles`で指定した単一の書式になり、部分的な装飾はできない）。

```json
{
  "id": "tx-001",
  "type": "text",
  "styles": { "padding": "0 35px 20px" },
  "content": {
    "type": "tiptap",
    "json": {
      "type": "doc",
      "content": [
        {
          "type": "paragraph",
          "attrs": { "textAlign": null, "lineHeight": "" },
          "content": [
            {
              "type": "text",
              "text": "通常の文言はこの色・太さで表示され、",
              "marks": [
                { "type": "textStyle", "attrs": { "color": "#212529", "fontSize": "18px", "fontFamily": "", "mobileFontSize": null, "backgroundColor": "" } }
              ]
            },
            {
              "type": "text",
              "text": "ここだけ太字＋オレンジ色",
              "marks": [
                { "type": "textStyle", "attrs": { "color": "#e95f3f", "fontSize": "18px", "fontFamily": "", "mobileFontSize": null, "backgroundColor": "" } },
                { "type": "bold" }
              ]
            },
            { "type": "hardBreak", "marks": [{ "type": "textStyle", "attrs": { "color": "#212529", "fontSize": "18px", "fontFamily": "", "mobileFontSize": null, "backgroundColor": "" } }] },
            {
              "type": "text",
              "text": "改行後、ここだけ小さい注釈サイズ",
              "marks": [
                { "type": "textStyle", "attrs": { "color": "#212529", "fontSize": "10px", "fontFamily": "", "mobileFontSize": null, "backgroundColor": "" } }
              ]
            }
          ]
        }
      ]
    }
  }
}
```

- `content.json.content[].content[]` は「テキストラン」(`type: "text"`) と「改行」(`type: "hardBreak"`) のノードが並ぶ配列。
- `text`ノード — `text`(表示文字列)と`marks`を持つ。`marks`に`textStyle`（color/fontSize等）や`bold`を個別に付けることで、ラン単位で書式を変えられる。
- `hardBreak`ノード — `\n`ではなく明示的な改行専用ノード。`text`プロパティは持たない独立した要素。直後のテキストと書式の連続性を保つため、同じ`marks`を付けておくとよい。
- 要素レベルの`styles`（padding等）は引き続き有効。文字の色・サイズはmarks側が優先される。
- 保存後に`getCreatorLandingPage`で取得すると、プレーン文字列で送った`text`/`heading`もこの形式に自動変換されて返ってくる（正引きの参考にできる）。

## `section` + `children` の例

```json
{
  "id": "sec-001",
  "type": "section",
  "content": "",
  "styles": { "padding": "64px 24px" },
  "attributes": {
    "background": { "type": "color", "color": "#ffffff" }
  },
  "children": [
    {
      "id": "hd-001",
      "type": "heading",
      "content": "見出しテキスト",
      "styles": { "color": "#000000", "fontSize": "30px" },
      "attributes": { "level": "2" }
    },
    {
      "id": "tx-001",
      "type": "text",
      "content": "本文テキスト",
      "styles": { "color": "#666666", "lineHeight": "1.8" }
    }
  ]
}
```

## スタイルの許可プロパティ・禁止プロパティ

編集画面（UI）は要素ごとに設定できるスタイルプロパティが決まっている。UI に用意されていないスタイルプロパティを `styles` に指定すると、編集画面で表示が崩れる原因になる。要素別に、下記の**使ってよいプロパティだけ**を使う。

### 使ってよいスタイルプロパティ（要素別）
- `section`: `padding` / `background` / `borderRadius` / `width` / `gap` / `flexWrap` / `display`

#### PC/SPでレイアウト方向(横並び・縦並び)を変える方法

`section` の子要素を「PCは横並び、SPは縦並び」のように**デバイス別に切り替える**には、`flexDirection` ではなく `display` プロパティを使う。編集画面の「レイアウトの方向」トグルが実際に書き込む値もこの形式。

編集画面での対応箇所(いずれも実際にUIから操作可能。ただし「レイアウトの方向」はBetaBadge付きのベータ機能):
- 「レイアウトの方向」(垂直/水平タブ) → `display` を書き込む
- 「要素の間隔」(数値入力) → `gap` を書き込む。**「レイアウトの方向」を水平にした場合のみ画面に表示される**
- 「折り返し」(スイッチ) → `flexWrap` を書き込む。同じく水平時のみ表示

`gap` / `flexWrap` は独立した常設プロパティではなく、`display: "flex"` にして初めてUI上に現れる付随プロパティである点に注意。

```json
{
  "id": "sec-row-001",
  "type": "section",
  "content": "",
  "styles": { "display": "flex", "gap": "16px", "flexWrap": "nowrap", "padding": "0 0 0 0" },
  "mobileStyles": { "display": "block" },
  "children": [ ... ]
}
```

- `styles.display: "flex"` — PC（デスクトップ）で子要素を横並びにする。
- `mobileStyles.display: "block"` — SP（モバイル）で縦並び（通常のブロック要素）に切り替える。**省略すると自動で縦並びにはならず、PC側の`display`をそのまま引き継ぐ**（`mobileStyles`は指定したキーだけがPC側の値を上書きし、未指定のキーは`styles`の値にフォールバックするため）。PC/SPで方向を変えたい場合は必ず`mobileStyles.display`を明示する。
- 横並びにする子要素側には `width`（例: `"48%"` / `"31%"` / `"18%"` など）を指定し、縦並びにする際は子要素の `mobileStyles.width` を `"100%"` にする。`width` はPC/SPで自動的に変わらないため、こちらも明示が必要。
- `display: "block"`（縦並び）の状態では `gap` は効かない（`gap` はflex/grid時のみ有効なCSS仕様）。縦並び時に要素間の余白を確保したい場合は、各子要素の `padding`（例: `mobileStyles.padding: "0 0 24px 0"`）で表現する。
- `flexDirection` は現在の編集画面では書き込まれない（過去のテンプレート互換のために読み取りだけされる古い仕組み）ため、新規に組み立てる際は使わない。

- `text` / `heading`: `color` / `fontFamily` / `fontSize` / `lineHeight` / `textAlign` / `padding`
- `button`: `background` / `color` / `borderRadius` / `width` / `padding` / `textAlign`
  - 上記は編集画面のプロパティパネルで**変更できる**プロパティ。これに加えて、UI でボタンを挿入すると変更不可の固定デフォルト `display: "inline-block"` / `cursor: "pointer"` / `fontSize: "14px"` / `fontWeight: "bold"` が常に付与される（UI では編集手段がないため値はこの固定値のまま）。ボタンを組み立てる際はこれら固定デフォルトも含めて出力し、`fontSize` は `14px` 以外にしない。

### 明示的に設定しないプロパティ
- `height`（特に `section` / `button`。高さは中身と `padding` で決める）
- `margin`（余白は `section` の `padding` で表現する）
- `boxShadow`
- 上記の許可リストに無い任意の CSS

特に `height` を指定すると編集画面でレイアウトが破綻する（公開ページは正常でも編集画面が崩れる）ため、高さは中身と `padding` で決める。ユーザーがこれらのスタイル（影・高さ固定・外側余白など）を要望した場合は、**設定できない旨と理由を伝え、許可プロパティ内の代替（`padding`・`background`・`borderRadius` 等）を提案**する。

## フォントの制約（fontFamily・fontSize）

`fontFamily` と `fontSize`（要素の `styles`、およびインライン装飾の tiptap `textStyle` マーク）は、編集画面（UI）で選べる固定値のみを使う。これ以外の値は UI のドロップダウンと一致せず、意図しない表示やフォント未ロードになる。

### fontFamily（4種のみ・引用符込みの文字列）
- `'Noto Sans JP'` / `'Sawarabi Gothic'` / `'M PLUS Rounded 1c'` / `'Noto Serif JP'`

### fontSize（固定スケール・px）
- `10px` / `12px` / `14px` / `16px` / `18px` / `20px` / `24px` / `30px` / `36px` / `48px`
- 既定は本文 `14px`・見出し `18px`。`rem` など固定スケール外の値は使わない。

## 絶対に避けること（クラッシュ防止）

以下は本ドキュメントの各所で個別に述べている制約のうち、**守らないとレンダラーがクラッシュし編集画面・プレビュー・公開ページのいずれかが開けなくなる**ものを1箇所に集約したもの。特に重要なので必ず守ること。

- **`text` / `heading` の `content` に `null` や、`json` キーを持たないオブジェクトを渡さない。** 内容が無い場合は空文字 `""` にする（「要素の共通フィールド」参照）。`content.json` を参照してレンダリングするため、`null` やオブジェクトだと LP 全体が描画できなくなる。
- **`schedule` の `content` は常にプレーン文字列のみ。** `null` やオブジェクト（tiptap 構造を含む）を渡さない。`text` / `heading` と異なり部分装飾の仕組みが無く、素の文字列として扱われるため、他の型で描画できてもクラッシュする。
- **`heading` の `attributes.level` は `"1"`〜`"4"` の範囲のみ。** 範囲外の値を入れると編集画面が開けなくなるおそれがある（「`attributes` フィールド」参照）。
- **`image-carousel` の `attributes.images` は必ず URL 文字列の配列。** 配列でない値や、配列でも要素が文字列でない場合（`{ "src": ..., "alt": ... }` 等のオブジェクト配列）はクラッシュする（「`image-carousel` の仕様」参照）。
