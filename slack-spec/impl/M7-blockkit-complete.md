# 実装仕様 M7 — Block Kit 全要素・全フィールド・全上限値リファレンス

> バリデータと描画エンジンをこの1枚で書けることを目標とする。
> 表記: ✔=必須、文字数は UTF-8 コードポイント数。

---

## 0. 配置可能性マトリクス（surface × block）

| Block | message | modal | home | 備考 |
|---|---|---|---|---|
| section | ✔ | ✔ | ✔ | |
| rich_text | ✔ | ✔(表示) | ✔ | ユーザー投稿の正本 |
| divider | ✔ | ✔ | ✔ | |
| header | ✔ | ✔ | ✔ | |
| context | ✔ | ✔ | ✔ | |
| image | ✔ | ✔ | ✔ | |
| actions | ✔ | ✔ | ✔ | |
| input | ✖(注1) | ✔ | ✔ | 注1: message 内 input は workflow フォーム面のみ |
| file | ✔ | ✖ | ✖ | remote file 専用 |
| video | ✔ | ✖ | ✖ | |
| call | ✔ | ✖ | ✖ | システム生成のみ |

**ブロック数上限**: message=50 / modal=100 / home=100。
**block_id**: 任意文字列 ≤255。同一 surface 内で一意（重複→ `invalid_blocks`）。省略時はサーバが採番。

---

## 1. Composition objects（共通部品）

### 1.1 text object
```jsonc
{ "type": "plain_text" | "mrkdwn",
  "text": "...",            // 必須。最小1
  "emoji": true,            // plain_text のみ。:smile: を絵文字化するか
  "verbatim": false }       // mrkdwn のみ。true でリンク/メンション自動解釈を停止
```

### 1.2 confirm dialog（破壊的操作の確認）
| field | 型 | 必須 | 上限 |
|---|---|---|---|
| title | plain_text | ✔ | 100 |
| text | text | ✔ | 300 |
| confirm | plain_text | ✔ | 30 |
| deny | plain_text | ✔ | 30 |
| style | `primary`/`danger` | – | danger=確認ボタン赤 |

### 1.3 option
| field | 必須 | 上限 |
|---|---|---|
| text (plain_text / overflow・select は mrkdwn 不可、radio/checkbox は mrkdwn 可) | ✔ | 75 |
| value | ✔ | 150 |
| description (plain_text) | – | 75 |
| url (overflow のみ) | – | 3000 |

### 1.4 option_group
- `label` plain_text ≤75 ✔ / `options` ≤100 ✔。グループ数 ≤100。

### 1.5 dispatch_action_config（text input の発火条件）
- `trigger_actions_on`: `["on_enter_pressed"]` / `["on_character_entered"]` / 両方。

---

## 2. Blocks 詳細

### 2.1 section
| field | 必須 | 制約 |
|---|---|---|
| text | text/fields どちらか✔ | ≤3000 |
| fields | 〃 | 配列 ≤10、各 text ≤2000。2列グリッド描画 |
| accessory | – | element 1個（右側に配置） |
| expand | – | bool。true で「Show more」折りたたみを無効化 |

accessory に置ける element: button, 全 select/multi_select, overflow, datepicker,
timepicker, image, checkboxes, radio_buttons, workflow_button。

### 2.2 header
- `text` plain_text ✔ ≤150。24px 太字描画。

### 2.3 context
- `elements` ✔ ≤10。各要素は `image` または text object。12px グレー描画、画像は 20×20。

### 2.4 image (block)
| field | 必須 | 制約 |
|---|---|---|
| image_url または slack_file | ✔ | url ≤3000。https のみ |
| alt_text | ✔ | ≤2000 |
| title | – | plain_text ≤2000 |
- 描画: 最大幅 360px(インライン)、アスペクト維持、クリックでライトボックス。
- 取得失敗時: alt_text + 壊れ画像アイコンを枠内表示（レイアウト崩し禁止）。

### 2.5 actions
- `elements` ✔ ≤25（描画上は1行に収まらなければ折返し）。
- 置ける element: button, 全 select 系, overflow, datepicker, timepicker,
  datetimepicker, checkboxes, radio_buttons, workflow_button。

### 2.6 input
| field | 必須 | 制約 |
|---|---|---|
| label | ✔ | plain_text ≤2000 |
| element | ✔ | 入力系 element 1個 |
| hint | – | plain_text ≤2000 |
| optional | – | bool。false(既定)なら未入力で submit 時に赤エラー |
| dispatch_action | – | true で値変更時に block_actions を発火 |

### 2.7 video
| field | 必須 | 制約 |
|---|---|---|
| video_url | ✔ | 埋め込み可能な https URL ≤3000 |
| thumbnail_url | ✔ | ≤3000 |
| alt_text / title | ✔ | title ≤200 |
| title_url | – | https のみ |
| description | – | ≤200 |
| provider_name / provider_icon_url / author_name | – | author ≤50 |
- 要 OAuth scope `links.embed:write` 相当の検証。インラインプレイヤー描画。

### 2.8 file
- `external_id` ✔ / `source:"remote"` ✔。files.remote 登録済みファイルのカード描画。

### 2.9 rich_text
- M2 §8 のスキーマに準拠。アプリも投稿可能（list/quote/preformatted/section）。

---

## 3. Block elements 詳細

### 3.1 button
| field | 必須 | 制約 |
|---|---|---|
| text | ✔ | plain_text ≤75（描画は ~30 文字で省略記号） |
| action_id | ✔ | ≤255、同一ブロック内一意 |
| value | – | ≤2000 |
| url | – | ≤3000。クリックで開く（block_actions も同時発火） |
| style | – | 省略=グレー / `primary`=緑 #007A5A / `danger`=赤 #E01E5A |
| confirm | – | confirm dialog |
| accessibility_label | – | ≤75 |

### 3.2 select 系（単一選択）共通
- `placeholder` plain_text ≤150 / `action_id` ✔ / `confirm` / `focus_on_load`。

| type | 固有 field | 規定 |
|---|---|---|
| static_select | `options` ≤100 **or** `option_groups` ≤100 ✔ / `initial_option` | クライアント内フィルタ検索付き |
| external_select | `min_query_length`(既定3) | 入力毎にアプリの options URL へ問合せ。**応答3秒以内** `{options:[...]}` |
| users_select | `initial_user` | メンバーから選択。削除済み除外 |
| conversations_select | `initial_conversation` / `default_to_current_conversation` / `filter` | filter: `{include:[im,mpim,private,public], exclude_bot_users, exclude_external_shared_channels}` |
| channels_select | `initial_channel` | public のみ |

### 3.3 multi_*_select
- 上記の複数版。`max_selected_items`(≥1) / `initial_options|users|channels|conversations`。
- 選択済みはトークンチップで表示、×で除去。

### 3.4 overflow
- `options` 2〜5 ✔。「⋯」ボタン→ドロップダウン。option に url 可。

### 3.5 datepicker / timepicker / datetimepicker
| type | initial | 出力値 |
|---|---|---|
| datepicker | `initial_date` "YYYY-MM-DD" | selected_date 同形式 |
| timepicker | `initial_time` "HH:mm" | selected_time。表示は 12/24h をユーザー設定に従う |
| datetimepicker | `initial_date_time` unix秒 | selected_date_time unix秒（**ユーザーTZで描画**） |

### 3.6 plain_text_input
| field | 規定 |
|---|---|
| multiline | true で高さ 3 行〜自動伸長 |
| min_length / max_length | 0〜3000。submit 時検証、違反は赤字エラー |
| initial_value | ≤3000 |
| dispatch_action_config | §1.5 |

### 3.7 number_input
- `is_decimal_allowed` ✔ / `min_value` / `max_value`（文字列で指定）/ `initial_value`。
- 数値以外の入力は submit 時 `errors` ではなく入力時点で弾く。

### 3.8 email_text_input / url_text_input
- フォーマット検証内蔵（RFC 形式 / http(s) スキーム）。違反は submit 時エラー。

### 3.9 checkboxes / radio_buttons
- `options` ≤10 ✔（checkboxes は `initial_options`、radio は `initial_option`）。
- option.text に mrkdwn 可（リンク含む）。description は 12px グレーで2行目。

### 3.10 image (element)
- `image_url` ✔ / `alt_text` ✔。section accessory では 88×88、context では 20×20。

### 3.11 rich_text_input
- リッチテキスト編集欄（modal 用）。出力は rich_text ブロック。
- `initial_value` は rich_text ブロック。`placeholder` ≤150。

### 3.12 workflow_button
- `text` ≤75 ✔ / `workflow:{trigger:{url, customizable_input_parameters}}` ✔。
- クリックでワークフロー起動（block_actions はアプリに飛ばない）。

---

## 4. インタラクションペイロード（受信側契約）

### 4.1 block_actions
```jsonc
{ "type":"block_actions",
  "user":{"id","username","team_id"}, "team":{"id","domain"},
  "channel":{"id","name"},                  // message 起点のみ
  "message":{ /* 元メッセージ全体 */ },      // 〃
  "view":{ /* modal/home 起点のみ */ },
  "container":{"type":"message"|"view", "message_ts","channel_id","is_ephemeral"},
  "trigger_id":"…",                          // 3秒有効。views.open に使える
  "response_url":"…",                        // message 起点のみ。30分/5回
  "actions":[{ "block_id","action_id","type",
               "value"|"selected_option"|"selected_user"|"selected_date"|…,
               "action_ts" }] }              // 1操作=1要素
```
- 応答: **200 空ボディを3秒以内**。UI 更新は response_url / chat.update / views.update で行う。

### 4.2 view_submission の state
```jsonc
"view": { "state": { "values": {
  "<block_id>": { "<action_id>": { "type":"plain_text_input", "value":"…" }}}},
  "private_metadata":"…", "callback_id":"…" }
```
- 全 input ブロックの現在値が `values[block_id][action_id]` に入る。
- 型別の値キー: value / selected_option(s) / selected_user(s) / selected_channel(s) /
  selected_conversation(s) / selected_date / selected_time / selected_date_time / rich_text_value。

---

## 5. バリデーション実装順序（invalid_blocks の決定性）
```
1. JSON スキーマ適合（未知 type → invalid_blocks）
2. surface 配置可否（§0 マトリクス）
3. ブロック数上限 → 各ブロックの必須 field → 文字数/個数上限
4. block_id / action_id 一意性
5. URL スキーム（https 必須箇所）・日付/時刻フォーマット
エラー応答: {"ok":false,"error":"invalid_blocks",
  "response_metadata":{"messages":["[ERROR] blocks[3]: text exceeds 3000 …"]}}
   ※ messages に index 付きで全違反を列挙する（最初の1件で止めない）
```

---

## 6. フォールバック描画規定
- blocks 非対応面（プッシュ通知・メール・検索結果・引用カード）では `text` を使用。
- アプリが text を省略した場合、blocks から自動生成: section.text → header →
  context の順に連結し 200 文字で切る。生成不能なら "This content can't be displayed."
