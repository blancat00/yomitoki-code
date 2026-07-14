# CLAUDE.md — よみときコード

## プロジェクト概要

「読む・直す・守る・学ぶ」の4機能を持つ、じぶん専用のプログラミング学習ノート。
claude.ai の artifacts 機能で作成した**単一HTMLファイルアプリ**（HTML/CSS/JSすべて `index.html` 1ファイル）。ビルド不要・npm不要。

## ファイル構成

```
yomitoki-code/
├── index.html    # アプリ本体（HTML/CSS/JSすべてここ）
├── glossary.js   # コード用語マスタ（約490語、ホバー解説の辞書）
├── README.md     # 利用者向け説明
└── CLAUDE.md     # このファイル
```

## タブ構成と実装場所（index.html内）

| タブ | 内容 | AI呼び出し |
|------|------|-----------|
| CH.01 学ぶ | 20テーマの教材 + 内蔵クイズ（`concepts` 配列にデータ、`/* LEARN */` 以降にUI） | 不要 |
| CH.02 読む | カメラでコードを撮影→OCR→glossary.jsと照合して下線+吹き出し解説（`/* READ: カメラで用語をよみとく */` セクション） | 不要（OCRはTesseract.js、端末内処理） |
| CH.03 直す | コード+エラー→原因仮説・直し方・再発防止（`/* DEBUG */` セクション） | 必要 |
| CH.04 守る | 脆弱性チェック（`/* SECURITY */` セクション） | 必要 |
| CH.05 コラム | 外部学習リソース集（静的HTML、`#panel-column`） | 不要 |

## 読むタブ（カメラOCR）の要点

- 2026-07-15にClaude依存を廃止し、カメラ+OCR方式に変更（出先でサブスク認証なしで使うため）
- 流れ: `getUserMedia`でカメラ起動 → canvasに静止画キャプチャ → **Tesseract.js**（CDNから初回のみ遅延読み込み、認識は端末内）でOCR → 単語のbboxを取得 → `glossary.js`と照合 → 該当語に下線オーバーレイ → タップで下線から棒(stick)を伸ばして吹き出し(callout)表示
- 撮影結果の表示は `object-fit:contain`。下線の座標計算も同じcontainの式（`Math.min`スケール）で合わせている。**coverに変えると座標がズレる**ので注意
- OCRは `fetch(url);` のように記号込みで単語を返すため、照合は「複合キー→ドット隣接ペア→内部の識別子」の順で緩く探す（`lookup`関数）
- カメラはHTTPS必須（localhost除く）。タブ切替時にストリームを停止する処理あり

## アーキテクチャの要点

- **AI呼び出し**: `callClaude(system, userText)` が唯一の入口。`https://api.anthropic.com/v1/messages` へ **APIキーなし** で fetch する。これは claude.ai artifacts 環境専用の仕組みで、**外部環境（ローカル/GitHub Pages）では動かない**（CORS/認証エラー）。外部で動かすにはAPIキーを保持するプロキシサーバーが必要。**APIキーをフロントに直書きしないこと**。
- **AI応答の形式**: 各タブのプロンプトで「JSON形式のみ」を指定し、`parseJSON()`（```json フェンス除去 + JSON.parse）で解析。失敗時は try/catch でエラーボックス表示。
- **データ保存**: artifacts の `window.storage` を使用。存在しない環境では冒頭のシムが `localStorage`（キー接頭辞 `yomitoki:`）にフォールバックする。保存対象はクイズの成績（`learn_done`）と各タブの履歴（`read_history` / `debug_history` / `sec_history`、各5件まで）。
- **XSS対策**: AI応答・ユーザー入力の表示は必ず `escapeHtml()` / `formatAiText()` を通す。新しい表示処理を足すときも同様にすること。

## 用語ホバー解説（glossary.js）

コードブロック（`pre.codeblock` / `code.inline`）内の単語にカーソルをのせると、`glossary.js` のマスタを参照して解説ツールチップが出る。

- マスタは `window.YOMITOKI_GLOSSARY = { "単語": {cat, desc}, ... }` の形。**1行足すだけで語彙を増やせる**
- `console.log` のようなドット付きキー、`=>` のような記号キー、`border-radius` のようなハイフン付きキーに対応
- 実装は index.html 末尾の `/* GLOSSARY HOVER */` セクション。`caretRangeFromPoint` でカーソル下の文字を特定 → トークン候補を組み立て → マスタを引く
- glossary.js が読み込まれていない環境（artifacts単体など）では自動的に無効化される（エラーにならない）
- ツールチップの表示は `escapeHtml()` を通している。マスタのdescにHTMLは書かない

## 教材データの編集

「学ぶ」タブの教材は `concepts` 配列（`/* LEARN: 教材データ */` セクション）。1テーマ = `{id, cat, title, desc, body, quiz}`。

- `body` 内のコード例は `cb()`（コードブロック）/ `ic()`（インラインコード）ヘルパーで必ずエスケープする
- `quiz` は `{q, choices, answer(正解のindex), explain}` の配列、各テーマ2問

## ローカル確認

```bash
npx serve yomitoki-code    # または .claude/launch.json の "yomitoki-code" 設定
```

「学ぶ」「読む」「コラム」タブは完全動作（読むのOCRエンジン読み込みに初回のみ通信が必要）。「直す」「守る」はAI呼び出しが失敗しエラー表示になる（仕様通り）。
