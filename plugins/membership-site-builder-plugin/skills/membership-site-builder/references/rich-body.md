# 本文の書き方（見出し・太字・箇条書き・リンク・Vimeo 埋め込み）

コンテンツの本文（`body`）は、管理画面のエディタと同じ **Tiptap JSON を文字列にしたもの**を送ると、見出しや太字などの装飾付きで会員に表示される。プレーンテキストで送った場合は、改行が段落になるだけで装飾は付かない。

完全な例は [../examples/article-rich-body.json](../examples/article-rich-body.json)。

## 形

- `body` は JSON の**文字列**（`"{\"type\":\"doc\",\"content\":[...]}"`）。オブジェクトのまま送らない
- 先頭は必ず `{"type":"doc","content":[...]}`。`{` で始まる本文だけが Tiptap JSON として読まれる
- 表示テキストは 30,000 文字まで（JSON 全体は 300,000 文字まで）

## 使える要素（これ以外は使わない）

**エディタが知らない要素名が 1 つでも入ると、本文全体が空白で表示される。** 下の表にある名前だけを使う。

| やりたいこと | ノード / マーク | 書き方 |
|---|---|---|
| 段落 | `paragraph` | `{"type":"paragraph","content":[{"type":"text","text":"…"}]}`。空行は `{"type":"paragraph"}` |
| 見出し | `heading` | `"attrs":{"level":2}`。`level` は 1〜3 だけ |
| 箇条書き | `bulletList` > `listItem` > `paragraph` | `listItem` の中は必ず `paragraph` で包む |
| 番号付きリスト | `orderedList` > `listItem` > `paragraph` | 同上 |
| チェックリスト | `taskList` > `taskItem` > `paragraph` | `taskItem` に `"attrs":{"checked":false}` |
| 引用 | `blockquote` > `paragraph` | — |
| 区切り線 | `horizontalRule` | 中身なし |
| 段落内の改行 | `hardBreak` | 段落の `content` に入れる |
| Vimeo 埋め込み | `vimeoEmbed` | `"attrs":{"src":"https://player.vimeo.com/video/{動画ID}"}`。限定公開は `?h={ハッシュ}` を付ける。`vimeo.com/{ID}` や `vimeo.com/{ID}/{ハッシュ}` で渡されたら、この形に直す。ハッシュは英小文字と数字だけ。player.vimeo.com の形でない `src`（YouTube 等を含む）はエラーにならず埋め込みだけが表示されないので、保存前に形を確かめる |
| 太字・斜体・下線・取り消し線 | マーク `bold` / `italic` / `underline` / `strike` | `{"type":"text","marks":[{"type":"bold"}],"text":"…"}` |
| リンク | マーク `link` | `"marks":[{"type":"link","attrs":{"href":"https://…","target":"_blank"}}]` |
| 文字の背景色 | マーク `highlight` | `"attrs":{"color":"#FEF08A"}` |

ほかに `codeBlock`・マーク `code` / `textStyle` もあるが、クリエイター向けの本文では使わない。

**画像（`image`）は本文に入れない。** 本文用に画像を上げる手段を確かめていないため、画像の差し込みは管理画面のエディタで行うよう案内する。

## 書くときに守ること

- **既存の本文を直すときは、取得した JSON を土台に、該当のノードだけを書き換える。** 管理画面で入れた表（table）など、上の表に無い要素が既にあってもそのまま残す（新しくは作らない）。一から組み直すと、管理画面で入れた画像・埋め込み・書式が消える。取得した本文が JSON でない（旧サイトの HTML 断片やプレーンテキスト）なら、作り直してよいかをユーザーに確認する
- `text` は空文字にしない（空の段落は `content` を省く）。空の `text` があると、表に無い要素と同じく本文全体が空白で表示される
- 本文の JSON はユーザーに見せない。保存後は「見出し 2 つと箇条書きを入れました」のように内容で伝える
- 保存後は `getCreatorMembershipSiteContent` で読み戻し、送った JSON がそのまま入っていることを確かめる
