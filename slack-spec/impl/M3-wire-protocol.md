# 実装仕様 M3 — ワイヤプロトコル（boot / WebSocket / 再接続 / 重複排除）

> クライアント⇄サーバ間のフレーム単位の契約。これに準拠すれば
> どのクライアント実装でも同一の同期挙動になる。

---

## 1. 接続状態機械（クライアント側）

```
            start
              │
              ▼
        ┌─ BOOTING ─┐  client.boot 成功
        │           ├───────────────► CONNECTING ── ws open+hello ──► CONNECTED
        │  失敗(retry │                    │  ws error/close              │
        │  w/backoff) │                    ▼                             │
        └────────────┘               RECONNECT_WAIT ◄── ws close/goodbye ─┤
                                          │ backoff 経過                  │
                                          ▼                              │
                                  (切断 < 5min) ─► CONNECTING + GAP_FILL ─┘
                                  (切断 ≥ 5min or バージョン不整合) ─► BOOTING (full re-boot)
backoff = min(30s, 1s * 2^attempt) + rand(0..1s)
```

---

## 2. boot 応答（必須フィールドの完全スキーマ）

`POST /api/client.boot` → 200 (gzip JSON)

```jsonc
{
  "ok": true,
  "self":   { "id":"U…", "name":"…", "prefs": { /* 全ユーザー設定 KV */ } },
  "team":   { "id":"T…", "name":"…", "domain":"…", "icon":{…}, "prefs":{…} },
  "ws_url": "wss://gateway…/?token=<短命署名 90s 有効>",
  "cache_ts": "1718089543.000000",        // このスナップショットの基準時刻
  "channels": [ {                          // 自分がメンバーの全会話 + 全 public のメタ
      "id":"C…", "name":"…", "is_member":true, "is_private":false,
      "last_read":"1718000000.000100",
      "latest":  "1718089000.000200",      // 最新メッセージ ts（プレビューは含めない）
      "unread_count_display": 12,          // join/leave 等を除いた未読数
      "mention_count": 2,
      "is_muted": false,
      "topic":{…}, "purpose":{…}
  } ],
  "ims":   [ { "id":"D…", "user":"U…", "last_read":…, "latest":…, … } ],
  "users": "deferred",                     // 大規模チーム: 別途 hydrate (下記§3)
  "bots":  [ … ],
  "usergroups": [ { "id":"S…", "handle":"eng", "user_ids":[…] } ],
  "emoji_version": "e58", "dnd": { "dnd_enabled":true, "next_dnd_start_ts":…, "next_dnd_end_ts":… },
  "subscriptions": { "threads": [ {"channel":"C…","thread_ts":"…","last_read":"…"} ] }
}
```

規定:
- boot は**1往復**で初期描画に必要な全メタを返す（メッセージ本文は含まない）。
- `cache_ts` 以降の差分は WS イベントで埋まる前提。boot と WS 接続の間に発生した
  イベントを失わないため、**サーバは ws_url 発行時点から当該ユーザーのイベントを
  最大 90 秒バッファし、hello 直後に再生**する（または client が §6 gap-fill を実行）。

## 3. ユーザー辞書の遅延 hydrate（flannel 方式）

- 閾値: メンバー数 > 5,000 のチームでは boot に users を含めない。
- クライアントは描画時に未知 user_id を検出 →
  `POST /api/users.batchInfo {ids:[U…×最大250]}` をデバウンス 50ms でまとめ撃ち。
- 取得結果は IndexedDB に `users(version)` でキャッシュ、`user_change` イベントで無効化。
- 名前が未解決の間は**グレーのプレースホルダ矩形**（幅 80px）を描画し、解決後に置換。

---

## 4. WebSocket フレーム仕様

- テキストフレーム、UTF-8 JSON、1 フレーム 1 イベント。クライアント→サーバも同形。
- 接続直後にサーバが送る最初のフレームは必ず `{"type":"hello"}`。
- **ping/pong**: クライアントは 10 秒間隔で `{"type":"ping","id":n}` を送信。
  サーバは `{"type":"pong","reply_to":n}`。**30 秒 pong 欠落で切断扱い**→再接続。
- サーバ主導の縮退: `{"type":"goodbye"}` 受信後、クライアントは新規接続を確立して
  から旧接続を閉じる（make-before-break）。

### クライアント→サーバ（上り）で許可されるフレーム
| type | ペイロード | 意味 |
|---|---|---|
| `ping` | `{id}` | keepalive |
| `typing` | `{channel, thread_ts?}` | 入力中。**3 秒に 1 回まで**（超過分はサーバが捨てる） |
| `presence_sub` | `{ids:[U…] 最大500}` | presence 購読の置換（差分でなく全置換） |
| `presence_query` | `{ids:[…]}` | 即時照会（応答は presence_change×n） |

> メッセージ送信は WS では行わない（必ず HTTP `chat.postMessage`）。
> WS 上りは揮発シグナルのみ。これにより送達保証を HTTP 側に一元化する。

### サーバ→クライアント（下り）主要イベントの厳密スキーマ

```jsonc
// 新規メッセージ
{ "type":"message", "channel":"C…", "ts":"1718089543.000412",
  "user":"U…", "text":"…", "blocks":[…],
  "thread_ts":"…",            // 返信のみ
  "client_msg_id":"uuid",     // ユーザー投稿のみ
  "event_ts":"1718089543.000413" }

// 編集（外側 ts ではなく message.ts が対象）
{ "type":"message", "subtype":"message_changed", "channel":"C…",
  "message":          { "ts":"…", "text":"new", "edited":{"user":"U…","ts":"…"} },
  "previous_message": { "ts":"…", "text":"old" },
  "event_ts":"…" }

// 削除
{ "type":"message", "subtype":"message_deleted", "channel":"C…",
  "deleted_ts":"1718089543.000412", "event_ts":"…" }

// リアクション
{ "type":"reaction_added", "user":"U…",
  "item": { "type":"message", "channel":"C…", "ts":"…" },
  "reaction":"thumbsup::skin-tone-2", "item_user":"U…", "event_ts":"…" }

// 既読同期（同一ユーザーの他端末から）
{ "type":"channel_marked", "channel":"C…", "ts":"1718089543.000412",
  "unread_count_display":0, "mention_count":0, "event_ts":"…" }

// スレッド既読
{ "type":"thread_marked", "channel":"C…", "thread_ts":"…", "ts":"…" }

// presence（購読中のみ届く）
{ "type":"presence_change", "user":"U…", "presence":"active"|"away" }

// 入力中
{ "type":"user_typing", "channel":"C…", "user":"U…", "thread_ts":"…?" }
```

---

## 5. クライアント側の適用アルゴリズム（イベント→ローカル状態）

```
on message(ev):
  ch = store.channel(ev.channel)
  if ev.client_msg_id in pending_outbox:          # 自分の楽観送信の折返し
      promote(pending_outbox[ev.client_msg_id], ev.ts); return
  if store.has(ev.channel, ev.ts): return         # at-least-once の dedup（キー= channel+ts）
  insert_sorted(ch.messages, ev)                  # ts 順挿入（通常は末尾 O(1)）
  if ev.thread_ts: update_parent_reply_meta(ev)
  if not viewing(ch) or not window_focused():
      ch.unread_count += (is_countable(ev) ? 1 : 0)   # join/leave 等は数えない
      ch.mention_count += (mentions_me(ev) ? 1 : 0)
  else:
      auto_mark(ch, ev.ts)                        # 表示中なら即既読 (conversations.mark)

on message_changed(ev): store.replace(ev.channel, ev.message.ts, ev.message)
on message_deleted(ev): store.tombstone_or_remove(ev.channel, ev.deleted_ts)
on channel_marked(ev):                            # 他端末同期。自端末由来でも冪等
  ch.last_read = max(ch.last_read, ev.ts)
  ch.unread_count = ev.unread_count_display; ch.mention_count = ev.mention_count
  recompute_app_badge()
```

### is_countable(ev)（未読として数える subtype）
`null(通常), bot_message, file_share, me_message, thread_broadcast` → 数える。
`channel_join, channel_leave, channel_topic, channel_purpose, channel_name,
 channel_archive, reminder_add, tombstone` → 数えない。

---

## 6. 再接続時の gap-fill（取りこぼし回復）の完全手順

```
function gap_fill(disconnect_duration):
  if disconnect_duration >= 5min: full_reboot(); return
  # 1. 軽量差分チェック
  resp = POST /api/client.counts            # 全会話の {id, latest, last_read,
                                            #  unread, mentions} だけ返す軽量 API
  for c in resp.channels:
    local = store.channel(c.id)
    local.last_read = c.last_read           # 他端末で進んだ既読を取り込み
    if c.latest > local.latest_known:
        mark_stale(c.id)                    # 本文が欠けている可能性
  # 2. 開いているチャンネルだけ即時穴埋め
  for ch in visible_channels where stale:
    msgs = conversations.history(ch, oldest=local.latest_known, inclusive=false)
    merge(msgs)                             # dedup キー= ts
  # 3. 非表示チャンネルは「開いた時」に同じ穴埋めを遅延実行
  # 4. boot 世代チェック: resp.cache_version != local.cache_version → full_reboot()
```

**規定**: gap-fill 中も UI はロックしない。穴埋め完了までそのチャンネルのスクロール
末尾に小さなスピナーを表示する。

---

## 7. 楽観送信の状態機械（クライアント outbox）

```
 [DRAFT] --enter--> [PENDING(表示: 通常+送信中の薄表示なし/即時通常表示)]
    PENDING: client_msg_id 採番済み・仮 ts = local_clock（表示順は楽観で末尾固定）
      │ HTTP 2xx {ts}           │ HTTP 失敗/タイムアウト(10s)
      ▼                          ▼
 [CONFIRMED(ts 置換)]      [FAILED(赤! アイコン+「再試行」リンク)]
                                 │ 再試行(同一 client_msg_id) / 端末再起動時に outbox 復元
                                 ▼
                              PENDING
```
- PENDING 中の編集・削除・リアクションは不可（confirmed まで操作メニューを出さない）。
- オフライン時に送信した場合も outbox に積み、再接続後に順次送信（FIFO、チャンネル毎直列）。

---

## 8. typing インジケータの規定

- 受信側: `user_typing` 受信→「<名前>が入力しています…」をコンポーザ上 18px 行に表示。
  **6 秒**新着がなければ消す。複数人は「A と B が入力しています」(3 人以上は「数人が…」)。
- 送信側: キー入力中 3 秒間隔で送出。送信完了・コンポーザ空化で即停止。
- スレッドビューでは `thread_ts` 一致のものだけ表示。

---

## 9. presence 購読の規定

- クライアントは「現在ビューポートに見えているユーザー」(サイドバー DM 一覧 + 表示中
  メッセージの投稿者 + 開いているプロフィール) の集合を 2 秒デバウンスで `presence_sub`。
- 上限 500。超過時は DM 一覧を優先。
- サーバは sub 集合の presence 変化のみ push。sub 時に現在値を即時 1 回送る。

---

## 10. 受け入れ基準

```gherkin
Scenario: 切断中のメッセージを失わない
  Given 端末Aが WS 切断中に他者が同チャンネルへ 3 件投稿した
  When  端末Aが 30 秒後に再接続する
  Then  gap-fill により 3 件が ts 順で過不足なく表示される

Scenario: 自分の投稿が二重表示されない
  Given postMessage の HTTP 応答より先に WS の message イベントが届いた
  Then  client_msg_id 照合により表示は 1 件であり、後着の HTTP 応答で重複しない

Scenario: 他端末の既読が同期される
  Given 端末Aでチャンネルを開き既読化した
  Then  端末Bのサイドバー太字とバッジが 1 秒以内に消える
```
