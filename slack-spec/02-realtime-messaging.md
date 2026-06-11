# Slack 完全再現仕様書 — 02. リアルタイム基盤・配信・整合性

> Slack の体感の正体は「即時配信 + 楽観 UI + 起動時スナップショット + 差分イベント」の合成。
> ここを忠実に作らないと「Slack っぽさ」が出ない。

---

## 2.1 接続ライフサイクル

### 起動シーケンス（クライアント boot）
1. **認証**: トークン（セッション or OAuth）を提示。
2. **ブートデータ取得**: `client.boot` 相当。1リクエストで以下のスナップショットを返す（巨大 JSON、gzip）:
   - `self`（自ユーザー）、`team`、`prefs`
   - `users`（軽量メタ。大規模チームでは遅延ロード＝**lazy-loading "flannel"**）
   - `channels`/`groups`/`ims`（自分のメンバーシップと各 `last_read`/`unread_count`/`latest`）
   - `bots`, `usergroups`, `emoji`, `dnd`
   - **WebSocket URL**（短命・署名付き）
3. **WebSocket 接続**: 取得した `wss://` URL に接続。
4. **`hello` イベント受信** → 接続確立。
5. クライアントは表示中チャンネルの履歴を `conversations.history` で取得。

### 再接続
- WS 切断時、クライアントは指数バックオフ（1s,2s,4s… + jitter）で再接続。
- 再接続後、**`last_event_ts` 以降の取りこぼしイベントを再取得**するため、各チャンネルの `latest` と `last_read` を突き合わせて `conversations.history` で穴埋め。
- 長期切断（数分超）は full boot をやり直し。

> **flannel（エッジキャッシュ）**: 大規模チームで全ユーザーオブジェクトをブートに含めると重すぎるため、ユーザー/チャンネルのメタを地理分散エッジキャッシュから遅延取得する。再現時は「ユーザー辞書の遅延 hydrate」を実装する。

---

## 2.2 WebSocket イベントプロトコル（RTM）

すべてのイベントは JSON。最小形:
```json
{ "type": "message", "channel": "C024BE91L", "user": "U...",
  "text": "hello", "ts": "1633036800.000100",
  "client_msg_id": "uuid", "team": "T..." }
```

### 必ず実装すべきイベント型（抜粋・完全再現対象）
| type | 意味 |
|---|---|
| `hello` | 接続確立 |
| `message` | 新規メッセージ（subtype 付きで join/edit/delete も表現） |
| `message`(subtype=`message_changed`) | 編集。`message` と `previous_message` を内包 |
| `message`(subtype=`message_deleted`) | 削除。`deleted_ts` |
| `message`(subtype=`channel_join`/`channel_leave`) | 入退室 |
| `reaction_added` / `reaction_removed` | リアクション |
| `pin_added` / `pin_removed` | ピン |
| `star_added` / `star_removed` | 保存（Saved） |
| `channel_created/rename/archive/unarchive/deleted` | チャンネル変更 |
| `member_joined_channel` / `member_left_channel` | メンバー変更 |
| `channel_marked` / `im_marked` / `group_marked` | 既読位置更新（**他端末同期**） |
| `thread_marked` | スレッド既読同期 |
| `user_typing` | 入力中インジケータ |
| `presence_change` | プレゼンス変化 |
| `presence_query` / `presence_sub` | プレゼンス購読（後述） |
| `dnd_updated` / `dnd_updated_user` | DND 変化 |
| `user_change` | プロフィール変更 |
| `team_join` | 新メンバー参加 |
| `emoji_changed` | カスタム絵文字追加/削除 |
| `file_created/shared/public/change/deleted` | ファイル |
| `pref_change` / `team_pref_change` | 設定変更 |
| `commands_changed`, `app_*` | プラットフォーム |
| `goodbye` | サーバ都合切断予告 → クライアントは再接続 |
| `reconnect_url` | 次回再接続用 URL の事前配布 |

### typing
- クライアントは composer 入力中、**最大 3 秒ごと**に `user_typing` を自チャンネルへ送信（投げっぱなし）。
- 受信側は数秒間「○○ さんが入力中…」を表示し、タイムアウトで消す。

### 既読の多端末同期
- あるユーザーが端末Aでチャンネルを既読にすると、サーバは `channel_marked {channel, ts, unread_count, num_mentions}` を**同一ユーザーの全接続**へ配信。端末B/Cの未読バッジが即同期。

---

## 2.3 配信（fan-out）アーキテクチャ

```
投稿API (chat.postMessage)
   │ 1. 採番: ts = next_monotonic(channel)
   │ 2. 永続化: messages へ INSERT（client_msg_id 冪等チェック）
   │ 3. 非正規化更新: channel.latest, thread reply_count 等
   │ 4. publish(channel_topic, message_event)
   ▼
Pub/Sub (Redis/NATS), topic = "ch:C024BE91L"
   ▼
WS Gateway（各ノードが購読チャンネルの subscriber socket を保持）
   ▼
購読中の全クライアントへ push（その channel をメンバーかつ「関心あり」のソケット）
```

- **購読モデル**: クライアントは boot で得た自分のチャンネル群に暗黙購読。ゲートウェイは `socket → channels` のマップを保持し、`channel → sockets` で逆引き配信。
- **ファンアウト最適化**: 数万人チャンネル (#general 等) は配信が重い。バッチ・コアレッシング・地理分散ゲートウェイで吸収。
- **順序保証**: 同一チャンネル内は `ts` 単調増加で全クライアント同順。チャンネル間順序は保証しない。

---

## 2.4 楽観的 UI（Optimistic Send）

投稿体験の核心。手順を厳密再現する:

1. ユーザーが Enter → クライアントは `client_msg_id = uuid()` を生成。
2. **即座にローカル表示**（送信中状態、薄い表示 or 時計アイコン）。`ts` はまだ無い → 一時キー = client_msg_id。
3. `chat.postMessage{channel,text,blocks,client_msg_id,thread_ts?}` を送信。
4. サーバ応答 `{ok:true, ts, message}` を受信 → ローカルの仮メッセージを**正式な ts に置換**（client_msg_id で照合）。
5. 並行して RTM の `message` イベントも届く → client_msg_id 一致なら重複表示しない（既に置換済み）。
6. 失敗時 → 「送信できませんでした。再試行」表示。再送は同一 client_msg_id（冪等）。

> **二重表示の回避**が肝。自分の投稿は「API レスポンス」と「RTM ブロードキャスト」の両方で届くため、`client_msg_id` で de-dup する。

---

## 2.5 Presence（オンライン状態）

- 状態は **`active` / `away`** の2値（+ `connection_count`）。
- **自動 away**: クライアントが一定時間（既定 ~10〜30分）操作なし、または全 WS 切断で away。
- **手動 away**: ユーザーが明示設定（`users.setPresence`）。
- **購読方式（重要なスケール工夫）**: 全員の presence をブロードキャストするとコストが爆発するため、**クライアントが「今表示中のユーザー」だけを購読**する。
  - `presence_sub {ids:[U...]}` を送信 → そのユーザーの `presence_change` のみ受信。
  - `presence_query` で即時問い合わせ。
- presence とカスタムステータス（emoji+text）は別概念。ステータスは `user_profile` に永続、presence は揮発。

---

## 2.6 DND（Do Not Disturb）

- 各ユーザーは DND スケジュール（開始/終了時刻、TZ 考慮）と手動スヌーズを持つ。
- DND 中は**プッシュ/デスクトップ通知を抑制**（バッジ・未読は更新される）。
- `dnd.info`/`dnd.setSnooze`、RTM `dnd_updated`（自分）/`dnd_updated_user`（他者）で同期。
- 送信側 UX: DND 中の相手にメンションすると「通知は届きません。今すぐ通知しますか？」（notify anyway）を提示。

---

## 2.7 未読・バッジ計算

クライアント側で boot データ + RTM イベントから算出:
- チャンネル `unread_count` = `latest` と `last_read` の間のメッセージ数。
- `mention_count` = その範囲のうち自分宛 mention（`<@self>`, `<!here>`(active時), `<!channel>`, 所属 `<!subteam>`）数。
- サイドバーは「未読あり=太字」「mention あり=赤バッジ＋数値」。
- アプリ全体バッジ（OS バッジ）= 全 DM/mention の合計。
- ミュートチャンネルは未読に**太字を出さない**（mention は出す設定可）。

---

## 2.8 整合性・配信保証レベル

| 対象 | 保証 |
|---|---|
| メッセージ永続化 | 強整合（DB コミット後に ts 確定・配信） |
| 同一チャンネル順序 | 全クライアントで同順（ts 単調） |
| RTM 配信 | at-least-once（client_msg_id / ts で冪等化） |
| 取りこぼし | 再接続時に history 差分で回復 |
| リアクション/ピン/既読 | 結果整合（イベントで収束） |
| presence/typing | ベストエフォート（揮発） |

---

## 2.9 レート・サイズ制限（配信系）

- 1メッセージ `text` 上限 ~40,000 文字（超過は分割 or ファイル化）。
- blocks 上限: 1メッセージ 50 ブロック、各種要素にサイズ上限（09 参照）。
- `chat.postMessage` は Tier 相当のレート（概ね 1 msg/sec/channel をバースト許容）。
- typing は投げ捨て、サーバ側で抑制。

---

## 2.10 再現チェックリスト（リアルタイム）
- [ ] boot で1往復スナップショット＋WS URL を返す
- [ ] WS で hello → message/reaction/marked 等を配信
- [ ] client_msg_id による楽観送信と de-dup
- [ ] ts 単調採番・チャンネル内全順序
- [ ] channel_marked による多端末既読同期
- [ ] presence_sub による表示中ユーザー限定購読
- [ ] typing の 3 秒間隔・タイムアウト消去
- [ ] 再接続時の history 差分穴埋め
- [ ] goodbye/reconnect_url によるグレースフル切替
