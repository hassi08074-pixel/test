# Slack 完全再現仕様書 — 09. プラットフォーム（Apps / Bot / Webhook / Block Kit / Workflow）

> Slack の競争優位の中核。これを忠実に作らないと「ただのチャット」になる。

## 9.1 App モデルと配布
- **App**: スコープ・機能（bot/コマンド/Webhook/イベント購読/ショートカット/ホームタブ等）の束。
- 配布: 単一 Workspace 用 / **公開 App Directory**（マーケットプレイス、審査あり）。
- インストール = OAuth 認可で team にスコープ付きトークン発行（`01` の `app_installations`）。
- 管理者承認制（Approved Apps 一覧、ホワイト/ブラックリスト）。

---

## 9.2 OAuth 2.0 / スコープ
- 認可フロー: `oauth/v2/authorize?client_id&scope&redirect_uri&state` → 同意 → `code` → `oauth.v2.access` で交換。
- **トークン種別**:
  - `xoxb-` **Bot token**（bot ユーザーとして動作。team 単位）
  - `xoxp-` **User token**（ユーザー本人として動作。`user_scope` 認可時）
  - `xapp-` **App-level token**（Socket Mode / 一部 API）
  - `xoxe-` refresh（トークンローテーション有効時）
- **スコープ例**: `chat:write`, `channels:read`, `channels:history`, `groups:*`, `im:*`, `users:read`, `files:write`, `reactions:write`, `commands`, `app_mentions:read`, `chat:write.public`, `channels:manage` …
- スコープは「最小権限」。bot/user で別系統。トークンローテーション（短命＋refresh）対応。

---

## 9.3 Incoming Webhook
- 特定チャンネル宛の**投稿専用 URL**。`POST {text, blocks, attachments}` でメッセージ投稿。
- インストール時にチャンネルを選択し URL 生成。シンプル通知連携の定番（CI/監視）。
- 投稿者表示名/アイコンをペイロードで上書き可（設定により制限）。

## 9.4 Outgoing Webhook（レガシー）/ Events API へ移行
- 旧: 特定トリガワードで外部 URL へ POST。現在は **Events API** 推奨。

## 9.5 Slash Commands
- `/command text` をユーザーが入力 → Slack が登録 URL へ POST:
  - ペイロード: `command, text, user_id, channel_id, team_id, response_url, trigger_id`。
- アプリは **3秒以内に HTTP 応答**（ack）。
  - 即時応答 or 空 200 → 後で `response_url`（30分有効、5回まで）に遅延応答。
- 応答可視性: `response_type: in_channel`（全員に見える）/ `ephemeral`（本人のみ）。
- `trigger_id`（3秒有効）で**モーダル**を開ける。

---

## 9.6 Events API（イベント購読）
- アプリは購読したイベント（`message.channels`, `app_mention`, `reaction_added`, `team_join`, `link_shared`, `member_joined_channel` …）を**HTTPS POST**で受け取る。
- **URL 検証ハンドシェイク**: 初回 `{type:"url_verification", challenge}` に `challenge` をそのまま返す。
- **署名検証**: `X-Slack-Signature` = `v0=HMAC-SHA256(signing_secret, "v0:"+timestamp+":"+body)`。`X-Slack-Request-Timestamp` の鮮度（±5分）でリプレイ防止。**必須再現**。
- **3秒以内に 200**。重い処理は非同期。
- リトライ: 失敗時は最大3回再送（`X-Slack-Retry-Num/Reason`）。冪等に処理。
- イベントエンベロープ: `{token, team_id, api_app_id, event:{...}, event_id, event_time, authorizations}`。

---

## 9.7 Socket Mode
- 公開 URL を用意できない環境向け。アプリが**WebSocket でイベントを受信**。
- `apps.connections.open`（app-level token `xapp-`）で WS URL 取得 → 接続。
- イベント/インタラクション/コマンドを WS で受け、`ack` を返す。
- 開発・社内ツール向け。本番大規模は HTTP Events API 推奨。

---

## 9.8 Interactivity（ボタン/モーダル/メニュー）
- ユーザーが Block Kit のボタン/選択/日付ピッカー等を操作 → Slack が**Interactivity Request URL** へ POST（署名付き）。
- ペイロード: `type(block_actions|view_submission|view_closed|shortcut|message_action), trigger_id, user, actions[], view, response_url`。
- 応答: 3秒以内。`response_action`（モーダル更新/エラー表示/クローズ）、`views.open/update/push/publish`。

### モーダル (Views)
- `views.open(trigger_id, view)` でモーダル表示。`view` は Block Kit（`input` ブロックで入力収集）。
- 多段（push でスタック）、`view_submission` で値受領（バリデーションエラー返却可）。
- **App Home**: アプリ専用のホームタブ（`views.publish` で `home` ビュー更新）。Messages タブと併用。

---

## 9.9 Block Kit（UI 仕様）

メッセージ/モーダル/ホームを構成する宣言的 UI。**ブロック型を網羅再現**:

| Block | 用途 |
|---|---|
| `section` | テキスト＋任意の accessory（ボタン/画像/選択） |
| `rich_text` | ユーザー投稿の正体（段落/リスト/コード/引用/quote） |
| `header` | 大見出し |
| `divider` | 区切り線 |
| `context` | 小さな補助テキスト＋画像（メタ情報） |
| `actions` | ボタン/選択などインタラクティブ要素の行 |
| `input` | モーダルの入力フィールド |
| `image` | 画像 |
| `file` | リモートファイル参照 |
| `video` | 動画 |
| `call` | 通話カード |

### Block Elements（要素）
- `button`（style: primary/danger, url, confirm dialog, value, action_id）
- `static_select` / `external_select`（動的オプション URL）/ `users_select` / `channels_select` / `conversations_select` / `multi_*_select`
- `overflow`, `datepicker`, `timepicker`, `datetimepicker`
- `plain_text_input`（multiline, max/min length）, `number_input`, `email_text_input`, `url_text_input`
- `checkboxes`, `radio_buttons`
- `image`, `rich_text_input`

### Text objects
- `plain_text`（emoji フラグ）/ `mrkdwn`（verbatim フラグ）。
- 各要素・ブロックに**文字数/個数上限**あり（例: section text 3000、ボタン text 75、blocks/message 50、actions 内要素 25 等）— 正確に踏襲。

### action_id / block_id
- インタラクション識別子。値は `value` や `selected_*` で受領。

---

## 9.10 Workflow Builder
- ノーコードの自動化ビルダー。**トリガ → ステップ**の連鎖。
- トリガ: ショートカット（メニュー/メッセージ）、スケジュール、Webhook、絵文字リアクション、チャンネル参加、フォーム送信、新メンバー参加。
- ステップ: メッセージ送信、フォーム収集、外部サービス（コネクタ）呼び出し、人への割当、分岐/遅延（高度版）。
- カスタムステップ: アプリが提供する関数（function）を Workflow から呼べる（次世代プラットフォーム）。
- 変数受け渡し、フォーム入力、承認フロー。

---

## 9.11 メッセージ投稿 API（プラットフォーム視点）
- `chat.postMessage`（text/blocks/attachments, thread_ts, reply_broadcast, unfurl_links, username/icon 上書き）。
- `chat.postEphemeral`（特定ユーザーにのみ見える一時メッセージ）。
- `chat.update` / `chat.delete` / `chat.scheduleMessage`。
- `chat.unfurl`（カスタムリンク展開）。
- bot 投稿は `as_user`/bot プロフィール、blocks 主体。

---

## 9.12 Bot の振る舞い
- bot は `users` 行＋`bot_users` を持つ一級ユーザー。メンション(`app_mention`)・DM・チャンネル招待で起動。
- bot は招待されたチャンネルのみ閲覧/投稿（`chat:write` + チャンネル所属）。
- レート制限・スコープ・署名検証を遵守。

---

## 9.13 再現チェックリスト（プラットフォーム）
- [ ] OAuth v2（bot/user/app トークン、スコープ、ローテーション）
- [ ] Incoming Webhook
- [ ] Slash Command（3秒 ack + response_url 遅延 + ephemeral/in_channel + trigger_id）
- [ ] Events API（url_verification、署名 HMAC、3秒 200、リトライ冪等）
- [ ] Socket Mode（xapp、WS、ack）
- [ ] Interactivity（block_actions/view_submission、views.open/update/push/publish、App Home）
- [ ] Block Kit 全ブロック/要素/テキストオブジェクト/上限値
- [ ] Workflow Builder（トリガ/ステップ/フォーム/カスタム関数）
- [ ] chat.postMessage/postEphemeral/update/delete/schedule/unfurl
