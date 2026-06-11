# Slack 完全再現仕様書 — 05. 検索

## 5.1 検索対象とインデックス
- 対象: メッセージ（自分がアクセス可能なチャンネル/DM/MPIM）、ファイル、Canvas、（People/Channels は別ファセット）。
- メッセージ投稿/編集/削除のたびに**非同期で検索インデックスを更新**（Elasticsearch/Solr）。
- インデックスは権限考慮: 検索結果は**そのユーザーがアクセス可能なもののみ**（private/DM は本人のみ）。
- インデックスフィールド: `text`(解析済), `user`, `channel`, `ts`, `has`(file/link/star/pin/reaction), `is`(thread), `react`, `before/after`, mention 抽出。

---

## 5.2 検索修飾子（完全再現リスト）

| 修飾子 | 例 | 意味 |
|---|---|---|
| `from:` | `from:@alice` | 投稿者 |
| `to:` | `to:@bob` | DM の宛先 |
| `in:` | `in:#general` / `in:@alice` | チャンネル/DM 限定 |
| `with:` | `with:@carol` | その人を含む DM/会話 |
| `before:` | `before:2026-01-01` | 日付以前 |
| `after:` | `after:yesterday` | 日付以降 |
| `on:` | `on:2026-01-01` | 特定日 |
| `during:` | `during:January` / `during:2025` | 期間 |
| `has:` | `has:link` `has:star` `has:pin` `has:reaction` `has::eyes:` | 含有条件 |
| `has:file` / 拡張子 | `has:pdf` | ファイル種別 |
| `is:` | `is:thread` `is:saved` `is:pinned` | 状態 |
| `-` 否定 | `-in:#random` | 除外 |
| `""` 完全一致 | `"exact phrase"` | フレーズ |

- 複数修飾子は AND。日付は相対語（today/yesterday/last week）対応。
- ハイライト表示、フィルタ UI（送信者/チャンネル/日付/種類）と修飾子は等価。

---

## 5.3 ランキング
- 既定の並び: **関連度 (relevance)** / **新しい順 (recent)** を切替。
- 関連度シグナル: テキスト一致スコア（TF-IDF/BM25）、新しさ、自分との関係（自分のチャンネル/自分宛/よく見るチャンネル）、リアクション/返信の多さ等。
- DM・自分宛・直近のものを軽くブースト。

---

## 5.4 検索 UI
- 上部検索バー。フォーカスで「最近の検索」「おすすめ修飾子」を提示。
- 結果は3タブ: **Messages / Files / Channels & People**（または統合 + サイドフィルタ）。
- 各メッセージ結果から: ジャンプ（文脈表示）、スレッドを開く、共有、保存。
- ファイル結果: プレビュー、ダウンロード、共有先。
- People/Channel 検索: クイックナビ（後述）と統合。

---

## 5.5 クイックスイッチャー / Jump to
- `Cmd/Ctrl+K`（または `Cmd+T`）: チャンネル/DM/人を素早く開くファジー検索。
- あいまい一致、最近開いた順、未読優先。
- 入力で channel/人/外部連絡先を横断サジェスト。

---

## 5.6 スコープ別検索
- ワークスペース内検索が基本。
- Enterprise Grid では**所属する複数 Workspace を横断**検索可能（org 検索）。
- 共有チャンネル（Slack Connect）は自分が見える範囲のみ。

---

## 5.7 再現チェックリスト
- [ ] 非同期インデックス（投稿→数秒で検索可能）
- [ ] 権限フィルタ（アクセス可能なものだけ）
- [ ] 全修飾子（from/in/has/before/after/is/否定/フレーズ）
- [ ] relevance/recent 切替
- [ ] Cmd+K クイックスイッチャー
- [ ] Messages/Files/People の3ファセット
