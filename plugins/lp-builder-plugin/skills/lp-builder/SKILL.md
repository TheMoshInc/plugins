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

- LP 以外のページ（サービス紹介、プロフィールリンク等）の編集
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

`mcp__taiyaki__patchCreatorLandingPageDraft` で下書きを更新する：

- `title`: LP タイトル
- `content`: 構造ルールに従った JSON オブジェクト
- `metaTitle` / `metaDescription`: SEO 用メタ情報

`content` は一度にすべて送信する。部分更新は不可。

### 5. レビュー

1. `mcp__taiyaki__getCreatorLandingPage` で現在の状態を取得する
2. セクション構成をユーザーに提示して確認を取る
3. 修正が必要なら Step 4 に戻る

### 6. 公開

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
| `video` の `attributes.src` に外部URL(GCS/Vimeo等の直リンク)をそのまま設定 | `video` は `moshVideoId` が無いと編集画面が**クラッシュ**するためMCPからは新規作成不可。編集画面からのアップロードを案内する。YouTubeなら`youtubeVideo`で代替可（`content-schema.md`の「`video`の仕様」参照） |
| `attributes.sectionType` に `"hero"` / `"cta"` | 許容値（`main` / `description` / `merit` / `faq` など14種）から選ぶ |
| `text` / `heading` の `content` に `null` やオブジェクト（`json`キー無し） | 内容が無ければ `content: ""`（**クラッシュ防止**。`references/content-schema.md`の「絶対に避けること」参照） |
| `schedule` の `content` にオブジェクトを渡す | プレーン文字列のみ（tiptap不可。**クラッシュ防止**） |
| `heading` の `attributes.level` が `"1"`〜`"4"` の範囲外 | 範囲内の値のみ使う（**クラッシュ防止**） |
| `image-carousel` の `attributes.images` が非配列 or `{src,alt}`等のオブジェクト配列 | URL文字列の配列のみ（**クラッシュ防止**。そもそも実験的パーツにつき新規作成不可） |

## エラーが返ったとき

操作が業務ルールに反する場合（作成・公開できる LP 数の上限に達しているなど）は、理由を示すエラーが返る。再試行せず、返ってきた内容に沿って原因と対処をユーザーへ平易な日本語で伝える（内部的なコードや用語は出さない）。

## References

| ファイル | 内容 |
|---|---|
| `references/content-schema.md` | PartType 一覧、ID 命名規約、`attributes` 詳細（`sectionType` の許容値を含む）、セクション+children の例 |
| `references/best-practices.md` | 推奨セクション構成（メイン → 課題提起 → 解決策 …）と対応する `sectionType`、モバイル対応 |
| `references/mcp-tools.md` | 各 MCP ツールのパラメータ仕様 |
| `examples/full-landing-page.json` | メイン / 特徴 / CTA を含む完成例 |
