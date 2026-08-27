# トラッキング同意（`isTrackingEnabled`）

**本文に URL を含めてクリック計測を有効にするときだけ読む。** 本文に URL が無い場合、または下書き保存だけで
終える場合は不要。

## 同意が必要な条件

`isTrackingEnabled` が true の場合（**既存の値をそのまま送る場合も含む**）、本文（メールは `body`、
LINE は `contents[].text`）に URL を含むなら**事前にクリエイター本人の同意が必要**。
下書き保存のみの場合は不要。

## 手順

1. `getCreatorScenariosTrackingConsents` で `isConsented` を確認する。
2. すでに true なら、そのまま `isTrackingEnabled: true` を指定してよい。
3. false の場合は以下の全項目をユーザーに提示し、**明示的な同意を得てから** `postCreatorScenariosTrackingConsent`
   を呼び、その後に `isTrackingEnabled: true` を指定する。同意が得られない場合は
   `isTrackingEnabled: false` のまま進めるか、URL を本文から外す。

提示する内容:

- 本文中の URL が MOSH 短縮 URL (mosh.jp/xxx) に自動変換される
- ゲストが短縮 URL から MOSH に会員登録・ログインすると、コンタクトの連絡先と MOSH ID が紐づく
- クリエイターに提供される情報: ゲストの MOSH ID / 会員登録時に利用したリンク情報
- 利用目的はサービスの利用状況の確認と、それに応じた案内・告知・連絡に限られる
- 提供された情報の目的外利用・第三者への再提供は禁止。個人情報の取扱いに関する関連法令を遵守すること
- 詳細はプライバシーポリシー https://mosh.jp/privacy-policy を案内する

**ユーザーの明示的な同意なしに `postCreatorScenariosTrackingConsent` を呼んではならない。**

## 管理画面との差異

管理画面では本文の編集ステップで、下書きかどうかによらず同意を求める。MCP で同意なしに保存した下書きを
管理画面で開くと同意を求められるため、ユーザーからその旨の質問があればこの差異を説明する。
