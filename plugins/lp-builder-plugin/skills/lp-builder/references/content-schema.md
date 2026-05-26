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
      "styles": { "color": "#000000", "fontSize": "2rem" },
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
