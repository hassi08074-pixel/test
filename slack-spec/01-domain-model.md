# Slack 完全再現仕様書 — 01. ドメインモデル / データスキーマ

> 物理スキーマは「シャーディングされた MySQL、shard key = team_id」を前提に記述する。
> JSON はクライアントへ返す論理表現（Web API 互換）。実カラムは正規化される。

---

## 1.1 シャーディング戦略

- **シャードキー = `team_id`**。1ワークスペースのデータは原則1シャードに収まり、結合がシャード内で完結する。
- Enterprise Grid の共有チャンネルは「共有メタ」を別管理し、各 team から参照する。
- グローバル一意が必要なもの（User の `W...`、File、App）は別途グローバル ID 空間。
- メッセージは `(team_id, channel_id)` でローカリティを持ち、`ts` で範囲スキャン。

---

## 1.2 Team / Workspace

```
table teams
  id              TID  PK        -- "T024BE7LD"
  enterprise_id   EID  NULL      -- Grid 所属時
  name            varchar
  domain          varchar UNIQUE -- myteam (.slack.com)
  email_domain    varchar        -- 自動参加許可ドメイン
  icon            json           -- {image_34,...,image_original}
  plan            enum(free,pro,business_plus,enterprise)
  created         bigint
  archived        bool
  locale          varchar        -- 既定ロケール
  -- 設定（実際は team_prefs 別テーブル）
```

```
table team_prefs            -- キー・バリュー（数百項目）
  team_id  TID
  name     varchar          -- e.g. "who_can_create_channels"
  value    json
  PK(team_id, name)
```

主要 prefs（再現すべき代表例）:
- `who_can_create_channels`, `who_can_archive_channels`, `who_can_create_private_channels`
- `who_can_post_general`（#general への投稿制限）
- `default_channels`（新規参加者の自動参加チャンネル）
- `msg_edit_window_mins`（編集可能時間。-1=無制限）
- `allow_message_deletion`
- `display_real_names`（表示名 vs 本名）
- `dnd_enabled`, `dnd_start_hour`, `dnd_end_hour`
- `retention_type`, `retention_duration`（メッセージ保持）
- `custom_status_presets`

---

## 1.3 User

```
table users
  id            UID PK         -- "U023BECGF" (Grid global は "W...")
  team_id       TID            -- ホームチーム
  enterprise_id EID NULL
  name          varchar        -- 旧来の handle（@name）。歴史的
  deleted       bool
  is_bot        bool
  is_app_user   bool
  is_admin      bool
  is_owner      bool
  is_primary_owner bool
  is_restricted    bool        -- multi-channel guest
  is_ultra_restricted bool     -- single-channel guest
  tz            varchar        -- "America/New_York"
  tz_offset     int            -- 秒
  updated       bigint
```

```
table user_profiles
  user_id       UID PK
  real_name     varchar
  display_name  varchar        -- @表示名（一意性は team 内推奨）
  first_name / last_name
  title         varchar        -- 役職
  phone / skype
  email         varchar
  avatar_hash   varchar
  image_24/32/48/72/192/512/1024  url
  status_text   varchar(100)
  status_emoji  varchar
  status_expiration bigint
  pronouns      varchar
  fields        json           -- カスタムプロフィール項目
```

- **表示名解決順**: `display_name` → なければ `real_name` → なければ `name`。
- **アバター**: 複数解像度を事前生成。`avatar_hash` 変化でキャッシュバスティング。
- **ゲスト2種**: multi-channel guest（複数チャンネル可）/ single-channel guest（1チャンネルのみ）。

---

## 1.4 Channel（public/private/DM/MPIM 統合モデル）

Slack は内部的に**会話 (conversation)** という統合概念で channel/DM/MPIM を扱う。`conversations.*` API はこの統合面。

```
table channels (conversations)
  id            CID PK         -- C.../D.../G...
  team_id       TID            -- shared 時は複数。connect_team_ids[]
  name          varchar        -- DM/MPIM は内部生成名
  name_normalized varchar
  is_channel    bool           -- public or private channel
  is_group      bool           -- 旧 private group（歴史的）
  is_im         bool           -- 1:1 DM
  is_mpim       bool           -- group DM
  is_private    bool
  is_archived   bool
  is_general    bool           -- #general
  is_shared     bool           -- Slack Connect / Grid 共有
  is_org_shared bool
  is_ext_shared bool           -- 外部組織と共有
  creator       UID
  created       bigint
  topic         json           -- {value,creator,last_set}
  purpose       json           -- {value,creator,last_set}
  num_members   int
  last_read     ts             -- ※ per-member（実際は membership 側）
  latest        ts             -- 最新メッセージ ts
  unlinked      int
  pending_shared json          -- connect 招待中の team
```

```
table channel_members
  channel_id    CID
  user_id       UID
  date_joined   bigint
  last_read     ts             -- 既読位置（このユーザーの）
  notification_prefs json      -- channel 単位の通知設定
  is_muted      bool
  section_id    varchar        -- サイドバー custom section
  PK(channel_id, user_id)
```

- **DM の生成規則**: 1:1 DM は `(team, userA, userB)` の組で一意。MPIM は参加者集合で一意。
- **既読 = `channel_members.last_read`（最後に読んだメッセージの ts）**。未読数 = `latest > last_read` のメッセージ数（mention 数は別カウント）。

---

## 1.5 Message（最重要エンティティ）

```
table messages
  team_id       TID
  channel_id    CID
  ts            varchar PK     -- "1633036800.000100"（channel内一意・単調増加）
  user          UID NULL       -- bot 投稿は bot_id 側
  bot_id        BID NULL
  app_id        AID NULL
  type          enum(message)  -- ほぼ "message"
  subtype       varchar NULL   -- bot_message, channel_join, me_message, thread_broadcast,
                                --  channel_topic, channel_purpose, channel_name,
                                --  channel_archive, reminder_add, file_share, tombstone...
  text          mediumtext     -- mrkdwn 原文（メンションは <@U..> エンコード）
  blocks        json           -- Block Kit 構造（リッチ表現の正本）
  attachments   json           -- レガシー attachments / unfurl
  files         json           -- 添付ファイル参照[]
  thread_ts     varchar NULL   -- 親 ts（返信の場合）。親自身は thread_ts==ts
  reply_count   int            -- 親に保持
  reply_users   json           -- 返信者 UID[]（上位N）
  reply_users_count int
  latest_reply  ts
  subscribed    bool           -- スレッド購読（per-user は別表）
  reactions     json           -- [{name, count, users[]}]（非正規化キャッシュ）
  pinned_to     json           -- [channel_id]
  edited        json NULL      -- {user, ts}
  deleted       bool
  client_msg_id uuid           -- 重複送信防止（楽観 UI の照合キー）
  hidden        bool
  is_locked     bool           -- スレッド locked
  parent_user_id UID NULL      -- 返信時、親の投稿者
```

### メッセージの不変条件
- `ts` は作成時に採番、編集・削除後も不変。
- 削除は通常**論理削除（tombstone）**。`subtype=tombstone` または `deleted=true` でクライアントに「This message was deleted」を表示。コンプライアンスエクスポートには原本保持の場合あり。
- 編集は `text/blocks` を更新し `edited={user,ts}` を付与。`ts`（並び順）は変えない。
- **`client_msg_id`**: クライアント生成 UUID。サーバは同一 client_msg_id の二重投稿を冪等に弾く（再送・楽観 UI 照合）。

### スレッドの表現
- 親メッセージ: `thread_ts == ts`、`reply_count > 0`。
- 返信: `thread_ts = 親ts`, `ts = 自身`。
- **thread_broadcast**: スレッド返信をチャンネル本流にも表示するフラグ（`subtype=thread_broadcast`）。

```
table thread_subscriptions    -- 「このスレッドをフォロー」
  channel_id CID
  thread_ts  varchar
  user_id    UID
  last_read  ts                -- スレッド内既読
  PK(channel_id, thread_ts, user_id)
```

---

## 1.6 Reactions

```
table reactions
  channel_id CID
  message_ts varchar
  name       varchar     -- "thumbsup" / "custom_emoji_name" / "skin-tone付き 'wave::skin-tone-3'"
  user_id    UID
  ts         bigint      -- リアクション時刻（順序保持）
  PK(channel_id, message_ts, name, user_id)
```

- 表示は `name` ごとに集約 `{name, count, users[]}`。順序は最初に付いた順。
- 1メッセージへのユニーク絵文字数に上限（実機 ~50）。
- カスタム絵文字・絵文字エイリアス・スキントーン修飾子に対応。

---

## 1.7 Custom Emoji

```
table emoji
  team_id TID
  name    varchar      -- ":partyparrot:" の "partyparrot"
  url     varchar      -- 画像 or "alias:other_name"
  creator UID
  created bigint
  is_alias bool
  alias_for varchar
```

- 標準絵文字は Unicode + ショートコード辞書（emoji-data 準拠）。
- カスタムはチーム固有。アニメ GIF 可。エイリアス可。

---

## 1.8 Files

```
table files
  id          FID PK
  team_id     TID
  user        UID
  created     bigint
  name        varchar
  title       varchar
  mimetype    varchar
  filetype    varchar       -- "pdf","png","javascript","gdoc"...
  pretty_type varchar
  size        bigint
  mode        enum(hosted, external, snippet, post, email)
  is_external bool
  url_private        url     -- 認証必須ダウンロード
  url_private_download url
  thumb_64/80/360/480/720/800/960/1024  url   -- 画像/動画サムネ
  permalink   url
  permalink_public url NULL
  preview     text          -- snippet/post の本文プレビュー
  channels    json          -- 共有先 channel[]
  ims/groups  json
  comments_count int
  is_public   bool
  shares      json          -- どこに共有されたか
```

- スニペット（コードブロックファイル）、Post（リッチ文書）、外部ファイル参照（Google Drive 等）を file モデルで統一。
- ダウンロードは署名付き URL（`url_private` は要認証、`permalink_public` は公開トグル時のみ）。

---

## 1.9 Pins / Saved(Later) / Reminders

```
table pins
  channel_id CID
  item_type  enum(message,file)
  item_id    ts | FID
  user_id    UID          -- ピン留めした人
  created    bigint
  PK(channel_id, item_type, item_id)
```

```
table saved_items          -- 旧 stars → Saved/Later
  user_id    UID
  item_type  enum(message,file,channel,thread)
  channel_id CID NULL
  item_id    ts | FID NULL
  state      enum(in_progress, archived, completed)  -- Later の状態
  remind_at  bigint NULL    -- リマインド予約
  created    bigint
```

```
table reminders
  id        RID PK
  creator   UID
  user      UID            -- 対象（自分 or 他人）
  text      varchar
  recurring varchar NULL    -- "every weekday at 9am" 等
  time      bigint
  complete_ts bigint NULL
```

---

## 1.10 Usergroups（@subteam）

```
table usergroups
  id          SID PK       -- "S0614TZR7"
  team_id     TID
  handle      varchar      -- "@engineering"
  name        varchar
  description varchar
  is_external bool
  date_create / date_update / date_delete
  auto_type   enum(null, admins, owners)  -- 自動メンバ
  user_count  int
```

```
table usergroup_members
  usergroup_id SID
  user_id      UID
  PK(usergroup_id,user_id)
```

- `<!subteam^S0614TZR7|@engineering>` でメンション → 全メンバーに mention 通知。
- default_channels を持ち、メンバー追加時に自動参加させられる。

---

## 1.11 Apps / Bots / Installations

```
table apps
  id           AID PK
  name         varchar
  description  text
  developer_team TID
  is_directory_approved bool
  redirect_urls json
  scopes_requested json
```

```
table bot_users
  bot_id   BID PK
  app_id   AID
  user_id  UID            -- bot も users 行を持つ
  name     varchar
  icons    json
```

```
table app_installations
  app_id    AID
  team_id   TID
  enterprise_id EID NULL
  installer UID
  bot_user_id BID
  bot_access_token (encrypted)
  user_access_token (encrypted, per-user 認可時)
  scopes    json           -- 付与された OAuth scope
  incoming_webhooks json   -- [{channel_id, url, configuration_url}]
  event_subscriptions json
  slash_commands json
  installed_at bigint
  PK(app_id, team_id)
```

---

## 1.12 Calls / Huddles

```
table calls
  id        RID PK         -- "R0E69JAID"
  date_start bigint
  external_unique_id varchar
  join_url   url
  desktop_app_join_url url
  title      varchar
  created_by UID
  participants json        -- [{slack_id|external_id, display_name, avatar_url}]
  channels   json          -- 紐づけ先
```

- Huddle は call の軽量サブタイプ。チャンネル/DM に紐づき、参加者プレゼンスをリアルタイム表示。

---

## 1.13 Canvas

```
table canvases
  id          varchar PK
  team_id     TID
  channel_id  CID NULL       -- channel canvas は 1:1。free-standing canvas もある
  title       varchar
  document    json           -- ブロックドキュメント（リッチテキスト＋埋め込み）
  created_by  UID
  updated     bigint
  access      json           -- 編集/閲覧権限
```

- Canvas は CRDT 風の共同編集ドキュメント。テキスト・チェックリスト・テーブル・埋め込み（チャンネル/人/ファイル参照）を持つ。

---

## 1.14 監査・コンプライアンス

```
table audit_logs          -- Enterprise Grid
  id        varchar
  date_create bigint
  action    varchar        -- "user_login","file_downloaded","channel_archive"...
  actor     json           -- {type,user}
  entity    json
  context   json           -- ip, ua, location
```

```
table message_history     -- eDiscovery / 編集履歴の完全保持
  channel_id CID
  ts         varchar
  revision   int
  text/blocks json
  edited_by  UID
  edit_ts    bigint
```

- 法的保持 (Legal Hold)、保持ポリシー (Retention)、DLP の対象。削除しても eDiscovery エクスポートで原本を取得可能（プラン依存）。

---

## 1.15 主要インデックス設計

- `messages (channel_id, ts)` クラスタ — チャンネル時系列読み出しの主役。
- `messages (channel_id, thread_ts, ts)` — スレッド読み出し。
- `channel_members (user_id, channel_id)` — ユーザーのチャンネル一覧（ブートデータ）。
- `reactions (channel_id, message_ts)` — リアクション集約。
- 全文検索は別系（Elasticsearch）に `messages` を非同期投入（`team_id, channel_id, user, ts, text, has:` 等のメタ付き）。

---

## 1.16 整合性ルールまとめ

1. `ts` は採番後不変・channel 内単調増加・一意。
2. 既読は per-(user,channel) の `last_read`。メッセージ単位既読は持たない。
3. 編集・削除で並び順を変えない。
4. `client_msg_id` で投稿冪等性を担保。
5. スレッド返信は親の `reply_count/reply_users/latest_reply` を更新（非正規化）。
6. リアクション・ピン・未読は最終的整合（リアルタイムイベントで収束）。
