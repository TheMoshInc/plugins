# LP設計ベストプラクティス

## 推奨セクション構成

典型的なLPは以下のセクション順で構成する。「役割」は設計上の呼び名で、実際に `attributes.sectionType` に入れる値は「`sectionType`」列の値を使う（`"hero"` や `"cta"` は無効値。許容値の一覧は `content-schema.md` の「`sectionType` の許容値」を参照）。

| 順 | 役割 | 内容 | `sectionType` |
|---|---|---|---|
| 1 | メイン（Hero） | キャッチコピーとメインビジュアル | `main` |
| 2 | 課題提起（Problem） | ユーザーの課題・悩み | `description` |
| 3 | 解決策（Solution） | 提供する解決策 | `description` |
| 4 | 特徴・メリット（Features） | サービスの特徴・メリット | `merit` |
| 5 | CTA | アクションボタン | `main` または `description` |
| 6 | よくある質問（FAQ） | よくある質問 | `faq` |
| 7 | 最終CTA | 最終アクションボタン | `main` または `description` |

すべての要素は `section` の `children` として配置する。直接 `elements` 配列にセクション以外の要素を置かない。

## モバイル対応

- `mobileStyles` でフォントサイズを小さくする（PC `30px` → モバイル `20px` 程度）。`rem` は使わず、`content-schema.md` の固定スケール（px）から選ぶ
- パディングをモバイル向けに調整する（PC `64px 48px` → モバイル `32px 16px` 程度）
- `attributes.showDesktop` / `attributes.showMobile` でデバイス別の表示制御が可能
