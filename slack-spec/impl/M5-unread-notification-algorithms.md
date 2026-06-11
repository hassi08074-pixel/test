# 実装仕様 M5 — 未読・メンション・通知判定の決定的アルゴリズム

> 「いつバッジが付き、いつ鳴るか」を入力→出力が一意に決まる関数として定義する。
> ここが曖昧だと体験の再現に失敗する（Slack らしさの大半は通知の節度にある）。

---

## 1. データ構造

```
per (user, channel):
  last_read        : ts        # 既読位置
  notify_pref      : enum { default, all, mentions, nothing }   # per-channel 上書き
  muted            : bool
  mobile_pref      : enum { default, all, mentions, nothing }
per user (global prefs):
  global_notify    : enum { all, mentions_dm, nothing }          # 既定 mentions_dm
  keywords         : [string]            # ハイライトワード（大文字小文字無視・単語境界一致）
  thread_replies_notify : bool           # 購読スレッドの返信を通知するか（既定 true)
  reaction_notify  : enum { all_own, dm_own, none }              # 自分の投稿へのリアクション
  notif_schedule   : {days: 曜日別 {start,end}} | always
  mobile_delay     : enum {0s,2m,5m,10m,30m,...}                 # デスクトップ活動時のモバイル抑制
  dnd              : {scheduled:{start,end,tz}, snooze_until:ts|null}
runtime:
  presence(user)   : active | away
  active_device(user): desktop_focused | desktop_idle | mobile_only | offline
```

---

## 2. 未読数・メンション数（サーバ正本の計算式）

```
function unread_count_display(user, ch):
  return COUNT(messages m WHERE m.channel = ch
               AND m.ts > last_read(user, ch)
               AND m.state = LIVE
               AND m.subtype IN countable_subtypes      # M3§5 の表
               AND m.thread_root IS NULL OR m.subtype = 'thread_broadcast')
               # スレッド返信は本流未読に数えない（broadcast を除く）

function mention_count(user, ch):
  return COUNT(同上範囲 AND mentions_user(m, user))

function mentions_user(m, user):
  e = extract_mentions(m.text)                          # M2§7
  return user.id ∈ e.users
      OR e.channel OR e.everyone                        # @channel/@everyone は常時
      OR (e.here AND presence(user)=active at delivery) # @here は配信時 active のみ
      OR ∃g ∈ e.usergroups: user ∈ members(g)
      OR keyword_hit(m.text_plain, user.keywords)       # 単語境界・大小無視
      OR (m.channel.is_im OR m.channel.is_mpim)         # DM は全件メンション扱い
```

規定:
- `@here` の active 判定は**メッセージ配信時点**で固定（後から active になっても遡らない）。
- キーワード一致は code span / code block 内も対象（メンションと異なる）。
- 数値の正本はサーバ。クライアントは増分計算し、`channel_marked` 受信時にサーバ値で上書き補正する。

---

## 3. 通知判定（1 メッセージ × 1 受信者 → 配信先集合）

```
function decide(m, recipient) -> {desktop, mobile, sound, badge, email_candidate}:
  if m.user == recipient: return NONE                  # 自分の投稿では鳴らさない
  if m.subtype NOT IN countable_subtypes: return NONE
  if not member(recipient, m.channel): return NONE
  if m.thread_root != NULL:                            # ── スレッド返信 ──
      if not subscribed(recipient, m.thread): return NONE
      if mentions_user(m, recipient): level = MENTION
      elif recipient.thread_replies_notify: level = THREAD
      else: return NONE
  else:                                                # ── 本流 ──
      pref = effective_pref(recipient, m.channel)      # per-ch が default ならグローバル
      if muted(recipient, m.channel) AND not mentions_user(m, recipient): return NONE
      if pref == nothing: return NONE
      if pref == all: level = MESSAGE
      elif mentions_user(m, recipient): level = MENTION
      else: return NONE

  # ── 抑制ゲート（順に適用）──
  if in_dnd(recipient, now()):
      return BADGE_ONLY                                # バッジ・未読は更新、音/バナー無し
  if not in_notif_schedule(recipient, now()):
      return BADGE_ONLY

  # ── 配信先ルーティング ──
  dev = active_device(recipient)
  case dev:
    desktop_focused_on_this_channel: return BADGE_ONLY # 見ている画面には鳴らさない
    desktop_focused_elsewhere:       return {desktop, sound, badge}
    desktop_idle(< mobile_delay):    return {desktop, sound, badge, schedule_mobile_after_delay}
    mobile_only / desktop_idle(≥delay): return {mobile_push, badge}
    offline:                         return {mobile_push, badge, email_candidate}
```

### email_candidate の確定
- 全端末 offline のまま **15 分**経過しても未読のメンション/DM が残る場合、
  メール通知ジョブを発火（ユーザー設定 on 時）。本文はメッセージ抜粋 + 返信リンク。
- 同一チャンネルの複数件は 1 通に集約。1 ユーザーあたり最短間隔 15 分。

### schedule_mobile_after_delay
- デスクトップが idle になった時点から `mobile_delay` 後に未読のままならプッシュ。
- 配信前に既読化されたらキャンセル（push job は配信直前に last_read を再確認する）。

---

## 4. プッシュ通知ペイロード（APNs/FCM 共通論理形）

```jsonc
{ "type": "message",
  "channel": "C…", "channel_name": "general", "team": "T…",
  "ts": "…", "thread_ts": "…?",
  "author_display_name": "alice",
  "preview": "alice: デプロイ完了しました 🎉",   // 設定で「内容を隠す」→ "New message"
  "badge": 4,                                   // 計算式は §5
  "actions": ["reply", "mark_read", "emoji"]    // リッチプッシュ
}
```
- グルーピング: チャンネル単位にスレッド ID を割り当て、OS 側で積み重ね表示。
- 既読化やリアクションをプッシュから直接実行（バックグラウンド API 呼び出し）。
- 受信側で既読済み (`ts <= last_read`) なら**表示せず破棄**（サイレントプッシュで取り消しも送る）。

## 5. バッジ数の定義（OS アイコンバッジ）

```
app_badge(user) = Σ_channels mention_count(user, ch)
                + Σ_DMs/MPIMs unread_count_display(user, dm)   # DM は全未読
                + Σ_subscribed_threads thread_unread_mentions
```
- 「未読チャンネルがあるだけ」ではバッジは増えない（メンション/DM のみ）。これが Slack の節度の核。
- サイドバー: 未読あり=チャンネル名太字（白）、メンションあり=赤バッジ+数値。
- ワークスペース切替レール: メンション合計の赤バッジ、未読のみは白ドット。

---

## 6. DND の判定関数

```
function in_dnd(user, t):
  if user.dnd.snooze_until and t < snooze_until: return true
  s = user.dnd.scheduled
  if s.enabled:
      lt = to_local(t, user.tz)
      return lt ∈ [s.start, s.end)      # 日跨ぎ (22:00→08:00) は wrap して判定
  return false
```
- DND 中の送信者側 UI: 相手の名前に 💤 zzz アイコン。DM 送信時に
  「@alice は通知を一時停止中です。今すぐ通知しますか？」→ Notify anyway で
  decide() の DND ゲートをバイパスする 1 回限りのフラグ付き配信。

---

## 7. Activity（通知履歴）フィード

- 左レールの「Activity」は自分宛イベントの永続フィード:
  `mention / thread_reply / reaction(自分の投稿) / app通知 / 招待 / keyword一致`
- 各行 = {種別アイコン, 発生チャンネル, 抜粋, 時刻, 未読ドット}。クリックで該当箇所へ。
- 既読管理は Activity 内独立（`activity_marked`）。フィルタ（メンションのみ等）。

---

## 8. 受け入れ基準

```gherkin
Scenario: 見ている画面では鳴らない
  Given デスクトップで #general を表示しフォーカスしている
  When  #general に自分宛メンションが投稿される
  Then  メッセージは即時表示・既読化され、バナー/音/バッジは一切発生しない

Scenario: モバイル遅延
  Given mobile_delay=2m、デスクトップが idle になって 30 秒
  When  DM を受信する
  Then  デスクトップ通知が出る。2 分後も未読ならモバイルにプッシュが届く
  And   1 分後にデスクトップで既読化した場合、モバイルには何も届かない

Scenario: @here は away に届かない
  Given Bob は away、Carol は active（同一チャンネル）
  When  @here 付きメッセージが投稿される
  Then  Carol は mention_count+1 と通知、Bob は unread のみで mention は付かない

Scenario: ミュートチャンネルのメンション
  Given #noisy をミュート中
  When  @自分 付きメッセージが投稿される
  Then  サイドバーは太字化しないが赤バッジ+1 と通知は届く
```
