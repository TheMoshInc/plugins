# プリセット JSON（5トーン）

`references/presets.md` の各プリセットに対応する、そのまま写せる LP 全体の JSON。

使い方: プリセットを1つ選び、**そのファイルだけ** Read する。章（`sec-NN-*`）単位でコピーし、文言・画像・href を差し替える。**styles は変えない**（ブランド色指定時の `brand`/`accent` 置換のみ例外）。章を減らす場合は `best-practices.md` の「ゴール別の構成目安」に従い、ID の順番を振り直す。

| ファイル | プリセット | ゴール | 章数 |
|---|---|---|---|
| navy_gold.json | 紺×金（高級・信頼） | D 無料相談 | 9 |
| brown_gold.json | 茶×金（温かい・上質） | D 無料相談 | 9 |
| black_vermilion.json | 黒×朱（強い・和） | D 無料相談 | 9 |
| yellow_blue.json | 黄×青（フレッシュ・テック） | D 無料相談 | 9 |
| orange_green.json | 橙×緑（ポップ・親しみ） | A LINE友だち登録 | 10 |

**写すときの必須差し替え**
- **ID**: catalog の ID はサンプルの章順。構成案表で承認した ID（`sec-{2桁順}-{役割}`）に置き換える。章を抜いたら番号を振り直す
- **href**: 全ボタンの href は差し替えトークン付きのサンプル形式。`https://mosh.jp/services/REPLACE_WITH_SERVICE_ID?openExternalBrowser=1`（MOSH のプラン・サービス）/ `https://lin.ee/REPLACE_WITH_LINE_ID`（LINE 友だち追加）。`REPLACE_WITH_` の部分をユーザーから確認した実 ID に置換する。未確定なら `要記入` として保存前に必ず解消（`REPLACE_WITH_` が残った状態で保存・公開しない）
- **画像**: `src` はダミー写真（灰色プレースホルダ素材、またはpreset内の一部セクションではフリー素材由来のイラストレーション用写真）。いずれも公開用の実画像ではないため、公開前に必ずユーザーの実画像へ差し替える（`best-practices.md`「画像の運用」参照）
- **文言・数字**: すべて仮置き。ヒアリングシートの値に置き換え、実績は確定値のみ

文言はサンプル（副業ライフコーチ講座 / LINE無料テンプレ配布）。数字はすべて仮置き。
