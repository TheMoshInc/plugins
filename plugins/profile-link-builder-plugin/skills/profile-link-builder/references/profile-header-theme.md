# プロフィール・ヘッダー・テーマ（ツール仕様）

3つとも「取得 → 更新」の1対。更新後のレスポンスは更新後の値（確認に使える）。

## プロフィール情報

### `getProfileLinkProfile`
引数なし。返り値: `iconPictureUrl` / `catchPhrase` / `name` / `selfIntroduction`（未設定は空文字）、`accountName`（共有URL用・読み取り専用）、`reportSummary`（星付き感想の平均と件数。読み取り専用・0件は null）。

### `patchProfileLinkProfile`
ボディ（4項目すべて必須。**変えない項目は `null` を渡すと現状維持**）:

| フィールド | 型 | 説明 |
|---|---|---|
| `name` | string \| null | クリエイター名 |
| `catchPhrase` | string \| null | キャッチコピー |
| `selfIntroduction` | string \| null | 自己紹介（改行可） |
| `iconPictureId` | string \| null | アイコン画像の ID |

- **ここだけは全置換ではない**。`null` を渡した項目は変わらない。**空文字は「変えない」ではなく空で上書きする**（名前やキャッチコピーを消したいときだけ。空文字の挙動は実機未確認で、null は維持するというツール定義の記載からの判断）。変えない項目に空文字を使わない
- 取得結果の `iconPictureUrl` は URL。更新に渡すのは `iconPictureId`（画像 ID）。画像を変える場合は [header-sns-gallery-media.md](header-sns-gallery-media.md) の取り込みで ID を得る。**変えないなら `null`**（URL から ID は作れない）
- 文字数上限などの制約は API 定義に明記なし。エラーが返ったら内容を直して再実行

## ヘッダー

### `getProfileLinkHeader`
引数なし。返り値: `templateId`（`header-1`〜`header-6`。未設定はサーバーが `header-1` で返す）/ `imageId` / `imageUrl` / `imageBlur`（0〜20）/ `textColor`（未設定は空文字）。

### `patchProfileLinkHeader`
**全置換**（4項目すべて必須）:

| フィールド | 型 | 説明 |
|---|---|---|
| `templateId` | `header-1`〜`header-6` | ヘッダーデザイン。更新では必ず明示する |
| `imageId` | string | 背景画像の ID。空文字で未設定 |
| `imageBlur` | integer 0〜20 | 背景のぼかし |
| `textColor` | string | 文字色。空文字で未設定 |

- テンプレートだけ変える場合も、他の3項目は**取得した値をそのまま**渡す。**`imageId` を空文字で送ると背景画像が消える**（取得結果の `imageUrl` しか見ずに `imageId` を空にしがち。必ず取得結果の `imageId` を使う）
- ヘッダーデザインによって、背景画像・ぼかし・文字色が効くものと効かないものがある。効かない組み合わせの値を保存しても画面に出ない。変えない項目は取得した値をそのまま渡す（どのデザインで何が効くかは、ユーザーに断定して案内しない）
- ヘッダーのSNSアイコンはここでは扱えない → [header-sns-gallery-media.md](header-sns-gallery-media.md)

## テーマ

### `getProfileLinkTheme`
引数なし。返り値は下の6項目（`patchProfileLinkTheme` と同じ）。

### `patchProfileLinkTheme`
**全置換**（6項目すべて必須）。背景は「どれか1つのモード」を使い、使わないモードの項目は決まった空値にする:

| フィールド | 型 | 使うモード / 空値 |
|---|---|---|
| `backgroundColor` | string | 単色モードの背景色（hex）。他モードでは `""` |
| `gradient` | string | グラデーションモードの CSS 定義。他モードでは `""` |
| `imageId` | string \| null | 画像モード（アップロード画像）の ID。プリセット画像・他モードでは `null` |
| `imageUrl` | string | 画像モードの URL（プリセットの静的パス or アップロード画像の解決済み URL）。他モードでは `""` |
| `imageBlur` | integer 0〜20 | 画像モードのぼかし。他モードでは `0` |
| `textColor` | string | 文字色（hex）。**全モード共通** |

- 現在どのモードかは、取得結果でどの項目に値が入っているかで判断する（単色なら `backgroundColor` が非空）
- モードを切り替えるときは、**新しいモードの項目を埋め、前のモードの項目を空値に戻す**（残すと不整合になり得る）。`textColor` は引き継ぐ
- プリセット画像の `imageUrl`（静的パス）は取得結果で見えるが、**選べるプリセットの一覧を取るツールは無い**。ユーザーが画面で選んだ画像を指定できない場合は、管理画面での選択を案内する
- 色はユーザーの言葉（「青っぽく」）から hex を創作しない。具体的な色を確認するか、候補を示して選んでもらう
