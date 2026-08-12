# DS検定（データサイエンティスト検定 リテラシーレベル）学習アプリ 開発ブリーフ（v1 / 確定版）

> 本書はCodexに実装を依頼するための仕様書である。判断の余地を残さないよう、選択肢がある箇所はすべて**採用案を1つに確定**して記述している。「〜でもよい」という記述は原則置かない。
>
> 本アプリは、同一人物が開発した [pmp-study-app](https://github.com/sakamoto4403/pmp-study-app)（PMP試験対策アプリ）を土台に、DS検定向けへ作り替えたものである。**画面構成・操作フロー・技術構成はpmp-study-appを踏襲する**。相違点は「章」を「分野」に置き換える点、章番号が数値ではなく文字列ID（`categoryId`）になる点、および分野数が4（各50問・計200問）である点である。相違点は本書中で都度明記する。

---

## 0. 前提・目的

| 項目 | 内容 |
|---|---|
| 利用者 | 本人1名のみ（個人利用） |
| 利用端末 | **iPhone（Safari）とPC（Chrome等）の両方**。両環境での動作を必須要件とする |
| 配信方法 | **GitHub Pages（publicリポジトリ）にデプロイし、URLでアクセス**（手順は§7） |
| サーバー処理 | なし（静的ファイルのみ。API・DB・認証は一切なし） |
| 問題データ | DS検定 全200問／4分野構成（`DS_questions.json`として提供済み）。データ自体は完成済みなので、アプリは所定のJSONスキーマを読み込む枠組みとして実装する |
| 学習記録 | ブラウザの`localStorage`に自動保存＋JSONファイルへの手動エクスポート／インポート |

### 0.1 問題データをリポジトリに含めない方針

DS検定の問題文はオリジナル生成データであり著作物としての制約はないが、**pmp-study-appと同じ運用（端末ごとの手動読み込み・リポジトリ非同梱）を踏襲する**方針とする。理由は、問題データの差し替え・改訂を行った際にリポジトリへコミットし直す手間を避けられること、および個人の学習記録同様に問題データも端末内で完結させる運用に統一できることによる。

- **GitHub Pagesに置くのはアプリのコード（HTML/CSS/JS）のみ。`questions.json`はリポジトリに含めない。**
- `questions.json`は利用者が各端末で**初回に1回だけ手動で読み込む**（`<input type="file">`でファイル選択）。読み込むファイルは添付済みの`DS_questions.json`をそのまま使用する（ファイル名は問わない。中身のスキーマのみを検証する）。
- 読み込んだ問題データは**`localStorage`にキャッシュ**し、2回目以降の起動では自動的にキャッシュから復元する（毎回選択させない）。
- iPhoneでは`questions.json`をiCloud Driveの「ファイル」アプリに置いておき、初回のみそこから選択する。

> データ量の見積り：200問×約400字＝約8万字。localStorageの容量上限（一般に約5MB＝UTF-16で約500万字）に対して十分収まる。IndexedDBは使用しない。

### 0.2 Safariのストレージ揮発リスクと対策

iOS Safariは、サイトを一定期間（目安7日間）利用しないと`localStorage`を破棄することがある（ITPによる挙動）。これに対し以下を実装する。

- 起動時に `navigator.storage.persist()` を呼び、可能な範囲でストレージの永続化を要求する（対応していない環境ではエラーを握りつぶして続行）。
- `localStorage`が空だった場合（＝消えた場合）は、トップ画面で「問題データを読み込んでください」「学習記録を復元してください」と明示的に案内する。**サイレントに初期状態へ戻さない。**
- 学習記録の定期的なエクスポートを促すため、トップ画面に「最終エクスポート：YYYY/MM/DD」を常時表示する（§3.1）。
- iPhoneでは**「ホーム画面に追加」して使うことを推奨**（ストレージが破棄されにくくなる）。この案内をデータ管理画面に一文入れる。

### 0.3 端末間の同期方法

iPhoneとPCで`localStorage`は共有されない。**学習記録の同期手段はエクスポート／インポートのみ**である。

- 例：PCで学習 → `progress.json`をエクスポート → iCloud Drive等に置く → iPhoneでインポート。
- この運用が前提であることを、データ管理画面に説明文として明記する。

---

## 1. 全体構成

### 1.1 分野構成（固定・4分野／各50問／全200問）

| 表示順 | `categoryId` | タイトル | 問題ID範囲 |
|---|---|---|---|
| 1 | `kiso` | 基礎（データリテラシー・数理基礎） | 1–50 |
| 2 | `data-science` | データサイエンス力 | 1001–1050 |
| 3 | `data-engineering` | データエンジニアリング力 | 2001–2050 |
| 4 | `value-creation` | 価値創造（ビジネス力） | 3001–3050 |

- **分野タイトル・分野数・表示順序はアプリ側にハードコードせず、読み込んだ`questions.json`の`categories`配列の並び順をそのまま使用する。** 上表は参考情報であり、分野数や問題数が異なるJSONを読み込んでも動作すること（例：1分野分だけのテスト用JSONでも正しく動く）。
- 問題IDは全体で一意な通し番号（分野をまたいで重複しない。上記4分野・200問で重複がないことは確認済み）。
- **pmp-study-appとの相違点**：pmp-study-appは`chapterNumber`（数値）を章の識別キーとして使用していたが、本アプリは`categoryId`（文字列）を分野の識別キーとして使用する。画面上の「第◯章」のような番号表示は、`categoryId`ではなく**`categories`配列内でのインデックス（0始まりを+1したもの）**を使って`分野1`のように動的に生成する。`categoryId`自体は画面に表示しない（`localStorage`のキーとしてのみ使用する）。

### 1.2 画面構成

単一ページアプリ（SPA）。以下5ビューをJSで切り替える（ルーティングライブラリ不要、`hidden`属性のON/OFFで実装。pmp-study-appと同一方式）。

1. トップ画面（分野一覧）
2. 演習画面（出題）
3. 演習終了サマリー画面
4. 復習モード（出題画面を再利用）
5. データ管理画面

---

## 2. データ仕様

### 2.1 問題データ `questions.json`

添付の`DS_questions.json`が実データであり、以下のスキーマに準拠している（変換不要でそのまま読み込める）。

```json
{
  "meta": {
    "exam": "DS検定（データサイエンティスト検定 リテラシーレベル）",
    "syllabusVersion": "スキルチェックリストver.6（2025年更新・2026年6月模擬問題準拠）",
    "difficulty": "本番相当",
    "generatedBy": "…",
    "totalQuestions": 200
  },
  "categories": [
    {
      "categoryId": "kiso",
      "title": "基礎（データリテラシー・数理基礎）",
      "questions": [
        {
          "id": 1,
          "text": "次のうち、社会全体でデータやAIを活用した新しい価値創造を目指す概念として、日本政府が提唱している考え方はどれか。",
          "choices": [
            { "key": "ア", "text": "Society 5.0" },
            { "key": "イ", "text": "Web 2.0" },
            { "key": "ウ", "text": "Industry 1.0" },
            { "key": "エ", "text": "Cloud First" }
          ],
          "answer": "ア",
          "explanation": "Society 5.0は、サイバー空間とフィジカル空間を高度に融合させ…"
        }
      ]
    }
  ]
}
```

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `meta` | object | ×（あれば使う） | 試験名・シラバス版数などのメタ情報。存在すればトップ画面に小さく表示する（§3.1）。存在しなくてもアプリは動作すること |
| `meta.exam` | string | × | 試験名 |
| `meta.syllabusVersion` | string | × | シラバス／出題範囲のバージョン表記 |
| `categories[].categoryId` | string | ○ | 分野の一意な識別子（`localStorage`保存キーとして使用。画面には表示しない） |
| `categories[].title` | string | ○ | 分野タイトル（画面にそのまま表示） |
| `questions[].id` | number | ○ | 全体で一意な通し番号 |
| `questions[].text` | string | ○ | 問題文 |
| `choices[].key` | string | ○ | `"ア"`／`"イ"`／`"ウ"`／`"エ"` |
| `choices[].text` | string | ○ | 選択肢本文 |
| `answer` | string | ○ | 正解のkey（1つ） |
| `explanation` | string | ○ | 解説文 |

- 選択肢は4つ固定。
- **読み込み時のバリデーションは最小限**：JSONパース可否と`categories`配列の存在のみ確認し、失敗時は画面上にエラーメッセージを表示（`alert`は使わずDOM上に表示）。各問題の中身の妥当性検査は不要。`meta`の有無はチェックしない（無くてもエラーにしない）。

### 2.2 学習記録 `progress.json`

**localStorageに保存する構造とエクスポートするJSONの構造は同一**とする（そのまま書き出す）。pmp-study-appの`chapterStatus`／`resumePointer`に相当するキーを、`categoryId`（文字列）で管理する点のみが異なる。

```json
{
  "version": 1,
  "exportedAt": "2026-08-12T12:00:00+09:00",
  "lastExportedAt": "2026-08-12T12:00:00+09:00",
  "records": {
    "1": {
      "attempts": 3,
      "lastResult": "correct",
      "history": [
        { "timestamp": "2026-08-10T10:00:00+09:00", "selected": "エ", "correct": false },
        { "timestamp": "2026-08-11T09:00:00+09:00", "selected": "ア", "correct": true },
        { "timestamp": "2026-08-12T11:00:00+09:00", "selected": "ア", "correct": true }
      ]
    }
  },
  "categoryStatus": {
    "kiso": "completed",
    "data-science": "in_progress"
  },
  "resumePointer": {
    "data-science": 1023
  }
}
```

| フィールド | 説明 |
|---|---|
| `version` | 固定値`1` |
| `exportedAt` | エクスポート実行時刻（ISO 8601）。エクスポート時に毎回更新 |
| `lastExportedAt` | 最後にエクスポートした時刻。トップ画面に表示（§3.1） |
| `records` | key＝問題IDの文字列。回答履歴を**累積**保持（上書きしない） |
| `records[id].attempts` | 回答回数。**必ず`history.length`と一致させる** |
| `records[id].lastResult` | 直近の正誤（`"correct"`／`"incorrect"`）。`history`末尾と一致 |
| `records[id].history` | 全回答履歴。新しいものを末尾に追加 |
| `categoryStatus[categoryId]` | `"not_started"`／`"in_progress"`／`"completed"` の3値。キーが存在しない分野は`"not_started"`とみなす |
| `resumePointer[categoryId]` | **次に出題すべき問題ID**。分野を最後まで解き終えたらそのキーを削除する |

### 2.3 localStorageのキー名（確定）

Codexは以下のキー名を使用すること（pmp-study-appとは別アプリ・別リポジトリ・別GitHub Pages URLになるため衝突しないが、識別しやすいよう専用のキー名にする）。

| キー | 内容 |
|---|---|
| `ds_kentei_app_questions_v1` | `questions.json`の全内容（JSON文字列） |
| `ds_kentei_app_progress_v1` | §2.2の学習記録（JSON文字列） |

- 起動時：`ds_kentei_app_questions_v1`があれば読み込み、無ければトップ画面に「問題データを読み込む」導線を表示。
- `ds_kentei_app_progress_v1`が無い場合は`{ version: 1, records: {}, categoryStatus: {}, resumePointer: {} }`で初期化する。

---

## 3. 画面詳細仕様

### 3.1 トップ画面（分野一覧）

**ヘッダー部（常時表示）**
- アプリタイトル「DS検定学習」
- `questions.json`に`meta.exam`／`meta.syllabusVersion`が含まれる場合、タイトル直下に小さく1行で表示する（例：`DS検定（データサイエンティスト検定 リテラシーレベル）｜スキルチェックリストver.6（2025年更新・2026年6月模擬問題準拠）`）。含まれない場合は何も表示しない
- 全体サマリー：`解答済 120/200問 ／ 全体正答率 78%`
- `最終エクスポート：2026/08/12 12:00`（未エクスポートなら「未エクスポート」と赤系で表示）
- 「解答履歴」ボタン（→履歴画面へ遷移。pmp-study-appの履歴機能をそのまま踏襲）
- 「データ管理」ボタン（→§3.5へ遷移）
- 「苦手問題を復習する（全分野横断）」ボタン。対象0問のときはボタンをdisableし「苦手問題はありません」と表示

**問題データ未読込のとき**
- 分野一覧の代わりに「問題データ（questions.json）を読み込んでください」と案内し、ファイル選択ボタンを大きく表示する。

**分野一覧（問題データ読込済のとき）**

`categories`配列の順序どおりに、各分野について1行（カード）で以下を表示：
- 表示順序番号＋分野タイトル（例：`分野1　基礎（データリテラシー・数理基礎）`。番号は配列インデックス+1から動的に生成し、`categoryId`は表示しない）
- ステータスバッジ：`未着手`／`進行中`／`完了`
- `解答済 30/50問 ・ 正答率 60%`（§3.6の計算式）
- アクションボタン（`categoryStatus`により文言を出し分け）
  - `not_started` → **「演習を始める」**（1問目から開始）
  - `in_progress` → **「続きから再開」**（`resumePointer`の問題から開始）＋ 副ボタン**「最初から解き直す」**
  - `completed` → **「もう一度解く」**（1問目から開始）＋ 副ボタン**「間違えた問題だけ復習」**（その分野の`lastResult === "incorrect"`が0問ならdisable）

### 3.2 演習画面（出題）

- 分野内の出題順は**id昇順（データの並び順どおり）**。シャッフルは行わない。
- 表示要素：
  - 上部：`分野2 ／ 問 13 / 50`（分野内の通し位置。IDではなく分野内順序で表示。「分野2」は表示順序番号であり`categoryId`ではない）
  - 「中断してトップへ戻る」リンク
  - 問題文
  - 選択肢4つ（**カード型のボタンとして実装**。ラジオボタンは使わない。理由：タップ領域を大きく取るため）
  - 「解答する」ボタン（選択肢未選択のときはdisable）

**解答後の挙動**
1. 全選択肢をdisableし、選択不可にする
2. 正誤を明示：正解の選択肢を緑系、誤答して選んだ選択肢を赤系で着色。冒頭に`正解`／`不正解`を大きく表示
3. 解説文を表示
4. `records[id]`に履歴を追記（`attempts`＋1、`lastResult`更新）→ `localStorage`へ即時保存
5. `categoryStatus[categoryId]`を`"in_progress"`に更新（既に`"completed"`の分野を解き直している場合も`"in_progress"`に戻す）
6. `resumePointer[categoryId]`を**次の問題のID**に更新（分野の最終問題を解いた場合はキーを削除）
7. 「次の問題へ」ボタンを表示。**分野の最終問題では「結果を見る」ボタンに変える**

**セッション内集計**
- 演習開始時にセッション用の配列（例：`sessionResults = []`）を初期化し、1問解くごとに`{ id, selected, correct }`を追加する。これを§3.3のサマリーで使用する。

### 3.3 演習終了サマリー画面

- 表示内容：
  - `分野2 演習結果`
  - **今回のセッションの成績**：`50問中 42問正解（84%）`（`sessionResults`ベース。§3.6の累積正答率とは別物である点に注意）
  - 間違えた問題の一覧（`sessionResults`の`correct === false`）。各行に「問題文の冒頭40字＋…」「あなたの解答：ア／正解：エ」を表示し、**行をタップすると展開して問題文全文と解説を表示**（アコーディオン）
  - 全問正解の場合は「全問正解です」と表示し、一覧は出さない
- ボタン：
  - 「間違えた問題だけ復習する」（間違いが0問ならdisable）
  - 「次の分野へ進む」（**`categories`配列の最後の分野の場合は非表示**。次の分野は配列上の次のインデックスとする）
  - 「トップに戻る」
- **この画面到達時に`categoryStatus[categoryId] = "completed"`に更新し、`resumePointer[categoryId]`を削除する。**

### 3.4 復習モード

**起動経路と出題対象**

| 経路 | 出題対象 |
|---|---|
| サマリー画面「間違えた問題だけ復習する」 | 直前セッションで誤答した問題 |
| トップ画面「苦手問題を復習する（全分野横断）」 | 全問題のうち`records[id].lastResult === "incorrect"` |
| 分野カードの「間違えた問題だけ復習」 | その分野のうち`lastResult === "incorrect"` |
| 解答履歴画面で問題行をタップ | その1問のみを復習出題（pmp-study-appと同一挙動） |

- 出題順はid昇順。
- 画面・操作は§3.2と**同一の描画関数を再利用**する。ヘッダー表示のみ`復習モード ／ 問 3 / 12`とする。
- 復習中の解答も通常どおり`records`に追記され、`lastResult`が更新される（＝正解すれば苦手リストから外れる）。
- **復習モードでは`categoryStatus`と`resumePointer`を更新しない**（分野の進捗を壊さないため）。
- 全問終えたら§3.3と同形式のサマリー画面を表示する。ただしボタンは「まだ間違えている問題をもう一度復習する」（対象0ならdisable）と「トップに戻る」の2つのみ。

### 3.5 解答履歴画面

pmp-study-appの解答履歴機能をそのまま踏襲する。

- フィルタ：分野（`categories`の順で選択肢を生成、値は`categoryId`）／ステータス（すべて・未解答・直近正解・直近不正解）／並び替え（問題番号順・解答回数が多い順・解答回数が少ない順）
- 表：行＝問題（`分野○ 問${id} ${問題文冒頭15字}`のラベル付きボタン。クリックでその1問を復習出題）、列＝1回目・2回目…と解答回数分の列を動的に生成し、各セルに正誤（○／✕）のみを表示する（日時は表示しない）
- 該当件数（`該当 ◯問 / 全200問`）を表の上に表示する

### 3.6 データ管理画面

以下を縦に並べる。各ボタンの直下に説明文を1〜2行添える。

1. **問題データ（questions.json）を読み込む**
   - `<input type="file" accept="application/json,.json">`
   - 読込成功時：`localStorage`の`ds_kentei_app_questions_v1`に保存し、「◯分野／◯問を読み込みました」と表示
   - 説明文：「問題データは端末内にのみ保存されます。買い替えやデータ消去の際は再読み込みが必要です。」

2. **学習記録をエクスポート（保存）**
   - 現在の`ds_kentei_app_progress_v1`に`exportedAt`と`lastExportedAt`を現在時刻でセットしてから、`progress.json`としてダウンロード
   - 実装：`Blob` → `URL.createObjectURL` → 動的生成した`<a download="progress.json">`をクリック → `URL.revokeObjectURL`
   - **iOS Safari対策**：`navigator.share`が利用可能な環境では、ダウンロードではなく**Web Share APIでファイル共有を試みる**（`navigator.canShare({ files: [...] })`で判定）。共有が使えない場合は上記の`<a download>`にフォールバックする
   - 説明文：「iPhoneでは『ファイル』アプリやiCloud Driveへ保存してください。別の端末でインポートすると学習記録を引き継げます。」

3. **学習記録をインポート（読み込み）**
   - `<input type="file" accept="application/json,.json">`
   - 選択後、`confirm("現在の学習記録を上書きします。よろしいですか？")`で確認 → OKなら`localStorage`を上書きし全画面を再描画
   - 説明文：「現在の記録は失われます。必要なら先にエクスポートしてください。」

4. **学習記録をすべてリセット**
   - `confirm()`で確認（文言：「すべての学習記録を削除します。この操作は取り消せません。よろしいですか？」）
   - 削除対象は`ds_kentei_app_progress_v1`のみ。**問題データ（`ds_kentei_app_questions_v1`）は消さない**

5. **利用上の注意（テキストのみ）**
   - 「iPhoneでは、ホーム画面に追加してご利用ください。学習記録が消えにくくなります。」
   - 「学習記録はブラウザ内に保存されます。週に一度はエクスポートして保管することをおすすめします。」

### 3.7 正答率の計算ルール（確定）

**用語**
- 「解答済み」＝`records[id]`が存在する問題
- 「現在の正誤」＝`records[id].lastResult`

**分野の表示**
- 解答済み数 ＝ その分野の問題のうち`records`に存在する数
- 正答率 ＝ `lastResult === "correct"` の数 ÷ **その分野の全問題数**
- 表示例：`解答済 30/50問 ・ 正答率 60%`（30問全問正解でも分母は50なので60%）

**全体の表示**
- 同じロジックを全問題（200問）に対して適用

**セッション成績（§3.3のみ）**
- `sessionResults`の`correct === true`の数 ÷ `sessionResults.length`。累積の正答率とは別に算出する

- パーセントは小数点以下四捨五入の整数表示。分母0のときは`—`と表示する。

---

## 4. 技術構成

### 4.1 使用技術

- **プレーンHTML＋CSS＋Vanilla JavaScript**。フレームワーク・ビルドツール・パッケージマネージャは一切使用しない。
- 外部CDNからのライブラリ読み込みも行わない（完全に自己完結）。
- JSは`app.js`単一ファイル。関数単位で整理し、グローバル状態は1つのオブジェクト（例：`const state = { questions, progress, view, session, history }`）に集約する。
- **pmp-study-appのapp.jsを参考実装として流用してよい**が、以下の識別子はすべて置き換えること。

| pmp-study-app | ds-kentei-study-app |
|---|---|
| `chapters` | `categories` |
| `chapter` / `chapterNumber` | `category` / `categoryId`（文字列） |
| `chapterStatus` | `categoryStatus` |
| `"第◯章"` の表示 | `"分野◯"`（配列インデックス+1）の表示 |
| `pmp_app_questions_v1` | `ds_kentei_app_questions_v1` |
| `pmp_app_progress_v1` | `ds_kentei_app_progress_v1` |
| `PMP学習`（アプリ名） | `DS検定学習` |

### 4.2 ファイル構成（確定）

```
/ds-kentei-study-app
  ├─ index.html
  ├─ style.css
  ├─ app.js
  ├─ manifest.json          ← PWA用
  ├─ icon-192.png           ← PWA用アイコン（単色＋文字程度の簡易なもので可）
  ├─ icon-512.png
  ├─ .gitignore             ← questions.json / progress.json を必ず除外
  ├─ docs/
  │   └─ BRIEF.md           ← 本書
  └─ README.md              ← 使い方とデプロイ手順を記載
```

**`.gitignore`には必ず以下を含める**（問題データの誤コミット防止）：
```
questions.json
progress.json
*.local.json
```

### 4.3 PWA対応（ホーム画面追加）

`index.html`の`<head>`に以下を含める：

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="default">
<meta name="apple-mobile-web-app-title" content="DS検定学習">
<meta name="robots" content="noindex">
<link rel="apple-touch-icon" href="./icon-192.png">
<link rel="manifest" href="./manifest.json">
```

`manifest.json`は`name`（`"DS検定学習"`）／`short_name`（`"DS検定"`）／`start_url`（`"./"`）／`display: "standalone"`／`background_color`／`theme_color`／`icons`を持つ最小構成とする。

- **Service Workerは実装しない**（オフラインキャッシュは不要。実装するとデプロイ後に古いキャッシュが残る事故が起きやすいため、意図的に対象外とする）。

### 4.4 UI／レイアウト

- **モバイルファースト**で設計。基準幅はiPhone縦持ち（375〜430px）。PCでは`max-width: 720px`のコンテナを中央寄せにする。
- タップ対象（選択肢カード、ボタン）は**最小高さ48px**を確保。選択肢間の余白は8px以上。
- 本文の基本フォントサイズは16px以上（iOSで入力時に自動ズームされるのを防ぐため、`input`も16px以上）。
- セーフエリア対応：`padding: env(safe-area-inset-*)`を下部固定要素に適用する。
- 「解答する」「次の問題へ」ボタンは**画面下部に固定表示**し、長い問題文でもスクロールせずに押せるようにする。
- 配色はライトモード基準の単色アクセント。**ダークモード対応は実装しない**（スコープ外）。
- アニメーション・トランジションは最小限にする。

---

## 5. スコープ外（実装しないもの）

- ログイン・認証・複数ユーザー対応
- サーバーサイド処理・DB・外部API通信
- ランダム出題・選択肢シャッフル
- タイマー付き模擬試験（将来拡張候補）
- 正答率推移グラフなどの可視化（将来拡張候補）
- Service Workerによるオフライン動作
- ダークモード

---

## 6. 完成の定義（Definition of Done）

**機能面**
- [ ] `questions.json`（添付の`DS_questions.json`、または1分野分のサンプルでも可）をファイル選択から読み込み、分野一覧が表示される
- [ ] ブラウザをリロードしても問題データが再選択なしで復元される（localStorageキャッシュ）
- [ ] `meta`情報がある場合、トップ画面に試験名・シラバス版数が表示される
- [ ] 分野を選んで演習し、正誤判定・着色・解説表示・次問遷移が正しく動作する
- [ ] 分野の途中で「中断してトップへ戻る」→ 再度開くと**続きの問題から**再開される
- [ ] 分野を最後まで解くとサマリー画面が表示され、誤答一覧がアコーディオンで展開できる
- [ ] 「間違えた問題だけ復習する」で該当問題のみが出題される
- [ ] トップの「苦手問題を復習する（全分野横断）」で全分野から誤答問題が集約出題される
- [ ] 復習で正解した問題が苦手リストから外れる
- [ ] 解答履歴画面で分野・ステータスによる絞り込みと、解答回数列のマトリクス表示が正しく動作する
- [ ] 分野別・全体の正答率が§3.7の計算式どおりに表示される
- [ ] エクスポートで`progress.json`が保存され、インポートで正しく復元される
- [ ] リセット後も問題データは残っている

**マルチデバイス面**
- [ ] GitHub PagesのURLをiPhone Safari／PCブラウザの両方で開いて全機能が動作する
- [ ] iPhoneで縦持ちしたときにレイアウト崩れ・横スクロールが発生しない
- [ ] 選択肢が指で押しやすいサイズになっている
- [ ] **iPhoneでエクスポート → PCでインポートして学習記録が引き継げる**（実機で確認）
- [ ] iPhoneで「ホーム画面に追加」するとアドレスバーなしで起動する

**リポジトリ面**
- [ ] `questions.json`／`progress.json`がGitの管理対象に**入っていない**（`git status`と公開URLで確認）
- [ ] `README.md`に使い方とデプロイ手順が書かれている

---

## 7. GitHub Pagesへのデプロイ手順

### 7.1 前提

- GitHubアカウント（`sakamoto4403`）を保有していること
- PCにGitがインストールされていること（`git --version`で確認）
- **リポジトリはpublicで作成する**（無料プランでGitHub Pagesを使うため）。§0.1のとおり問題データは含めないので、公開されるのはアプリのコードのみ。

### 7.2 手順（初回）

**① リポジトリを作成**

GitHubにログイン → 右上の「＋」→「New repository」

| 項目 | 設定値 |
|---|---|
| Repository name | `ds-kentei-study-app` |
| Description | （任意） |
| Visibility | **Public** |
| Add a README file | チェックしない（ローカルから push するため） |

「Create repository」をクリック。

**② ローカルでコードを用意してpush**

作成したアプリのフォルダで、以下を実行する。

```bash
cd /path/to/ds-kentei-study-app

# .gitignore に questions.json / progress.json が入っていることを必ず先に確認
cat .gitignore

git init
git add .
git status          # ← questions.json が一覧に出ていないことをここで必ず確認する
git commit -m "Initial commit: DS kentei study app"
git branch -M main
git remote add origin https://github.com/sakamoto4403/ds-kentei-study-app.git
git push -u origin main
```

> `git status`で`questions.json`が出てきた場合は、`.gitignore`の記述を修正してから`git rm --cached questions.json`を実行し、再度確認すること。

**③ GitHub Pagesを有効化**

1. リポジトリのページ → 上部タブ「**Settings**」
2. 左サイドバー「**Pages**」
3. 「Build and deployment」の「Source」を「**Deploy from a branch**」にする
4. 「Branch」で「**main**」／フォルダは「**/ (root)**」を選び「**Save**」
5. 1〜2分待つと同じ画面上部に公開URLが表示される

公開URL：
```
https://sakamoto4403.github.io/ds-kentei-study-app/
```

**④ 各端末で初期設定**

- PC：上記URLを開く → データ管理画面 → `DS_questions.json`を読み込む
- iPhone：`DS_questions.json`をiCloud Driveに保存しておく → Safariで上記URLを開く → データ管理画面 → 「ファイル」アプリから`DS_questions.json`を選択 → 共有ボタン →「ホーム画面に追加」

### 7.3 更新手順（2回目以降）

コードを修正したら以下を実行するだけで、1〜2分後に公開サイトへ反映される。

```bash
git add .
git commit -m "変更内容の説明"
git push
```

反映されない場合はブラウザのスーパーリロード（PC：`Ctrl+Shift+R` / `Cmd+Shift+R`）を試す。

### 7.4 ローカルでの開発・確認

`file://`で直接開くと一部機能が正しく動かないため、簡易サーバー経由で確認する。

```bash
cd /path/to/ds-kentei-study-app
python3 -m http.server 8000
```

ブラウザで `http://localhost:8000` を開き、`DS_questions.json`を選択して確認する。

### 7.5 注意事項

- GitHub Pagesは**サイト全体がインターネットに公開される**。URLは推測されにくいが検索エンジンにインデックスされる可能性はあるため、`index.html`に`<meta name="robots" content="noindex">`を入れておく（§4.3で指定済み）。
- 学習記録（`progress.json`）は絶対にリポジトリへコミットしない。

---

## 8. Codexへの依頼手順（ステップバイステップ）

以下の順序で段階的に実装を依頼すると、レビューしやすく手戻りが少ない。各ステップの冒頭で本書（`docs/BRIEF.md`）を参照させること。

**Step 1：リポジトリ雛形の作成**
プロンプト例：
> `docs/BRIEF.md` を読み、§4.2のファイル構成に沿って `index.html` / `style.css` / `manifest.json` / `.gitignore` / `README.md` の雛形を作成してください。`app.js` は空でよいので、まずガワだけ用意してください。

**Step 2：データ読み込みとトップ画面**
プロンプト例：
> `docs/BRIEF.md` の §2.1（データ仕様）、§2.3（localStorageキー）、§3.1（トップ画面）を実装してください。`questions.json` のファイル選択読み込み、`localStorage`へのキャッシュ、分野一覧の表示までを対象とします。動作確認には添付の `DS_questions.json` を使います。

**Step 3：演習画面・サマリー画面**
プロンプト例：
> `docs/BRIEF.md` の §3.2（演習画面）、§3.3（サマリー画面）、§3.7（正答率の計算ルール）を実装してください。

**Step 4：復習モード・解答履歴画面**
プロンプト例：
> `docs/BRIEF.md` の §3.4（復習モード）、§3.5（解答履歴画面）を実装してください。

**Step 5：データ管理画面（エクスポート／インポート／リセット）**
プロンプト例：
> `docs/BRIEF.md` の §3.6（データ管理画面）を実装してください。iOS Safariでの`navigator.share`対応も含めます。

**Step 6：PWA対応・レイアウト調整**
プロンプト例：
> `docs/BRIEF.md` の §4.3（PWA対応）、§4.4（UI／レイアウト）に沿って、`manifest.json` の仕上げとモバイルファーストのスタイル調整を行ってください。`icon-192.png` / `icon-512.png` は単色＋文字程度の簡易なもので構いません。

**Step 7：通し確認**
プロンプト例：
> `docs/BRIEF.md` §6（完成の定義）のチェックリストを1つずつ確認し、未達の項目があれば修正してください。

一度に全部（Step1〜7の内容をまとめて）依頼することも可能だが、DoDが多いため、上記のように段階分けした方がレビューの精度は上がる。
