# トレース例: 「背景色を変えて」（全置換の基本形）

取得 → 変えない項目を引き継ぐ → 提示 → 承認 → 実行 → 読み戻し、までの完全な流れ。

## ユーザー発話

> プロフィールリンクの背景を、薄いグレーの #F2F2F2 にして。

## 1. 現在値の取得

```
getProfileLinkTheme()
→ { backgroundColor: "", gradient: "linear-gradient(135deg, #ffd1dc, #c1e1ff)", imageId: null, imageUrl: "", imageBlur: 0, textColor: "#333333" }
```

今はグラデーションモード。単色に切り替えるので、`gradient` を空にし、`backgroundColor` を埋める。`textColor` は全モード共通なので引き継ぐ。

## 2. 変更後の提示

> 今の背景は、ピンクから水色のグラデーションです。以下の内容で更新します。
>
> - 背景: グラデーション → **単色 #F2F2F2**（グラデーションの設定は外れます）
> - 文字色: #333333（そのまま）
>
> この内容で進めてよいですか？

## 3. 承認 → 4. 実行

→ ユーザー: 「お願いします」

```
patchProfileLinkTheme(
  bodyParams: {
    backgroundColor: "#F2F2F2",
    gradient: "",
    imageId: null,
    imageUrl: "",
    imageBlur: 0,
    textColor: "#333333"
  }
)
```

## 5. 読み戻しと報告

返り値（更新後のテーマ）で `backgroundColor` が `#F2F2F2`、`gradient` が空であることを確認してから伝える。

> 背景を単色の #F2F2F2 に変更しました。文字色はそのままです。プロフィールページには通常すぐ反映されます。

## NG（やってはいけない）

- `textColor` を省略する、または適当な色で埋める → 文字色が変わってしまう
- `gradient` を残したまま `backgroundColor` だけ埋める → 2つのモードが同居する
- 取得せずに「たぶんこうだろう」で6項目を埋める
