# LP設計ベストプラクティス

## 推奨セクション構成

典型的なLPは以下のセクション順で構成する：

1. **Hero** — キャッチコピーとメインビジュアル
2. **Problem** — ユーザーの課題・悩み
3. **Solution** — 提供する解決策
4. **Features** — サービスの特徴・メリット
5. **CTA** — アクションボタン
6. **FAQ** — よくある質問
7. **Final CTA** — 最終アクションボタン

すべての要素は `section` の `children` として配置する。直接 `elements` 配列にセクション以外の要素を置かない。

## モバイル対応

- `mobileStyles` でフォントサイズを小さくする（PC `2rem` → モバイル `1.4rem` 程度）
- パディングをモバイル向けに調整する（PC `64px 48px` → モバイル `32px 16px` 程度）
- `attributes.showDesktop` / `attributes.showMobile` でデバイス別の表示制御が可能
