# 仕様検証レポート (VERIFICATION)

対象: `slack-spec/` 機能編 00–12 + 実装編 M1–M11（計24ファイル, 3,942行）
方法: 数値・規則・データフローをファイル横断で突き合わせ、相互参照の整合と内部論理の健全性を検査。
日付: 2026-06-11

---

## サマリ

| 重大度 | 件数 | 状態 |
|---|---|---|
| Critical（実装すると壊れる） | 3 | **修正済** (A,B,C) |
| Major（設計の穴・将来破綻） | 1 | **修正済** (D) |
| Minor（曖昧さ・二重計上リスク） | 1 | **修正済** (E) |
| 整合確認 OK（合格項目） | 7 領域 | 後述 |

修正は本検証で該当ファイルに直接適用済み。以下は内容と根拠。

---

## A. [Critical/修正済] クライアントがスレッド返信を本流未読に二重計上

- **場所**: `M3 §5 on message()` の `is_countable(ev)` ↔ `M5 §2 unread_count_display`
- **不整合**: M3 の `is_countable` は **subtype だけ**を見る。通常のスレッド返信は
  `subtype=null`（→ countable=true）だが `thread_ts` を持つ。一方 M5 のサーバ式は
  `thread_root IS NULL OR thread_broadcast` で**スレッド返信を本流未読から除外**する。
  → クライアントが返信を受けるたびにチャンネル未読 +1 し、`channel_marked` が来るまで
  サーバ正本と食い違う（バッジがチラつく／過大表示）。
- **修正**: `on message()` で「`thread_ts` あり かつ 非 broadcast」を `is_countable`
  到達前に `return` し、本流カウンタに触れないよう変更。スレッド未読は
  `bump_thread_unread`（購読時のみ）へ分離。is_countable に注記を追加。

## B. [Critical/修正済] 未読クエリの述語が演算子優先順位で誤動作

- **場所**: `M5 §2 unread_count_display`
- **不整合**: `AND m.subtype IN (...) AND m.thread_root IS NULL OR m.subtype='thread_broadcast'`
  は括弧が無く、SQL の `AND > OR` 優先順位により
  `(... AND thread_root IS NULL) OR (subtype='thread_broadcast')` と解釈される。
  → `last_read` 以前・他チャンネル・削除済みを含む**全 thread_broadcast を無条件に計上**。
- **修正**: `AND (m.thread_root IS NULL OR m.subtype='thread_broadcast')` と括弧で閉じ、
  「括弧必須」コメントを付与。

## C. [Critical/修正済] ts 採番が各秒の先頭1件で未定義値を返す

- **場所**: `M1 §2 allocate_ts` 方式A
- **不整合**: `INSERT ... ON DUPLICATE KEY UPDATE next_seq=LAST_INSERT_ID(next_seq+1)`
  の **新規行 INSERT 経路では `LAST_INSERT_ID()` が設定されない**（このテーブルに
  AUTO_INCREMENT 列が無いため、直前の呼び出しのセッション値か 0 が返る）。
  → その秒で最初に投稿されたメッセージの `ts_seq` が不定。I-1（単調増加・一意）を破る恐れ。
- **修正**: `ROW_COUNT()` で分岐（`==1`=新規→seq=1 / `==2`=更新→`LAST_INSERT_ID()`）。
  境界も `seq <= 1000000`（→ 0始まり最大 999999）に整合。

## D. [Major/修正済] `@here` のメンション数が履歴再計算不能

- **場所**: `M5 §2 mention_count`（COUNT(mentions_user)） ↔ `M4 §4 conversations.mark`
  （「unread_count_display/mention_count はサーバ再計算値」）↔ `M3 §6 client.counts`
- **不整合**: `mentions_user` は `@here` を **「配信時に active だったか」** で判定する
  （M5 §規定で「配信時点で固定」と明記）。ところが mention_count は `last_read` 以降を
  対象に**後から SQL で COUNT** する設計。presence は揮発で過去値を保持しないため、
  boot/gap-fill/mark のいずれでも `@here` を含む過去メッセージの mention 数を再現できない。
  3経路で値がdrift する。
- **修正**: `message_mentions` materialization テーブルを M1 §3（投稿 Tx 内）と M5 §2.1 に追加。
  配信時に `mentions_user=true` の受信者を `reason` 付きで展開保存し、`mention_count` は
  この表の COUNT に変更。`@here` は確定した active 集合のみ行を持つ＝後から増えない
  （規定どおり）。これで boot・gap-fill・mark が同一テーブルから決定的に一致する。
  副次効果として mention_count が索引付きクエリ化され高速。

## E. [Minor/修正済] アプリバッジで MPIM を二重計上し得る

- **場所**: `M5 §5 app_badge`
- **不整合**: `Σ_channels mention_count + Σ_DMs/MPIMs unread`。`mention_count` は DM/MPIM 内で
  「全件 mention 扱い」(§2 の `is_im OR is_mpim` 節)。`Σ_channels` が im/mpim を含むと、
  同じ DM/MPIM が両項で加算され**約2倍**になる。
- **修正**: 第1項の集合を `channels(¬im ∧ ¬mpim)` に限定し、「二重計上禁止」を明記。

---

## 合格した整合確認（横断チェック OK）

| 領域 | 確認内容 | 判定 |
|---|---|---|
| `trigger_id` 有効期間 | 09 / M4§11 / M7§4.1 すべて「3秒・1回限り」 | ✅ 一致 |
| `response_url` | 09 / M7 すべて「30分・5回・message起点」 | ✅ 一致 |
| メッセージ text 上限 | 02 / M1§1 / M4§1 すべて 40,000 | ✅ 一致 |
| ファイルアップロード | 08 / 12 / M4§10 すべて getUploadURLExternal→直PUT→complete の3段、旧 files.upload 非推奨 | ✅ 一致 |
| blocks 上限 | 09 / 02 / M1§1 / M4§1 / M7§0 すべて message=50 | ✅ 一致（M7 が modal/home=100 を追加定義、矛盾なし） |
| 編集ウィンドウ | 01(`-1=無制限`) / 03(`0=不可`) / M1§5(`window==-1 OR …<=window*60`) | ✅ 論理一致（0 で常に不可、-1 で常に可） |
| reaction 上限 | M1(ユニーク~50) と M4(`too_many_emoji>50` + `too_many_reactions`=1ユーザー23) | ✅ 別軸の2制限で矛盾なし |
| countable subtype 表 | M3§5 と M5§2 が同一集合を参照（A 修正後は thread 条件も一致） | ✅ 修正後一致 |
| reply_broadcast 前提 | M1§7 / M4§1 ともに `thread_ts` 必須・実体1行表示2箇所 | ✅ 一致 |
| 楽観送信 de-dup | M1(client_msg_id 冪等) / M3§5,§7(client_msg_id 照合) | ✅ 一致 |
| presence_sub 上限 | M3§4(最大500) / M3§9(上限500) | ✅ 一致 |

---

## 残存する設計判断（バグではないが要明示の選択）

1. **`conversations.history` は ts 降順**（M4§2）だが**クライアント store は昇順**（M3§5）。
   レイヤ間で並びが逆 → クライアント受信時に reverse が必要。矛盾ではないが実装者向けに
   M4 へ「クライアントは反転して merge」の一文を足すと親切（未適用・任意）。
2. **`channels.last_read`**（01 のスキーマにカラムとして併記、注釈で「実体は membership 側」）。
   正規化上は `channel_members.last_read` が唯一の真実。01 の二重掲載は注釈付きなので可だが、
   実装時は channels 側カラムを作らないこと（レポートとして明記）。
3. **キーワード一致が mention_count に算入**（M5§2 `keyword_hit`）。Slack 実機もキーワードは
   赤バッジ相当だが、組織によっては「ハイライトのみ・バッジ非加算」を望む場合がある。
   現仕様は加算で固定 — 設定化するなら `keywords_count_as_mention` pref を追加する余地あり。

---

## 検証の結論

- **Critical 3件はいずれも「そのまま実装するとデータ不整合（未読・ts）を生む」実害級**で、
  仕様レビューの主目的を果たした。すべて修正適用済み。
- Major 1件（@here）はアーキテクチャの欠落で、materialization 追加により boot/gap-fill/mark の
  三経路整合を回復。
- 横断する固定値（3秒・30分・40000・3段アップロード等）は全ファイルで一貫しており、
  仕様群の内部参照規律は概ね健全。
- 次の検証対象候補: M7 のブロック上限値を Slack 実 API のエラーメッセージ文言と突合、
  M8 ランキング係数の単調性、M9/M10 の障害シナリオの網羅性。
