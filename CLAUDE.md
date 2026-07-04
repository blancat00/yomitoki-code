# CLAUDE.md — よみときコード

## プロジェクト概要

「読む・直す・守る・学ぶ」の4機能を持つ、じぶん専用のプログラミング学習ノート。
claude.ai の artifacts 機能で作成した**単一HTMLファイルアプリ**（HTML/CSS/JSすべて `index.html` 1ファイル）。ビルド不要・npm不要。

## ファイル構成

```
yomitoki-code/
├── index.html   # アプリ本体（約1150行、HTML/CSS/JSすべてここ）
├── README.md    # 利用者向け説明
└── CLAUDE.md    # このファイル
```

## タブ構成と実装場所（index.html内）

| タブ | 内容 | AI呼び出し |
|------|------|-----------|
| CH.01 学ぶ | 20テーマの教材 + 内蔵クイズ（`concepts` 配列にデータ、`/* LEARN */` 以降にUI） | 不要 |
| CH.02 読む | コード貼り付け→注釈付き解説（`/* READ */` セクション） | 必要 |
| CH.03 直す | コード+エラー→原因仮説・直し方・再発防止（`/* DEBUG */` セクション） | 必要 |
| CH.04 守る | 脆弱性チェック（`/* SECURITY */` セクション） | 必要 |
| CH.05 コラム | 外部学習リソース集（静的HTML、`#panel-column`） | 不要 |

## アーキテクチャの要点

- **AI呼び出し**: `callClaude(system, userText)` が唯一の入口。`https://api.anthropic.com/v1/messages` へ **APIキーなし** で fetch する。これは claude.ai artifacts 環境専用の仕組みで、**外部環境（ローカル/GitHub Pages）では動かない**（CORS/認証エラー）。外部で動かすにはAPIキーを保持するプロキシサーバーが必要。**APIキーをフロントに直書きしないこと**。
- **AI応答の形式**: 各タブのプロンプトで「JSON形式のみ」を指定し、`parseJSON()`（```json フェンス除去 + JSON.parse）で解析。失敗時は try/catch でエラーボックス表示。
- **データ保存**: artifacts の `window.storage` を使用。存在しない環境では冒頭のシムが `localStorage`（キー接頭辞 `yomitoki:`）にフォールバックする。保存対象はクイズの成績（`learn_done`）と各タブの履歴（`read_history` / `debug_history` / `sec_history`、各5件まで）。
- **XSS対策**: AI応答・ユーザー入力の表示は必ず `escapeHtml()` / `formatAiText()` を通す。新しい表示処理を足すときも同様にすること。

## 教材データの編集

「学ぶ」タブの教材は `concepts` 配列（`/* LEARN: 教材データ */` セクション）。1テーマ = `{id, cat, title, desc, body, quiz}`。

- `body` 内のコード例は `cb()`（コードブロック）/ `ic()`（インラインコード）ヘルパーで必ずエスケープする
- `quiz` は `{q, choices, answer(正解のindex), explain}` の配列、各テーマ2問

## ローカル確認

```bash
npx serve yomitoki-code    # または .claude/launch.json の "yomitoki-code" 設定
```

「学ぶ」「コラム」タブは完全動作。「読む」「直す」「守る」はAI呼び出しが失敗しエラー表示になる（仕様通り）。
