# plugins

MOSH が配布する [Claude Code](https://code.claude.com/) プラグインマーケットプレイス。

## 収録プラグイン

| プラグイン | 説明 |
| :--- | :--- |
| `lp-builder-plugin` | ランディングページを構築するための lp-builder skill |
| `workflow-builder-plugin` | ワークフロー（ステップ配信）を構築する workflow-builder skill と、既存ワークフローを点検する workflow-review skill |
| `contact-broadcast-plugin` | コンタクト向け一斉配信（メール / LINE）を作成・運用するための contact-broadcast skill |
| `sales-reporter-plugin` | 売上を確認・分析するための sales-reporter skill |
| `file-share-plugin` | ファイル共有（動画・画像・PDF）を操作するための file-share skill |
| `product-navigator-plugin` | 商品・商品プランを参照するための product-navigator skill |
| `line-rich-menu-plugin` | LINE リッチメニューを確認・編集・デフォルト設定・削除するための line-rich-menu skill |
| `product-builder-plugin` | 商品を作成・更新・公開・削除するための product-builder skill |
| `membership-site-builder-plugin` | 会員サイトのフォルダ・コンテンツ・タグを作成・編集・公開するための membership-site-builder skill |
| `contact-list-manager-plugin` | コンタクト（LINE友だち / メール購読者）を検索・確認・削除・CSVインポートするための contact-list-manager skill |

## インストール

Claude Code 内で以下を実行:

```shell
/plugin marketplace add TheMoshInc/plugins
/plugin install lp-builder-plugin@mosh-plugins
/plugin install workflow-builder-plugin@mosh-plugins
/plugin install contact-broadcast-plugin@mosh-plugins
/plugin install sales-reporter-plugin@mosh-plugins
/plugin install file-share-plugin@mosh-plugins
/plugin install product-navigator-plugin@mosh-plugins
/plugin install line-rich-menu-plugin@mosh-plugins
/plugin install product-builder-plugin@mosh-plugins
/plugin install membership-site-builder-plugin@mosh-plugins
/plugin install contact-list-manager-plugin@mosh-plugins
```

## 更新

```shell
/plugin marketplace update mosh-plugins               # カタログを再取得
/plugin update lp-builder-plugin@mosh-plugins               # プラグイン本体を更新
/plugin update workflow-builder-plugin@mosh-plugins         # プラグイン本体を更新
/plugin update contact-broadcast-plugin@mosh-plugins        # プラグイン本体を更新
/plugin update sales-reporter-plugin@mosh-plugins           # プラグイン本体を更新
/plugin update file-share-plugin@mosh-plugins               # プラグイン本体を更新
/plugin update product-navigator-plugin@mosh-plugins        # プラグイン本体を更新
/plugin update line-rich-menu-plugin@mosh-plugins           # プラグイン本体を更新
/plugin update product-builder-plugin@mosh-plugins          # プラグイン本体を更新
/plugin update membership-site-builder-plugin@mosh-plugins  # プラグイン本体を更新
/plugin update contact-list-manager-plugin@mosh-plugins     # プラグイン本体を更新
```

引数を省略するとすべてのマーケットプレイス/プラグインが対象になります。

## アンインストール

```shell
/plugin uninstall lp-builder-plugin@mosh-plugins
/plugin uninstall workflow-builder-plugin@mosh-plugins
/plugin uninstall contact-broadcast-plugin@mosh-plugins
/plugin uninstall sales-reporter-plugin@mosh-plugins
/plugin uninstall file-share-plugin@mosh-plugins
/plugin uninstall product-navigator-plugin@mosh-plugins
/plugin uninstall line-rich-menu-plugin@mosh-plugins
/plugin uninstall product-builder-plugin@mosh-plugins
/plugin uninstall membership-site-builder-plugin@mosh-plugins
/plugin uninstall contact-list-manager-plugin@mosh-plugins
/plugin marketplace remove mosh-plugins
```

## 利用規約

本リポジトリは、オープンソースソフトウェアとして公開するものではありません。
本リポジトリは、[MOSH](https://mosh.jp)を利用するために必要なファイル等を公開する目的で設置されています。

本リポジトリ内のファイルの利用には、以下の利用規約および利用ルールが適用されます。

- MOSH利用規約: <https://mosh.jp/terms>
- MOSH MCP利用ルール: <https://moshjp.notion.site/MOSH-MCP-366647fa6ef6806dabfbd279a8956a17>

本リポジトリ内のファイルをダウンロード、clone、インストール、実行、複製、改変、またはその他の方法で利用した場合、上記の利用規約および利用ルールに同意したものとみなします。

本リポジトリの公開およびGitHub上の機能利用に関して、上記の利用規約または利用ルールとGitHub利用規約が矛盾する場合、GitHub上の公開リポジトリについてGitHub利用規約に基づき利用者に認められる範囲に限り、GitHub利用規約が優先して適用されます。
