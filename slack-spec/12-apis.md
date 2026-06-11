# Slack 完全再現仕様書 — 12. Web API / Events API / Socket Mode / レート制限

## 12.1 Web API の様式
- エンドポイント: `https://slack.com/api/<method>`（例: `chat.postMessage`）。
- 命名: `<名詞>.<動詞>`（`conversations.history`, `users.info`, `chat.update`）。
- 認証: `Authorization: Bearer xoxb-...`（または `xoxp-`）。
- 入力: `application/x-www-form-urlencoded` または `application/json`（JSON 必須メソッドあり: blocks 等）。
- 出力: 常に `{"ok": true|false, ...}`。失敗時 `{"ok":false,"error":"<code>"}`。HTTP は基本 200（エラーも body で表現）。
- ページング: カーソル方式 `cursor` / `response_metadata.next_cursor` + `limit`。
- `warning` フィールド、`response_metadata.messages` で非致命警告。

---

## 12.2 主要メソッド一覧（再現対象コア）

### conversations.*（チャンネル/DM/MPIM 統合）
- `conversations.list`（types=public_channel,private_channel,mpim,im / exclude_archived / cursor）
- `conversations.history`（channel, oldest, latest, inclusive, limit, cursor）
- `conversations.replies`（channel, ts → スレッド取得）
- `conversations.info / create / rename / archive / unarchive / setTopic / setPurpose`
- `conversations.join / leave / invite / kick`
- `conversations.members`
- `conversations.mark`（既読位置更新）
- `conversations.open`（DM/MPIM を開く・取得）

### chat.*
- `chat.postMessage` / `chat.postEphemeral` / `chat.update` / `chat.delete`
- `chat.scheduleMessage` / `chat.deleteScheduledMessage` / `chat.scheduledMessages.list`
- `chat.meMessage` / `chat.unfurl` / `chat.getPermalink`

### users.*
- `users.info / list / lookupByEmail / profile.get / profile.set`
- `users.setPresence / getPresence / setActive`
- `users.conversations`（自分の会話一覧）

### files.*
- `files.getUploadURLExternal` / `files.completeUploadExternal`（現行）
- `files.info / list / delete / sharedPublicURL / revokePublicURL`

### reactions / pins / stars(saved) / bookmarks
- `reactions.add / remove / get / list`
- `pins.add / remove / list`
- `stars.add / remove / list`（Saved）
- `bookmarks.add / edit / remove / list`

### search.*
- `search.messages`（query + 修飾子, sort, count, page/cursor, highlight）
- `search.files` / `search.all`

### usergroups.*
- `usergroups.list / create / update / disable / enable`
- `usergroups.users.list / update`

### emoji / team / dnd / reminders
- `emoji.list`
- `team.info / team.profile.get`
- `dnd.info / setSnooze / endSnooze / endDnd / teamInfo`
- `reminders.add / list / info / delete / complete`

### views / interactivity（プラットフォーム）
- `views.open / update / push / publish`
- `apps.connections.open`（Socket Mode）

### admin.*（Grid 管理）
- `admin.users.* / admin.conversations.* / admin.teams.* / admin.apps.*` 等。

### auth / oauth
- `auth.test`（トークン検証・identity）/ `auth.revoke`
- `oauth.v2.access`（code→token 交換）

---

## 12.3 RTM / イベント配信（再掲・正式化）
- 旧 `rtm.connect`/`rtm.start` で WS URL を取得 → イベントストリーム（02 参照）。
- 現行クライアントは内部の boot + flannel + WS を使用。サードパーティは **Events API（HTTP push）/ Socket Mode** が推奨。

---

## 12.4 Events API（再掲・要点）
- 購読イベントを **HTTPS POST** で受信。エンベロープ `{event, event_id, event_time, team_id, authorizations}`。
- **URL 検証**（challenge エコー）、**署名検証**（`v0=HMAC-SHA256`）、**3秒 200**、**リトライ冪等**。
- イベント種別: `message.*`, `app_mention`, `reaction_added/removed`, `member_joined_channel`, `team_join`, `link_shared`, `channel_*`, `file_*`, `app_home_opened`, `tokens_revoked` ほか。

---

## 12.5 Socket Mode（再掲）
- `xapp-` app-level token + `connections:write`。
- `apps.connections.open` → WS。イベント/コマンド/インタラクションを受信、各メッセージに `envelope_id` で `ack`。

---

## 12.6 レート制限（Tier 制・忠実再現）

メソッドごとに Tier が割り当てられ、概ね per-app / per-workspace のトークンバケット:

| Tier | 目安（req/min） |
|---|---|
| Tier 1 | ~1+/min（重い管理系） |
| Tier 2 | ~20/min |
| Tier 3 | ~50/min |
| Tier 4 | ~100/min |
| 特例 | `chat.postMessage` ≈ 1 msg/sec/channel（短バースト許容） |

- 超過時: **HTTP 429** + `Retry-After`（秒）。クライアントは指数バックオフ。
- `ok:false, error:"ratelimited"` も併用。
- Events API はアプリ全体のイベント流量に上限（超過分は配信ドロップ＋警告）。

---

## 12.7 エラーコード規約
- 文字列コード: `channel_not_found`, `not_in_channel`, `not_authed`, `invalid_auth`, `account_inactive`, `missing_scope`（＋ `needed`/`provided`）, `is_archived`, `msg_too_long`, `rate_limited`, `cant_update_message`, `message_not_found`, `restricted_action` …
- スコープ不足は `missing_scope` に必要 scope を明示。

---

## 12.8 Webhook ペイロード署名（再掲・必須）
```
basestring = "v0:" + X-Slack-Request-Timestamp + ":" + raw_body
signature  = "v0=" + hex(HMAC_SHA256(signing_secret, basestring))
検証: constant-time compare( signature, X-Slack-Signature )
     かつ |now - timestamp| <= 300s （リプレイ防止）
```

---

## 12.9 再現チェックリスト（API）
- [ ] `{ok:...}` 一貫レスポンス＋文字列エラーコード
- [ ] conversations/chat/users/files/reactions/pins/search/usergroups/dnd/reminders/views 主要メソッド
- [ ] カーソルページング
- [ ] OAuth v2、auth.test、トークン種別
- [ ] Events API（検証/署名/3秒/リトライ）と Socket Mode（ack/envelope_id）
- [ ] Tier ベースのレート制限と 429+Retry-After
- [ ] HMAC 署名検証＋リプレイ防止
```
```
