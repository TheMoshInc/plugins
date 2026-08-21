---
name: lp-builder
description: MOSH (taiyaki) のランディングページ (LP) を taiyaki MCP 経由で作成・編集・公開するスキル。要件ヒアリング → テンプレート選択 → コンテンツJSON構築 → 下書き保存 → レビュー → 公開、というワークフローを対話で進める。「LP作りたい」「ランディングページ作って」「LP公開して」「LPの下書き更新」「lp-builder」など、MOSH の LP に関する作成・編集・公開リクエストで使用する。
---

# MOSH LP ビルダー

## Overview

taiyaki MCP ツール (`mcp__taiyaki__*LandingPage*`) を使って、MOSH クリエイターのランディングページを作成・編集・公開する。テンプレートから起こすか白紙から作るかを選び、`content` フィールドに渡す JSON を組み立て、下書き保存・確認のうえで公開する。

`content` の構造には固有のルール（必須フィールド、ID命名規約、PartType）があるため、構築前に `references/content-schema.md` を必ず参照する。

## When to use

- ユーザーが新しい LP を作りたいと依頼したとき
- 既存 LP の内容を更新したいとき
- LP を公開・アーカイブしたいとき
- `mcp__taiyaki__*LandingPage*` 系ツールが必要な文脈

## When NOT to use

- LP 以外のページ（プラン・サービス紹介、プロフィールリンク等）の編集
- シナリオ・配信設計（別スキル領域）

## Workflow

### 1. 要件ヒアリング

ユーザーに以下を確認する：

- LP の目的（集客、販売、告知など）
- ターゲット（誰に向けたページか）
- 掲載したい内容（テキスト、画像、動画など）
- テンプレートを使うかどうか

### 2. テンプレート選択（任意）

テンプレートを使う場合：

1. `mcp__taiyaki__getCreatorLandingPageTemplates` でテンプレート一覧を取得する
2. ユーザーに候補を提示して選んでもらう
3. `mcp__taiyaki__getCreatorLandingPageTemplate` で選択したテンプレートの詳細を確認する

### 3. LP 作成

`mcp__taiyaki__postCreatorLandingPage` で LP を新規作成する。

- テンプレート使用時: `bodyParams: { templateId: <選択したID> }`
- 白紙から作成: `bodyParams: { templateId: null }`

返却された `id` を以降のステップで使用する。

### 4. コンテンツ構築

`references/content-schema.md` を読み込み、構造ルールに沿った JSON を組み立てる。設計指針は `references/best-practices.md` を参照。完成イメージは `examples/full-landing-page.json` を参照。

- **新規構築時**: 組み立ての前に `best-practices.md` の「構成計画」に従い、LP 全体のストーリーとセクション構成を先に決める（構成をお任せされたフル構築なら最低10セクション。ユーザーが構成・規模を指定した場合はその指定を優先）
- **既存 LP の編集時**: patch の前に「何を・どう・なぜ変更するか」のサマリーをユーザーに提示し、確認を得てから実行する（新規構築時はこの確認は不要）
- 送信前に `best-practices.md` の「セルフレビューチェックリスト」で content を自己点検する

`mcp__taiyaki__patchCreatorLandingPageDraft` で下書きを更新する：

- `title`: LP タイトル
- `content`: 構造ルールに従った JSON オブジェクト
- `metaTitle` / `metaDescription`: SEO 用メタ情報

`content` は一度にすべて送信する。部分更新は不可。

### 5. レビュー

1. `mcp__taiyaki__getCreatorLandingPage` で現在の状態を取得する
2. 読み戻した content に対して `best-practices.md` の「セルフレビューチェックリスト」を再実行し、問題があれば修正して再保存する
3. セクション構成をユーザーに提示して確認を取る
4. 修正が必要なら Step 4 に戻る

### 6. 公開

公開前に `src` が空・ダミー URL の `image` 要素が残っていないか最終確認する（公開ページにも同じレンダラーが使われるため、そのまま公開すると空の画像枠が表示される）。見つけた場合は削除するか、実 URL の提供・編集画面からのアップロードを案内して解消してから公開する。

ユーザーの承認を得てから `mcp__taiyaki__patchCreatorLandingPageStatus` で公開する。

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

- `id` — `"プレフィックス-識別子"` 形式（例: `"sec-001"`, `"hd-main"`）
- `type` — PartType 名（`section` / `heading` / `text` / `button` など）
- `content` — テキスト内容。中身がない場合は空文字 `""`
- `styles` — スタイルオブジェクト。空 `{}` でもOK
- `mobileStyles` — 任意

セクション以外の要素は必ず `section` の `children` に入れる。`elements` 配列直下にトップレベルの `section` のみを並べる。

`text` / `heading` の `content` は通常プレーン文字列でよいが、**一文の中で部分的に太字・色・フォントサイズを変えたい場合**は tiptap のdoc構造をそのまま渡せる。詳細と実例は `references/content-schema.md` の「インライン装飾」の項を参照。

`image` 要素は実在する画像 URL が提供された場合のみ作成する。`src: ""` のプレースホルダー配置もダミー URL の捏造も禁止。画像未提供時はテキスト・背景色で構成を組み、編集画面からのアップロードを案内する（詳細は `references/best-practices.md` の「画像の運用」）。

スタイルは要素別の**許可プロパティ内のみ**を使う。`height` / `margin` / `boxShadow` など UI に無いプロパティは設定しない（指定すると編集画面が崩れる）。要望されても設定できない旨と代替を伝える。詳細は `references/content-schema.md` の「スタイルの許可プロパティ・禁止プロパティ」の項を参照。

PartType / ID 命名 / attributes の詳細は `references/content-schema.md`。

## よくあるミス

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
| `heading` の `attributes.level` が `"1"`〜`"4"` の範囲外 | 範囲内の値のみ使う（**クラッシュ防止**） |
| `image-carousel` の `attributes.images` が非配列 or `{src,alt}`等のオブジェクト配列 | URL文字列の配列のみ（**クラッシュ防止**。そもそも実験的パーツにつき新規作成不可） |
| `video` の `attributes.src` に外部URL(GCS/Vimeo等の直リンク)をそのまま設定 | `video` は `moshVideoId` が無いと編集画面が**クラッシュ**するためMCPからは新規作成不可。編集画面からのアップロードを案内する。YouTubeなら`youtubeVideo`で代替可（`content-schema.md`の「`video`の仕様」参照） |

## エラーが返ったとき

操作が業務ルールに反する場合（作成・公開できる LP 数の上限に達しているなど）は、理由を示すエラーが返る。再試行せず、返ってきた内容に沿って原因と対処をユーザーへ平易な日本語で伝える（内部的なコードや用語は出さない）。

## References

| ファイル | 内容 |
|---|---|
| `references/content-schema.md` | PartType 一覧、ID 命名規約、`attributes` 詳細（`sectionType` の許容値を含む）、セクション+children の例 |
| `references/best-practices.md` | 構成計画（最低10セクション・推奨構成と `sectionType` 対応）、テキストの具体性、余白の原則、表現テクニック、モバイルファースト、CTA 規約、画像の運用、セルフレビューチェックリスト |
| `references/mcp-tools.md` | 各 MCP ツールのパラメータ仕様 |
| `examples/full-landing-page.json` | メイン / 特徴 / CTA を含む構造の最小例（実際の LP は `best-practices.md` の構成計画に従い最低10セクションで組む） |
