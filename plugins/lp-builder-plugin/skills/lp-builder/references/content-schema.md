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
  "id": "sec-hero",
  "type": "section",
  "content": "",
  "styles": {},
  "mobileStyles": {}
}
```

- `id` — 必須。`"文字列-文字列"` 形式（例: `"sec-001"`, `"hd-hero"`）
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
| `image-carousel` | 画像カルーセル |
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
  "sectionType": "hero",
  "showDesktop": true,
  "showMobile": true
}
```

- `level` — `heading` のレベル: `"1"` | `"2"` | `"3"` | `"4"`
- `href` / `target` / `rel` — `button` のリンク設定
- `src` / `alt` — `image` の設定
- `background.type` — `"color"` | `"image"`

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
- `section`: `padding` / `background` / `borderRadius` / `width` / `gap` / `flexWrap`
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
