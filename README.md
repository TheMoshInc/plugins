# plugins

MOSH が配布する [Claude Code](https://code.claude.com/) プラグインマーケットプレイス。

## 収録プラグイン

| プラグイン | 説明 |
| :--- | :--- |
| `lp-builder-plugin` | ランディングページを構築するための lp-builder skill |

## インストール

Claude Code 内で以下を実行:

```shell
/plugin marketplace add TheMoshInc/plugins
/plugin install lp-builder-plugin@mosh-plugins
```

## 更新

```shell
/plugin marketplace update mosh-plugins        # カタログを再取得
/plugin update lp-builder-plugin@mosh-plugins  # プラグイン本体を更新
```

引数を省略するとすべてのマーケットプレイス/プラグインが対象になります。

## アンインストール

```shell
/plugin uninstall lp-builder-plugin@mosh-plugins
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
