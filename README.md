# DS検定学習

DS検定（データサイエンティスト検定 リテラシーレベル）向けの、端末内保存型学習アプリです。問題データはリポジトリに含めず、初回に各端末でJSONファイルを読み込みます。

## 使い方

1. GitHub Pagesの公開URLを開きます。
2. 「問題データを読み込む」から `DS_questions.json` を選択します。
3. 解答履歴はブラウザ内に保存されます。端末をまたぐときは「データ管理」から `progress.json` をエクスポートし、別端末でインポートします。
4. iPhoneではSafariの共有メニューから「ホーム画面に追加」して利用してください。

## ローカル確認

静的サーバーで開いてください（`file://` を直接開かない）。

```powershell
python -m http.server 8000
```

`http://localhost:8000` を開きます。

## GitHub Pages

公開リポジトリの Settings → Pages で **Deploy from a branch**、`main`、`/(root)` を選択します。問題データの `questions.json` と学習記録の `progress.json` は `.gitignore` により公開されません。
