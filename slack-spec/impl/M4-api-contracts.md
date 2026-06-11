# 実装仕様 M4 — コア API のフィールド単位契約

> 主要メソッドを「リクエスト全引数・バリデーション・全エラー・応答スキーマ」の
> 粒度で定義する。ここに無い挙動は M1/M3 の規定に従う。

---

## 0. 共通規約

```
URL      : POST https://slack.com/api/<method>
Auth     : Authorization: Bearer xox{b|p}-…
Content  : application/json; charset=utf-8（blocks を含むものは JSON 必須）
応答     : HTTP 200 + {"ok":bool, ...}     ※レート超過のみ HTTP 429
ページング: cursor 方式。応答の response_metadata.next_cursor が "" なら終端
時刻     : すべて ts 文字列 "sec.μseq" または unix 秒
```

共通エラー（全メソッドで起こり得る）:
`not_authed` `invalid_auth` `account_inactive` `token_revoked` `missing_scope`
`ratelimited` `accesslimited` `request_timeout` `fatal_error`

---

## 1. chat.postMessage

| 引数 | 型 | 必須 | 制約・規定 |
|---|---|---|---|
| `channel` | string | ✔ | C/D/G の ID。または `#name`（public のみ解決） |
| `text` | string | blocks 無時✔ | ≤40,000 文字。blocks 有時は通知フォールバック文 |
| `blocks` | array | – | ≤50 個。M2/M6 のスキーマ検証 |
| `attachments` | array | – | ≤20 個（レガシー） |
| `thread_ts` | ts | – | 親 ts。返信 ts を渡された場合は親に正規化 (M1§3) |
| `reply_broadcast` | bool | – | thread_ts 必須。本流にも表示 |
| `unfurl_links` / `unfurl_media` | bool | – | 既定 true/true（bot は false/true） |
| `mrkdwn` | bool | – | false で text を書式解釈しない |
| `parse` | enum | – | `none`(既定)/`full`。full は平文 @user 等も自動リンク化 |
| `link_names` | bool | – | parse=none でも @name をリンク化（レガシー互換） |
| `username` / `icon_url` / `icon_emoji` | string | – | bot 投稿の表示上書き（`chat:write.customize` 要） |
| `metadata` | object | – | `{event_type, event_payload}` アプリ用メタ |

バリデーション順序（先に当たったエラーを返す）:
`channel_not_found` → `is_archived` → `not_in_channel` → `restricted_action`
→ `msg_too_long` → `no_text`(text/blocks/attachments 全て空) → `invalid_blocks`
→ `rate_limited`

応答:
```jsonc
{ "ok": true, "channel": "C…", "ts": "1718089543.000412",
  "message": { /* 確定した完全なメッセージオブジェクト */ } }
```

---

## 2. conversations.history

| 引数 | 型 | 必須 | 規定 |
|---|---|---|---|
| `channel` | string | ✔ | 呼び出し主体が閲覧権を持つこと → `channel_not_found`（存在を漏らさない） |
| `latest` | ts | – | 既定=now。この ts **以前**を返す |
| `oldest` | ts | – | 既定=0 |
| `inclusive` | bool | – | latest/oldest 自体を含むか。既定 false |
| `limit` | int | – | 1..999、既定 100。実際の返却はこれ以下 |
| `cursor` | string | – | 前回の next_cursor |
| `include_all_metadata` | bool | – | metadata 同梱 |

応答:
```jsonc
{ "ok": true,
  "messages": [ /* ts 降順（新しい→古い） */ ],
  "has_more": true,
  "pin_count": 3,
  "response_metadata": { "next_cursor": "bmV4dF90czox…" } }
```
規定:
- 返るのは**本流のみ**（スレッド返信は thread_broadcast を除き含まれない）。
  親メッセージには reply_count 等のメタが付く。返信本文は `conversations.replies`。
- tombstone は `subtype:"tombstone"` で返す（返信ゼロなら返さない）。
- cursor は `latest/oldest` と排他的に優先される。

## 3. conversations.replies
- 引数: `channel, ts(親), cursor, limit, oldest, latest, inclusive`
- 応答 `messages[0]` は**必ず親**、以降が返信の ts 昇順。
- 親が返信でも可（親に正規化して返す）。`thread_not_found` あり。

## 4. conversations.mark
- 引数: `channel, ts`。`ts` を last_read として保存。
- 規定: 過去方向への巻き戻し可（Mark unread の実装）。保存後、同一ユーザーの
  全接続へ `channel_marked` を配信（M3§4）。`unread_count_display` はサーバ再計算値。

## 5. conversations.open
- 引数: `users`（カンマ区切り 1..8 人）または `channel`（既存 D/G の ID）, `return_im`
- 規定: 同一メンバー集合の既存 DM/MPIM があればそれを返す（**新規作成しない**）。
  応答 `{ok, channel:{id:"D…"|"G…"}, already_open:bool}`。

## 6. conversations.create / rename
- `name`: 正規化（小文字化、空白→`-`、許可文字 `[a-z0-9-_]`、≤80字）。
  衝突 → `name_taken`。予約語（`general` 等）→ `restricted_action`。
- `is_private`: private 作成。作成者は自動参加・チャンネル管理者扱い。
- rename: ID 不変。`channel_name` システムメッセージを自動投稿。

## 7. reactions.add / remove

| 引数 | 規定 |
|---|---|
| `channel, timestamp` | 対象メッセージ |
| `name` | 絵文字名。skin-tone 接尾辞可。`::` を含む合成名はそのまま保存 |

エラー: `already_reacted`（同者同絵文字）/ `no_reaction`（remove 時未付与）/
`too_many_emoji`（メッセージのユニーク絵文字 >50）/ `too_many_reactions`
（1ユーザー1メッセージ 23 個上限）/ `invalid_name`。
成功時は全閲覧者へ `reaction_added/removed` を配信。

## 8. users.info / users.batchInfo
- `users.info {user}` → user + profile の完全体。`user_not_found`。
- 削除済みユーザーも返す（`deleted:true`）。表示側は名前をグレー表示・presence 無し。

## 9. search.messages

| 引数 | 規定 |
|---|---|
| `query` | M5′(検索文法) のクエリ文字列。修飾子込み |
| `sort` | `score`(既定) / `timestamp` |
| `sort_dir` | `desc`(既定) / `asc` |
| `count` / `page` | ページサイズ ≤100 / ページ番号 ≤100 |
| `highlight` | true で一致部を ``〜`` のマーカーで囲んで返す |

応答: `{ok, query, messages:{total, pagination, matches:[{channel, user, ts, text, permalink, previous, next}]}}`
（previous/next は文脈表示用の前後 1 メッセージ）

## 10. files.getUploadURLExternal / completeUploadExternal
```
getUploadURLExternal { filename, length, alt_txt?, snippet_type? }
  → { ok, upload_url(15分有効・1回限り), file_id:"F…" }
完了前の file は所有者にのみ見える pending 状態。
completeUploadExternal { files:[{id,title?}], channel_id?, thread_ts?, initial_comment? }
  → ファイルを確定し、channel_id があれば file_share メッセージを投稿
  → { ok, files:[完全な file オブジェクト] }
エラー: file_not_found / posting_to_channel_denied / invalid_channel
```

## 11. views.open / update / push / publish
- `views.open {trigger_id, view}`: trigger_id は発行から **3 秒**有効・1回限り
  → `expired_trigger_id`。view スタックは最大 3 枚 → `view_stack_exhausted` (push時)。
- `view` 検証: `type:"modal"`, `title`(plain_text ≤24字), `blocks ≤100`,
  `submit/close ≤24字`, `callback_id ≤255`, `private_metadata ≤3000`。
- `view_submission` への応答 `response_action`:
  `clear`(全閉) / `update {view}` / `push {view}` /
  `errors {block_id: "msg"}`（該当 input の下に赤字表示）。

---

## 12. レート制限の実装規定

```
バケット: (app_id or session, team_id, method_tier) ごとのトークンバケット
Tier1=1/min  Tier2=20/min  Tier3=50/min  Tier4=100/min  (容量=レートの2倍までバースト)
特例: chat.postMessage は (app, channel) ごとに 1/sec、短時間バーストは許容
超過: HTTP 429, ヘッダ Retry-After: <整数秒>, body {"ok":false,"error":"ratelimited"}
```
クライアント SDK の規定挙動: 429 受信 → Retry-After 秒待機 → 自動再試行（最大3回）。

---

## 13. Events API 配信の契約（アプリ向け送信側）

```
POST <app の Request URL>
Headers:
  X-Slack-Signature: v0=hex(hmac_sha256(signing_secret, "v0:"+ts+":"+raw_body))
  X-Slack-Request-Timestamp: <unix秒>
  Content-Type: application/json
Body: { "token":"<verification(レガシー)>", "team_id":"T…", "api_app_id":"A…",
        "event": { …イベント本体（RTM と同形）… },
        "type":"event_callback", "event_id":"Ev…(グローバル一意)",
        "event_time":1718089543,
        "authorizations":[{"team_id","user_id","is_bot"}] }
配信保証: at-least-once。応答 2xx を 3 秒以内に受けなければ失敗とみなし、
  即時 / 1分後 / 5分後 の最大 3 回再送（X-Slack-Retry-Num: 1..3）。
  アプリは event_id で冪等化する義務を負う。
失敗率規定: 95% 超の失敗が 60 分続いたアプリはイベント購読を自動停止し、
  オーナーに警告 DM。再有効化は管理画面から。
```
