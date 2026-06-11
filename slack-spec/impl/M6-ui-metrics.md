# 実装仕様 M6 — UI 寸法・タイポグラフィ・コンポーネント状態の定義

> ピクセル単位の再現を目的に、デスクトップクライアントの計測値・状態遷移を定義する。
> 値は標準表示密度・100% ズーム時。

---

## 1. デザイントークン

### 1.1 カラー（コア）
```
brand.primary        #4A154B   (aubergine — レール/ブランド)
brand.blue           #36C5F0   brand.green #2EB67D
brand.yellow         #ECB22E   brand.red   #E01E5A
text.primary         #1D1C1D   text.secondary #616061
link                 #1264A3   link.hover  #0B4C8C
bg.primary           #FFFFFF   bg.secondary #F8F8F8
border               #DDDDDD   border.subtle #EBEAEB
presence.active      #2BAC76   presence.away  枠線のみ #616061
notification.badge   #CD2553   (赤バッジ)
composer.border      #8D8D8D → focus時 #1264A3 (1px→内側影)
mention.self.bg      #FCF3DA   mention.chip.bg #E8F5FA  mention.chip.fg #1264A3
dark.bg              #1A1D21   dark.sidebar #19171D     dark.text #D1D2D3
```
- サイドバーテーマはユーザー定義 8 色（column_bg, menu_bg(hover), active_item,
  active_item_text, hover_item, text_color, active_presence, badge）の組で表現し、
  プリセット（Aubergine / Hoth / Monument / …）はこの 8 色の定義済みセット。

### 1.2 タイポグラフィ
```
font.family   : "Lato", "Slack-Lato" 相当 (システムフォールバック: -apple-system, Segoe UI)
                日本語: "Noto Sans JP" / "Yu Gothic UI" フォールバック
message.body  : 15px / line-height 22px / weight 400
sender.name   : 15px / weight 900 (Black)
timestamp     : 12px / #616061 (hover で表示、左ガター内)
channel.header: 18px / weight 900
sidebar.item  : 15px / weight 400 (未読時 weight 900 + 白)
section.label : 13px / weight 700 / letter-spacing 0.8px
code          : "Monaco", "Consolas" 等幅 12px
```

---

## 2. レイアウト寸法

```
┌──┬─────────┬──────────────────────────────┬─────────┐
│A │    B    │              C               │    D    │
└──┴─────────┴──────────────────────────────┴─────────┘
A レール          : 幅 70px 固定。WSアイコン 36×36 radius 8、選択中 radius 4+白枠
B サイドバー      : 既定 260px、ドラッグで 220〜480px。背景=テーマ色
C メインペイン    : 残り全部。最小 400px
D 右ペイン        : 既定 400px、ドラッグ 320〜≈50%。閉時 0
ウィンドウ最小    : 800×600
チャンネルヘッダ  : 高さ 49px、border-bottom 1px
コンポーザ        : 最小 92px (ツールバー 36 + 入力 56)、入力は内容で最大 50% まで伸長
サイドバー行      : 高さ 28px、左 padding 16px、アイコン #/🔒 15px、間隔 8px
```

## 3. メッセージ行の構造（本流）

```
パディング: 上下 8px、左 20px、右 20px (hover 背景 #F8F8F8 はフル幅)
┌────────┬──────────────────────────────────────────┐
│ avatar │ [sender 15px black]  [time 12px grey]     │
│ 36×36  │ message body 15px/22px                    │
│ r=4    │ [reactions row] [thread bar]              │
└────────┴──────────────────────────────────────────┘
gutter 幅: 36 + 8 = 44px
```

### 連続投稿の集約（コアレッシング）
- 直前のメッセージと **同一投稿者かつ間隔 ≤ 5 分かつ間に日付境界・システム行が無い**
  → アバター・名前行を省略し本文のみ（上 padding 2px）。
- 集約行の hover 時、ガターに時刻 (12px) を表示。
- スレッド返信・broadcast・subtype 付きは集約しない。

### 日付区切り
- `─────  Today / Yesterday / June 11th, 2026  ─────` 中央ピル型ボタン
  (クリックでカレンダージャンプ)。高さ 28px、sticky（スクロール中も上端に滞留）。

### New（未読）ライン
- last_read 直後の位置に赤 1px 線 + 右端に `New` ラベル (12px 赤)。
- チャンネル滞在中に既読化が進んでも**ラインは表示位置に残す**（次回開き直しで消える）。

### リアクション行
- チップ: 高さ 24px、radius 12px、padding 0 8px、絵文字 16px + count 12px。
- 既押下（自分）チップ: 背景 #E8F5FA、枠 1px #1264A3。未押下: 背景 #F8F8F8、枠 #EBEAEB。
- 末尾に「+」絵文字追加ボタン（hover 時のみ可視ではなく、リアクションが 1 つでもあれば常時表示）。

### スレッドバー（reply_count > 0 の親の下）
```
[avatar×最大5 (20×20, -4px 重なり)]  N replies   Last reply 2 hours ago   [›]
高さ 34px。hover で背景白+枠+「View thread」右端表示。クリックで右ペインへ。
```

## 4. ホバーアクションバー
- メッセージ hover 後 **即時**（遅延なし）右上にフローティング表示。
- 構成（左→右）: 絵文字 ×3 (最頻使用) / 絵文字ピッカー / スレッド返信 / 共有 /
  保存 / その他(⋯)。各 32×32px、radius 4、hover 背景 #F8F8F8。
- `⋯` メニュー項目順: 未読にする / リマインダー / リンクをコピー / ピン留め /
  （編集 / 削除 — 自分のみ、削除は赤字最下段・区切り線上）/ アプリショートカット。

## 5. コンポーザ詳細
- 枠: radius 8px、枠線 1px #8D8D8D、focus で #1264A3。
- 上段ツールバー: B I S 〔リンク〕〔ol ul〕〔引用〕〔code codeblock〕— 選択中テキストに即時適用。
- 下段: ＋ / 書式トグル / 絵文字 / メンション / 録画 / 音声 / `/`短縮 … 右端に送信▶(青 #007A5A
  ※送信可能時 green、不能時 grey) + ▼(スケジュール送信)。
- placeholder: `#channel-name へのメッセージ`。
- Enter=送信 / Shift+Enter=改行（設定で反転）。IME 変換確定の Enter は送信しない
  （composition イベント中は送信抑制 — 日本語入力の必須要件）。
- ドラッグ&ドロップ: ウィンドウ全域がドロップ対象。オーバーレイ「#channel にアップロード」。
- 貼り付け: 画像クリップボード→添付プレビュー。テキスト >4000 文字→「スニペットにしますか?」。

## 6. 絵文字ピッカー
- サイズ 354×416px。上: 検索。中: カテゴリタブ (使用頻度/人/自然/食べ物/活動/旅行/物/記号/旗/カスタム)。
- グリッド 9 列、セル 32×32。hover で名前表示（下部プレビューバー 36px）。
- スキントーン選択は右下のデフォルトトーン設定に従う。
- 「Frequently used」はクライアント集計の使用頻度上位 18。

## 7. スクロール挙動
- 仮想リスト。**下端アンカー**: 最下部にいる時のみ新着で自動スクロール。
  スクロール中に新着 → 下端に「↓ 新しいメッセージ」ピル（クリックで最下部へ）。
- 上方向ロード: 残り 1.5 画面分でプリフェッチ (`conversations.history` 100件)。
- チャンネル切替時の復元位置: 未読あり→ New ライン直上 / 未読なし→ 最下部。
- 画像ロードによるレイアウトシフト禁止（サムネは寸法既知でプレースホルダ確保）。

## 8. 右ペイン（スレッドビュー）
- ヘッダ: `Thread` + `#channel-name`(リンク) + ×。
- 親メッセージ全文 → `N replies` 区切り線 → 返信列（集約規則は本流と同じ）→ 専用コンポーザ。
- コンポーザ下に `□ #channel にも投稿する`（broadcast チェックボックス）。
- 「Also send to channel」チェック状態は送信ごとにリセット（既定 off）。

## 9. 主要モーダル・パネル
- クイックスイッチャー (⌘K): 中央、幅 600px。入力行 44px + 結果 8 行（行 44px:
  アイコン/名前/所属 WS）。↑↓選択、Enter 遷移、結果はファジー一致スコア+最近度。
- チャンネル詳細: 右ペイン差し替え。タブ: About / Members / Integrations / Settings。
- プロフィール: 右ペイン。アバター 256 表示、名前/役職/TZ(現地時刻表示)/連絡先/
  フィールド、ボタン: メッセージ / Huddle / VIEW FULL PROFILE。

## 10. 空状態・エッジ表示
- 新規チャンネル: 「# channel-name へようこそ 🎉」+ 説明追加/メンバー招待ボタン。
- チャンネル先頭: 「ここが #x の始まりです」+ 作成者・作成日。
- 検索 0 件: イラスト + 「●●に一致する結果はありません」+ 修飾子のヒント。
- 接続断: 上部に黄色バナー「接続を試みています… [今すぐ再試行]」。復帰で 2 秒緑→消滅。
- 送信失敗: メッセージ左に赤 ⚠、本文グレー、「再送信 / 削除」リンク。

## 11. アニメーション規定
- 全トランジション 80〜200ms / ease-out。リアクション追加: チップが 1.2→1.0 scale で 150ms。
- 右ペイン開閉: width 200ms。モーダル: fade+scale(0.98→1) 120ms。
- 新着行の挿入はアニメーションなし（即時）。削除は高さ collapse 150ms。
- 「アニメーションを減らす」設定/OS 設定で全て無効化。

## 12. アクセシビリティ
- 全機能キーボード到達可能。F6 でペイン間フォーカス循環。
- メッセージリストは `role=list`、新着は aria-live=polite（フォーカス中チャンネルのみ）。
- コントラスト比: 本文 4.5:1 以上を全テーマで保証。フォーカスリングは 2px #1264A3。
