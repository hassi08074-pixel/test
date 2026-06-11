# 実装仕様 M1 — メッセージエンジン（書き込みパスの完全定義）

> 本書は「投稿ボタンが押されてから全クライアントに表示されるまで」のすべてを、
> そのまま実装できる粒度（DDL・擬似コード・状態機械・障害時挙動）で定義する。

---

## 1. 物理スキーマ（MySQL 8 / InnoDB, shard key = team_id）

```sql
-- メッセージ本体。PRIMARY KEY が時系列クラスタリングを兼ねる
CREATE TABLE messages (
  team_id        BIGINT UNSIGNED NOT NULL,
  channel_key    BIGINT UNSIGNED NOT NULL,        -- channel_id の内部整数 (Cxxx は外部表現)
  ts_sec         INT UNSIGNED    NOT NULL,        -- unix 秒
  ts_seq         MEDIUMINT UNSIGNED NOT NULL,     -- 同一秒内連番 (0..999999)
  user_key       BIGINT UNSIGNED NULL,
  bot_key        BIGINT UNSIGNED NULL,
  subtype        VARCHAR(32)     NULL,
  text           MEDIUMTEXT      NOT NULL,        -- mrkdwn 原文 (entity encoded)
  blocks         JSON            NULL,            -- rich_text 正本
  attachments    JSON            NULL,
  file_ids       JSON            NULL,            -- ["F...",...]
  thread_root_sec INT UNSIGNED   NULL,            -- 親 ts (返信のみ)
  thread_root_seq MEDIUMINT UNSIGNED NULL,
  client_msg_id  BINARY(16)      NULL,            -- UUID
  edited_by      BIGINT UNSIGNED NULL,
  edited_at      BIGINT          NULL,
  state          TINYINT NOT NULL DEFAULT 0,      -- 0=live 1=deleted(tombstone) 2=ephemeral_expired
  -- 非正規化 (親メッセージのみ使用)
  reply_count        INT UNSIGNED NOT NULL DEFAULT 0,
  reply_users        JSON NULL,                   -- 先頭5名の user_key
  reply_users_count  INT UNSIGNED NOT NULL DEFAULT 0,
  latest_reply_sec   INT UNSIGNED NULL,
  latest_reply_seq   MEDIUMINT UNSIGNED NULL,
  PRIMARY KEY (team_id, channel_key, ts_sec, ts_seq),
  KEY idx_thread (team_id, channel_key, thread_root_sec, thread_root_seq, ts_sec, ts_seq),
  KEY idx_user   (team_id, user_key, ts_sec)
) ENGINE=InnoDB ROW_FORMAT=COMPRESSED;

-- 冪等性テーブル（クライアント再送の重複排除。TTL 24h で purge）
CREATE TABLE message_idempotency (
  team_id       BIGINT UNSIGNED NOT NULL,
  channel_key   BIGINT UNSIGNED NOT NULL,
  client_msg_id BINARY(16) NOT NULL,
  ts_sec        INT UNSIGNED NOT NULL,
  ts_seq        MEDIUMINT UNSIGNED NOT NULL,
  created_at    BIGINT NOT NULL,
  PRIMARY KEY (team_id, channel_key, client_msg_id)
) ENGINE=InnoDB;

-- ts 採番カウンタ（チャンネル×秒。方式Aで使用）
CREATE TABLE ts_counters (
  team_id     BIGINT UNSIGNED NOT NULL,
  channel_key BIGINT UNSIGNED NOT NULL,
  ts_sec      INT UNSIGNED NOT NULL,
  next_seq    MEDIUMINT UNSIGNED NOT NULL,
  PRIMARY KEY (team_id, channel_key, ts_sec)
) ENGINE=InnoDB;

-- 編集履歴（コンプライアンス用。編集前の全文を保存）
CREATE TABLE message_revisions (
  team_id     BIGINT UNSIGNED NOT NULL,
  channel_key BIGINT UNSIGNED NOT NULL,
  ts_sec      INT UNSIGNED NOT NULL,
  ts_seq      MEDIUMINT UNSIGNED NOT NULL,
  revision    SMALLINT UNSIGNED NOT NULL,         -- 1 から増分
  text        MEDIUMTEXT NOT NULL,
  blocks      JSON NULL,
  replaced_at BIGINT NOT NULL,
  replaced_by BIGINT UNSIGNED NOT NULL,
  PRIMARY KEY (team_id, channel_key, ts_sec, ts_seq, revision)
) ENGINE=InnoDB;
```

### ts の外部表現
```
ts_string = printf("%d.%06d", ts_sec, ts_seq)      -- "1718089543.000412"
比較     = (ts_sec, ts_seq) のタプル比較。文字列の数値比較と等価
```
**不変条件 I-1**: 同一 (team, channel) 内で `(ts_sec, ts_seq)` は一意かつ挿入順に単調増加。
**不変条件 I-2**: ts は編集・削除・スレッド化でも永久に不変。permalink `p{ts_sec}{ts_seq}` が壊れないこと。

---

## 2. ts 採番アルゴリズム

### 方式A: DB カウンタ（シンプル・正確。中規模まで推奨）
```
function allocate_ts(team, channel):
  now = wall_clock_seconds()
  # チャンネルの最新 ts より過去の秒を採番しない（時計巻き戻り対策）
  now = max(now, channel.latest_ts_sec)
  loop:
    seq = INSERT INTO ts_counters (team,channel,now,1)
          ON DUPLICATE KEY UPDATE next_seq = LAST_INSERT_ID(next_seq + 1)
          -> LAST_INSERT_ID()                      # アトミックなインクリメント
    if seq <= 999999: return (now, seq - 1)        # seq は 0 始まりで返す
    now += 1                                       # 秒内 100 万件溢れ → 次秒へ
```

### 方式B: チャンネル単位シングルライタ（大規模。Slack 実機に近い）
- チャンネルごとに担当ワーカー（consistent hashing で channel→worker を固定）。
- ワーカーはメモリ上に `(last_sec, last_seq)` を保持し、ロックなしで採番。
- ワーカー failover 時は DB の `MAX(ts)` を読み直してから再開（I-1 を保証）。

**テスト基準**: 同一チャンネルに 64 並列で 10 万件投稿し、ts の重複ゼロ・逆順ゼロであること。

---

## 3. 投稿トランザクション（chat.postMessage の内部）

```
function post_message(auth, req):
  # ---- 1. 検証フェーズ（DB 外）----
  channel = load_channel(req.channel)                 # キャッシュ可
  assert channel.exists                  else error "channel_not_found"
  assert membership(auth.user, channel)  else error "not_in_channel"
       # 例外: chat:write.public スコープの bot は public へ未参加投稿可
  assert not channel.is_archived         else error "is_archived"
  assert posting_permitted(auth, channel) else error "restricted_action"
       # #general 投稿制限 / read-only channel
  assert utf8_len(req.text) <= 40000     else error "msg_too_long"
  assert count(req.blocks) <= 50         else error "invalid_blocks"
  validate_blocks_schema(req.blocks)      else error "invalid_blocks_format"
  if req.thread_ts:
    parent = load_message(channel, req.thread_ts)
    assert parent.exists                 else error "thread_not_found"
    assert parent.thread_root is null OR parent.thread_root == parent.ts
       # 「返信への返信」は親スレッドに付け替える（孫スレッドを作らない）
    if parent.is_reply: req.thread_ts = parent.thread_root
  normalize: text 内の生 URL/絵文字/メンション → エンティティエンコード (M2 参照)

  # ---- 2. 冪等性チェック ----
  if req.client_msg_id:
    hit = SELECT ts FROM message_idempotency WHERE (team,channel,client_msg_id)
    if hit: return existing_message(hit.ts)            # 重複再送 → 同一応答を再返却

  # ---- 3. 書き込みトランザクション ----
  BEGIN
    ts = allocate_ts(team, channel)
    INSERT INTO messages (...)
    if req.client_msg_id: INSERT INTO message_idempotency (...)
    if req.thread_ts:                                  # 親の非正規化を同 Tx で更新
      UPDATE messages parent SET
        reply_count = reply_count + 1,
        latest_reply = ts,
        reply_users = add_capped(reply_users, auth.user, cap=5),
        reply_users_count = recompute_if_needed()
      WHERE ts = req.thread_ts
    UPDATE channels SET latest_ts = ts WHERE id = channel
  COMMIT

  # ---- 4. コミット後の非同期ファンアウト（必ずコミット後）----
  publish("ch:"+channel.id, build_message_event(msg))   # WS 配信 (M3)
  enqueue(JOB_NOTIFY,   msg)                            # 通知判定 (M5)
  enqueue(JOB_INDEX,    msg)                            # 検索インデックス
  enqueue(JOB_UNFURL,   msg)  if contains_urls(msg)     # リンク展開
  enqueue(JOB_WEBHOOK,  msg)                            # Events API 配信

  return {ok:true, channel, ts, message: render(msg)}
```

### 障害時の規定挙動
| 障害 | 挙動 |
|---|---|
| Tx 失敗 | エラー応答。クライアントは同一 client_msg_id で再送（冪等） |
| コミット成功・応答喪失 | 再送 → 冪等テーブルがヒット → 同一 ts を再返却。**二重投稿は発生しない** |
| publish 失敗 | メッセージは存在するが配信漏れ。クライアントは再接続時の gap-fill (M3 §6) で回復。publish はリトライ付きアウトボックス(outbox table + relay)を推奨 |
| 通知ジョブ失敗 | ジョブリトライ（at-least-once、通知側で dedup key = channel+ts+user） |

---

## 4. unfurl（リンク展開）の後付け更新

unfurl は投稿を**ブロックしない**。投稿の数百 ms〜数秒後に attachments を後付けする:

```
JOB_UNFURL(msg):
  urls = extract_urls(msg.text)[0:5]                   # 1メッセージ最大5件展開
  for url in urls:
    if app_registered_for_domain(url): emit link_shared event; continue  # アプリ unfurl 優先
    meta = fetch_ogp(url, timeout=5s, max_size=2MB, ua="Slackbot-LinkExpanding")
    if meta: attachment = {title, text(=description 最大700字), image_url, service_name, from_url:url, is_unfurl:true}
  if attachments:
    UPDATE messages SET attachments = merge(...)
    publish "message_changed" (subtype=message_changed, previous_message 同梱)
```
- 同一 URL がチャンネル内で直近 1 時間に展開済みなら再展開しない（スパム対策）。
- ユーザーが unfurl を×で消す → `attachments` から除去し、`(channel, url, user)` を suppress リストに記録（同者の再投稿では展開しない）。

---

## 5. 編集（chat.update）の状態遷移

```
                 ┌──────── edit (within window) ────────┐
                 ▼                                       │
  [LIVE] ──────────────────────────────────────────► [LIVE(edited)]
     │                                                   │
     │ delete                                            │ delete
     ▼                                                   ▼
  [TOMBSTONE] ──(retention/物理削除ジョブ)──► [PURGED(行削除)]
```

```
function update_message(auth, channel, ts, new_text, new_blocks):
  msg = load(channel, ts);             assert msg else "message_not_found"
  assert msg.state == LIVE             else "cant_update_message"
  assert msg.user == auth.user OR (auth.is_bot AND msg.bot == auth.bot)
                                       else "cant_update_message"
       # 管理者でも他人のメッセージは編集不可（削除のみ可）— Slack の仕様
  window = team_pref("msg_edit_window_mins")
  assert window == -1 OR now() - msg.created <= window*60   else "edit_window_closed"
  BEGIN
    INSERT INTO message_revisions (revision = msg.rev+1, 旧 text/blocks)
    UPDATE messages SET text, blocks, edited_by, edited_at
  COMMIT
  publish message_changed {message: new, previous_message: old}
  re-run mention extraction:
    added_mentions   → 通知ジョブ投入（新規メンションのみ通知）
    removed_mentions → 通知は取り消さない（既送）が mention_count 再計算イベント
  enqueue(JOB_INDEX, msg)              # 検索の再インデックス
```

### 編集の表示規定
- `(edited)` ラベルをメッセージ末尾にグレー 12px で表示。ホバーで編集時刻 tooltip。
- 編集しても並び順・New ライン・既読位置に影響しない（ts 不変のため自動的に満たされる）。

---

## 6. 削除（chat.delete）

```
function delete_message(auth, channel, ts):
  msg = load(channel, ts)
  allowed = (msg.user == auth.user AND team_pref("allow_message_deletion"))
            OR auth.role >= ADMIN
  assert allowed else "cant_delete_message"
  BEGIN
    if msg.is_thread_parent AND msg.reply_count > 0:
      UPDATE messages SET state=TOMBSTONE, text='', blocks=NULL   # 殻を残す
    else:
      UPDATE messages SET state=TOMBSTONE                          # 一覧から非表示
      if msg.is_reply:
        UPDATE parent SET reply_count = reply_count - 1,
                          reply_users = recompute(parent)          # 再計算
    DELETE pins / saved_items 参照
    files: msg にのみ共有されていた file の share を除去（file 実体は残す）
  COMMIT
  publish message_deleted {channel, deleted_ts: ts}
  enqueue(JOB_INDEX_DELETE, msg)
```

### 表示規定
- 返信ゼロの削除 → リストから即時除去（高さアニメーション 150ms で collapse）。
- 返信ありの親の削除 → 「このメッセージは削除されました」をイタリック・グレーで表示し、スレッドは存続。
- 削除はコンプライアンスエクスポート（Discovery API）には `message_revisions` ごと残る（プラン設定依存）。

---

## 7. スレッドの規定挙動（エッジケースまで）

| ケース | 規定挙動 |
|---|---|
| 返信に「スレッドで返信」した | 孫を作らず親スレッドへ付く（§3 で正規化） |
| `reply_broadcast=true` | 返信を保存しつつ、チャンネル本流にも同一メッセージを subtype=thread_broadcast として表示（実体は1行、表示が2箇所） |
| 親より古い ts の返信 | 存在しない（ts 単調増加により構造的に不可能） |
| スレッド購読の自動付与 | (a)親の投稿者 (b)返信した人 (c)スレッド内で @mention された人 (d)手動 Follow した人 |
| 購読解除 | 手動 Unfollow。以後そのスレッドの新着は Threads ビュー/通知に出ない |
| 親が削除されたスレッドへの返信 | 可能（tombstone 親の下に続く） |
| スレッドの最大深さ | 1（親→返信のみ。ネスト無し） |

---

## 8. 受け入れ基準（抜粋）

```gherkin
Scenario: 冪等な再送
  Given クライアントが client_msg_id=X で postMessage を送信し応答を受信できなかった
  When  同一 client_msg_id=X で再送する
  Then  サーバは新規行を作らず、最初の投稿と同一の ts を返す
  And   他クライアントに message イベントは1回しか配信されていない

Scenario: 編集ウィンドウ超過
  Given team_pref msg_edit_window_mins = 5
  And   6分前に投稿したメッセージがある
  When  chat.update を呼ぶ
  Then  ok=false, error="edit_window_closed" が返り、UI は編集メニュー自体を出さない

Scenario: 返信付き親の削除
  Given reply_count=3 の親メッセージ
  When  投稿者が削除する
  Then  本流に「This message was deleted」が残り、3件の返信は閲覧可能のまま
```
